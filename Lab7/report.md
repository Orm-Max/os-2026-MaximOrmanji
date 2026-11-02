# Lab 7: Three Small Programs: Processes, Redirection, a Pipe

**MINISTRY OF EDUCATION, CULTURE AND RESEARCH**  
**OF THE REPUBLIC OF MOLDOVA**  

**Technical University of Moldova**  
**Faculty of Computers, Informatics and Microelectronics**  
**Department of Software Engineering and Automation**  


**MAXIM ORMANJI FAF-242**

Checked by: Reițman Petricia

Chișinău, 2026

## Purpose of the work

The purpose of this work is to run the answers to three questions from the lecture: how a process is born, where `printf` writes, and how `ls | wc -l` works. I compile and run three short C programs that do by hand what the shell does: `fork` + `exec` + `wait`, redirection with `close` + `open`, and a pipe with `pipe` + `dup`. I change one line in some of them and look at what happens.


## Before you start

I created the working folder `~/os-lab7` and checked that the compiler is installed.

```
vboxuser@Ubuntu:~$ mkdir -p ~/os-lab7 && cd ~/os-lab7
vboxuser@Ubuntu:~/os-lab7$ gcc --version | head -1
gcc (Ubuntu 15.2.0-16ubuntu1) 15.2.0
```


## Part 1. fork + exec + wait

### 1.1 The program run.c

The program does what the shell does when I type `ls -l`: it copies itself (`fork`), the copy becomes `ls` (`exec`), and the original waits for it (`wait`).

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
/* the three steps of the shell for the command: ls -l */
int main(void){
    printf("parent: my PID is %d\n", getpid());
    fflush(stdout);
    pid_t pid = fork();                 /* step 1: copy myself */
    if (pid < 0) { perror("fork"); return 1; }
    if (pid == 0) {                     /* only the child runs this */
        printf("child: my PID is %d, my parent is %d\n", getpid(), getppid());
        fflush(stdout);
        execlp("ls", "ls", "-l", (char *)NULL);   /* step 2: become ls */
        perror("exec");                 /* runs only if exec failed */
        exit(127);
    }
    int status;
    waitpid(pid, &status, 0);           /* step 3: wait for the child */
    printf("parent: child %d ended with status %d\n", pid, WEXITSTATUS(status));
    return 0;
}
```

<p align="center"><em>Figure 1.1 – Contents of run.c</em></p>

### 1.2 Compile and run

```
vboxuser@Ubuntu:~/os-lab7$ gcc -Wall -Wextra -o run run.c
vboxuser@Ubuntu:~/os-lab7$ ./run
parent: my PID is 13820
child: my PID is 13821, my parent is 13820
total 24
-rwxrwxr-x 1 vboxuser vboxuser 16392 Nov  1 18:57 run
-rw-rw-r-- 1 vboxuser vboxuser   866 Nov  1 18:57 run.c
parent: child 13821 ended with status 0
```

<p align="center"><em>Figure 1.2 – Running run.c with ls -l</em></p>

### 1.3 The child runs another program (date)

I replaced the `execlp` line with `execlp("date", "date", (char *)NULL);` (in a copy of the file, `run_date.c`, so that `run.c` stays with `ls`).

```
vboxuser@Ubuntu:~/os-lab7$ sed 's/execlp("ls", "ls", "-l"/execlp("date", "date"/' run.c > run_date.c
vboxuser@Ubuntu:~/os-lab7$ gcc -Wall -Wextra -o run_date run_date.c
vboxuser@Ubuntu:~/os-lab7$ ./run_date
parent: my PID is 13828
child: my PID is 13829, my parent is 13828
Sun Nov  1 06:57:49 PM UTC 2026
parent: child 13829 ended with status 0
```

<p align="center"><em>Figure 1.3 – The child became date</em></p>

### 1.4 A program that does not exist

```
vboxuser@Ubuntu:~/os-lab7$ sed 's/execlp("ls", "ls", "-l"/execlp("nosuchprog", "nosuchprog"/' run.c > run_nosuch.c
vboxuser@Ubuntu:~/os-lab7$ gcc -Wall -Wextra -o run_nosuch run_nosuch.c
vboxuser@Ubuntu:~/os-lab7$ ./run_nosuch
parent: my PID is 13836
child: my PID is 13837, my parent is 13836
exec: No such file or directory
parent: child 13837 ended with status 127
```

<p align="center"><em>Figure 1.4 – exec fails for nosuchprog</em></p>

### Observe

**1. The line `perror("exec")` is in the child, right after `execlp`. Why is it printed only with `nosuchprog`?**
When `exec` works, the code of the child process is replaced by the new program (`ls` or `date`), so the old code, including the line with `perror`, does not exist any more and `execlp` never returns. When the program does not exist, `exec` fails and returns to the old code, so the next line runs: `perror("exec")` prints `No such file or directory`, and `exit(127)` ends the child.

**2. Which number does the child print as "my parent", and whose PID is it?**
In Figure 1.4 the child (13837) prints `my parent is 13836`. It is the PID of the parent process, that is, of my `run_nosuch` program that called `fork` (the parent printed `my PID is 13836` in the first line). The same is true in the other runs (13820 → 13821, 13828 → 13829).

**3. What status does the parent see with `ls`, and what status with `nosuchprog`?**
With `ls` the status is 0 (success). With `nosuchprog` it is 127: this is the number from `exit(127)` in the child after the failed `exec`, and it is the same code that the shell returned for `command not found` in Lab 3.


## Part 2. A zombie, on purpose

A zombie is a child that has ended but has not been waited for. I made one by adding `sleep(20);` before `waitpid`, so the parent sleeps and the child `ls` ends in the meantime.

### 2.1 The program zombie.c

I made `zombie.c` from `run.c` (the version with `ls`) and checked that the new line is just before `waitpid`:

```
vboxuser@Ubuntu:~/os-lab7$ sed 's/^\( *\)waitpid(pid, &status, 0);/\1sleep(20);\n\1waitpid(pid, \&status, 0);/' run.c > zombie.c
vboxuser@Ubuntu:~/os-lab7$ grep -n -B1 -A1 "sleep(20)" zombie.c
18-    int status;
19:    sleep(20);
20-    waitpid(pid, &status, 0);           /* step 3: wait for the child */
vboxuser@Ubuntu:~/os-lab7$ gcc -Wall -Wextra -o zombie zombie.c
```

<p align="center"><em>Figure 2.1 – The line sleep(20) added to zombie.c</em></p>

### 2.2 Look at the process table

```
vboxuser@Ubuntu:~/os-lab7$ ./zombie > /dev/null &
[1] 13848
vboxuser@Ubuntu:~/os-lab7$ sleep 2; ps -o pid,ppid,stat,cmd
    PID    PPID STAT CMD
  13455   13426 Ss   /usr/bin/bash
  13848   13455 S    ./zombie
  13850   13848 Z    [ls] <defunct>
  13855   13455 R+   ps -o pid,ppid,stat,cmd
```

<p align="center"><em>Figure 2.2 – The zombie in the process table</em></p>

### 2.3 After 20 seconds

```
vboxuser@Ubuntu:~/os-lab7$ sleep 20; ps -o pid,ppid,stat,cmd
[1]+  Done                       ./zombie > /dev/null
    PID    PPID STAT CMD
  13455   13426 Ss   /usr/bin/bash
  13857   13455 R+   ps -o pid,ppid,stat,cmd
```

<p align="center"><em>Figure 2.3 – The zombie is gone</em></p>

### Observe

**1. Which letter does `ls` have in the STAT column, and what does it mean?**
It has `Z` (the line is `[ls] <defunct>`). It means zombie: the process `ls` (PID 13850) has already ended, but its parent `./zombie` (PID 13848) has not called `wait` yet, so the system keeps its entry in the process table (with the exit status) until the parent collects it. The parent is in the state `S`, because it is sleeping in `sleep(20)`.

**2. After 20 seconds, is the `[ls] <defunct>` line still there? Which line of `zombie.c` made it disappear?**
No, it is not there any more (Figure 2.3), and `./zombie` also finished (`Done`). The line `waitpid(pid, &status, 0);` made it disappear: after `sleep(20)` the parent called `waitpid`, which took the exit status of the child and let the system remove its entry. After that the parent printed the message and ended.


## Part 3. Redirection: ls -l > out.txt

The program does what the shell does for `ls -l > out.txt`: in the child it closes descriptor 1, opens the file (which gets the smallest free number, 1), and then runs `ls`.

### 3.1 The program redirect.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>
/* does the same as the shell command: ls -l > out.txt */
int main(void)
{
    pid_t pid = fork();
    if (pid == 0) {
        close(1);                       /* descriptor 1 is now free */
        int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (fd < 0) { perror("open"); exit(1); }
        fprintf(stderr, "open gave descriptor %d\n", fd);
        execlp("ls", "ls", "-l", (char *)NULL);   /* ls writes to 1, as always */
        perror("exec");
        exit(127);
    }
    waitpid(pid, NULL, 0);
    printf("done: the listing is in out.txt\n");
    return 0;
}
```

<p align="center"><em>Figure 3.1 – Contents of redirect.c</em></p>

### 3.2 Compile, run and look inside the file

```
vboxuser@Ubuntu:~/os-lab7$ gcc -Wall -Wextra -o redirect redirect.c
vboxuser@Ubuntu:~/os-lab7$ ./redirect
open gave descriptor 1
done: the listing is in out.txt
vboxuser@Ubuntu:~/os-lab7$ head -3 out.txt
total 116
-rw-r--r-- 1 vboxuser vboxuser     0 Nov  1 18:58 out.txt
-rwxrwxr-x 1 vboxuser vboxuser 16336 Nov  1 18:58 redirect
```

<p align="center"><em>Figure 3.2 – The listing is in out.txt, not on the screen</em></p>

The file `out.txt` is shown with size 0 in its own listing: `ls` reads the directory when the file has just been created and is still empty.

### Observe

**1. Which number did `open` give, and why exactly that one?**
It gave descriptor 1 (`open gave descriptor 1`). The function `open` always returns the smallest free descriptor number. Descriptors 0 (stdin) and 2 (stderr) were still used, and the number 1 was just freed by `close(1)`, so it was the smallest free one.

**2. `ls` was not changed at all. Why does its listing not appear on the screen?**
`ls` always writes its listing to descriptor 1 and does not know where it points. In the child, descriptor 1 now points to `out.txt` and not to the terminal, and `exec` keeps the open descriptors, so the listing went into the file. The message `done: ...` was printed by the parent, whose descriptor 1 was not changed and still points to the screen. The line `open gave descriptor 1` was written to stderr (descriptor 2), which still pointed to the screen too.

**3. To redirect the errors instead (like `2> err.txt`), which descriptor would you close before `open`?**
Descriptor 2 (stderr): `close(2)`, then `open` gives the number 2 and everything that `ls` writes as an error goes to the file.


## Part 4. A pipe: ls /etc | wc -l

The program builds the pipeline by hand: one pipe and two forks. In `ls`, descriptor 1 becomes the writing end `p[1]`, in `wc` descriptor 0 becomes the reading end `p[0]`. The function `dup` copies a descriptor into the smallest free number, exactly like `open`.

### 4.1 The program pipeline.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
/* does the same as the shell command: ls /etc | wc -l */
int main(void)
{
    int p[2];
    if (pipe(p) < 0) { perror("pipe"); return 1; }
    if (fork() == 0) {                  /* ls: its 1 becomes p[1] */
        close(1);
        dup(p[1]);                      /* smallest free number: 1 */
        close(p[0]);
        close(p[1]);
        execlp("ls", "ls", "/etc", (char *)NULL);
        perror("exec ls");
        exit(127);
    }
    if (fork() == 0) {                  /* wc: its 0 becomes p[0] */
        close(0);
        dup(p[0]);                      /* smallest free number: 0 */
        close(p[0]);
        close(p[1]);
        execlp("wc", "wc", "-l", (char *)NULL);
        perror("exec wc");
        exit(127);
    }
    close(p[0]);                        /* the parent uses neither end */
    close(p[1]);
    wait(NULL);
    wait(NULL);
    return 0;
}
```

<p align="center"><em>Figure 4.1 – Contents of pipeline.c</em></p>

### 4.2 Compare with the real shell

```
vboxuser@Ubuntu:~/os-lab7$ gcc -Wall -Wextra -o pipeline pipeline.c
vboxuser@Ubuntu:~/os-lab7$ ./pipeline
237
vboxuser@Ubuntu:~/os-lab7$ ls /etc | wc -l
237
```

<p align="center"><em>Figure 4.2 – Our pipeline and the shell give the same number</em></p>

### 4.3 Break it on purpose

In the copy `pipeline_broken.c` I turned the line `close(p[1]);` (in the parent, after the comment "the parent uses neither end") into a comment:

```
vboxuser@Ubuntu:~/os-lab7$ sed '/the parent uses neither end/{n;s|close(p\[1\]);|/* close(p[1]); */|}' pipeline.c > pipeline_broken.c
vboxuser@Ubuntu:~/os-lab7$ grep -n -A1 "parent uses neither" pipeline_broken.c
28:    close(p[0]);                        /* the parent uses neither end */
29-    /* close(p[1]); */
vboxuser@Ubuntu:~/os-lab7$ gcc -Wall -Wextra -o pipeline_broken pipeline_broken.c
vboxuser@Ubuntu:~/os-lab7$ ./pipeline_broken &
[1] 13911
vboxuser@Ubuntu:~/os-lab7$ ps -o pid,ppid,stat,cmd
    PID    PPID STAT CMD
  13455   13426 Ss   /usr/bin/bash
  13911   13455 S    ./pipeline_broken
  13914   13911 S    wc -l
  13921   13455 R+   ps -o pid,ppid,stat,cmd
vboxuser@Ubuntu:~/os-lab7$ kill %1
vboxuser@Ubuntu:~/os-lab7$ 237
[1]+  Terminated              ./pipeline_broken
```

<p align="center"><em>Figure 4.3 – The broken pipeline hangs: wc -l is still alive</em></p>

I started the broken program in the background (instead of waiting and pressing Ctrl-C) and looked at the process table: `ls` is already gone, and `wc -l` is still there in the state `S`. After `kill` of the parent, its copy of `p[1]` was closed, `wc` got the end of the data and printed `237`, so no hanging process was left.

### Observe

**1. Did `./pipeline` and `ls /etc | wc -l` give the same number?**
Yes, both printed 237 (Figure 4.2).

**2. After the change, which of the two children never ends, `ls` or `wc`?**
`wc` never ends. In Figure 4.3 the process `wc -l` (PID 13914) is still in the table, and `ls` has already finished and is not there.

**3. Why, in one sentence: what is `wc` waiting for?**
`wc` waits for the end of the data (EOF) on the pipe, and EOF comes only when all the writing ends `p[1]` are closed, but the parent still keeps its own copy of `p[1]` open, so `wc` waits forever (and the parent waits for `wc` in the second `wait`).


## Conclusion

In this work I saw how the shell does its basic jobs with only a few system calls. With `fork`, `execlp` and `waitpid` I made a child that becomes `ls`, `date` or a program that does not exist, and saw that the parent gets the status 0 or 127 and that the code after a successful `exec` never runs. By delaying `waitpid` I made a zombie and saw it with the letter `Z` in `ps`, and then saw it disappear when the parent waited for it. With `close` and `open` I redirected the output of `ls` to a file without changing `ls`, because `open` returns the smallest free descriptor. With `pipe` and `dup` I built `ls /etc | wc -l` by hand, and got the same number as the shell (237). When I did not close the writing end of the pipe in the parent, `wc` never got EOF and the pipeline hung, so I understood why every process must close the pipe ends that it does not use.
