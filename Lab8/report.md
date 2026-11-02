# Lab 8: The Same Calls, Inside Your Own OS

**MINISTRY OF EDUCATION, CULTURE AND RESEARCH**  
**OF THE REPUBLIC OF MOLDOVA**  

**Technical University of Moldova**  
**Faculty of Computers, Informatics and Microelectronics**  
**Department of Software Engineering and Automation**  


**MAXIM ORMANJI FAF-242**

Checked by: Reițman Petricia

Chișinău, 2026

## Purpose of the work

The purpose of this work is to run the same program that I ran on Linux in Lab 7 (`fork` + `exec` + `wait`) inside xv6, the small OS that I build this semester, and to see that the system calls are the same. Then I make two processes talk to each other through two pipes (`pingpong`).


## Before you start

I used my tree `~/xv6-riscv`. Every new program is a file `user/NAME.c` plus one line `$U/_NAME\` in the `UPROGS` list of the `Makefile`. The line starts with a TAB and nothing follows the backslash.

```
vboxuser@Ubuntu:~$ cd ~/xv6-riscv
vboxuser@Ubuntu:~/xv6-riscv$ grep -nP '^\t\$U/_(pingpong|run|runls|runnosuch)\\$' Makefile
133:	$U/_pingpong\
134:	$U/_run\
135:	$U/_runls\
136:	$U/_runnosuch\
```

<p align="center"><em>Figure 0.1 – The new lines in UPROGS</em></p>

The check with `grep -P` finds all four lines, so every line starts with a TAB and has nothing after the backslash.


## Part 1. run, inside xv6

### 1.1 The program user/run.c

The same three steps as `run.c` in Lab 7, written for xv6: there is no glibc here, only the system calls of the kernel that I compiled. The child becomes `echo`.

```c
#include "kernel/types.h"
#include "user/user.h"
// the three steps of the shell, inside xv6: echo hello from the child
int
main(int argc, char *argv[]){
  printf("parent: my pid is %d\n", getpid());
  int pid = fork();                     // step 1: copy myself
  if(pid < 0){
    printf("fork failed\n");
    exit(1);
  }
  if(pid == 0){                         // only the child runs this
    printf("child: my pid is %d\n", getpid());
    char *args[] = { "echo", "hello", "from", "the", "child", 0 };
    exec("echo", args);                 // step 2: become echo
    printf("exec failed\n");            // runs only if exec failed
    exit(1);
  }
  int status;
  wait(&status);                        // step 3: wait for the child
  printf("parent: child %d ended with status %d\n", pid, status);
  exit(0);
}
```

<p align="center"><em>Figure 1.1 – Contents of user/run.c</em></p>

### 1.2 Run it in xv6

```
vboxuser@Ubuntu:~/xv6-riscv$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ run
parent: my pid is 3
child: my pid is 4
hello from the child
parent: child 4 ended with status 0
$
```

<p align="center"><em>Figure 1.2 – run: the child became echo</em></p>

### 1.3 The child runs ls

I made a copy of the program, `user/runls.c`, where the two lines are changed to `char *args[] = { "ls", 0 };` and `exec("ls", args);`, so `run.c` stays with `echo`. The line `$U/_runls\` is in `UPROGS`.

```
vboxuser@Ubuntu:~/xv6-riscv$ sed -e 's/char \*args\[\] = {.*};/char *args[] = { "ls", 0 };/' -e 's/exec("echo", args);/exec("ls", args);/' user/run.c > user/runls.c
vboxuser@Ubuntu:~/xv6-riscv$ sed -i 's|// step 2: become echo|// step 2: become ls|' user/runls.c
vboxuser@Ubuntu:~/xv6-riscv$ grep -n 'args\[\]\|exec(' user/runls.c
14:    char *args[] = { "ls", 0 };
15:    exec("ls", args);                 // step 2: become ls
```

After `make qemu`:

```
$ runls
parent: my pid is 3
child: my pid is 4
.              1 1 1024
..             1 1 1024
README         2 2 2441
run            2 3 35888
runls          2 4 35888
runnosuch      2 5 35896
args           2 6 35272
cat            2 7 36728
echo           2 8 35592
sleep          2 9 35280
forktest       2 10 18136
grep           2 11 44184
init           2 12 35976
kill           2 13 35512
ln             2 14 35304
ls             2 15 42880
mkdir          2 16 35568
rm             2 17 35552
sh             2 18 58312
stressfs       2 19 36416
usertests      2 20 206800
grind          2 21 51824
wc             2 22 37728
zombie         2 23 34848
mystat         2 24 36232
find           2 25 37880
logstress      2 26 37560
forphan        2 27 36352
dorphan        2 28 35800
sync           2 29 34944
console        3 30 0
parent: child 4 ended with status 0
$
```

<p align="center"><em>Figure 1.3 – runls: the child became ls</em></p>

### 1.4 A program that does not exist

The third copy, `user/runnosuch.c`, has the line `exec("nosuchprog", args);`. The line `$U/_runnosuch\` is in `UPROGS`.

```
vboxuser@Ubuntu:~/xv6-riscv$ sed 's/exec("echo", args);/exec("nosuchprog", args);/' user/run.c > user/runnosuch.c
vboxuser@Ubuntu:~/xv6-riscv$ sed -i 's|// step 2: become echo|// step 2: become nosuchprog|' user/runnosuch.c
vboxuser@Ubuntu:~/xv6-riscv$ grep -n 'exec(' user/runnosuch.c
15:    exec("nosuchprog", args);                 // step 2: become nosuchprog
```

After `make qemu`:

```
$ runnosuch
parent: my pid is 7
child: my pid is 8
exec failed
parent: child 8 ended with status 1
$
```

<p align="center"><em>Figure 1.4 – exec fails for nosuchprog</em></p>

### Observe

**1. Write the three calls that `run.c` uses in xv6 and the three calls that `run.c` used on Linux in Lab 7. What is the same?**
In xv6 the three calls are `fork()`, `exec("echo", args)` and `wait(&status)`. On Linux (Lab 7) they were `fork()`, `execlp("ls", "ls", "-l", NULL)` and `waitpid(pid, &status, 0)`. The idea is the same: the three steps of the shell (copy myself, become another program, wait for the child), and in both systems the parent gets the PID of the child from `fork` and the child gets 0. Only the names and the arguments are a little different: the Linux `execlp` looks for the program in `PATH` by itself, and `waitpid` can wait for one chosen child, but the system calls under them do the same work. On Linux they are wrappers in glibc, in xv6 they are the calls of the kernel that I compiled.

**2. Who prints "hello from the child": the code of `run`, or `echo`?**
`echo` prints it. The code of `run` prints only the lines with the PIDs. After a successful `exec` the child is no longer `run`: its memory is replaced by the program `echo`, which receives the arguments `hello from the child` and prints them. The line `exec failed` is not printed in Figure 1.2, because after a successful `exec` the old code is gone.

**3. Why is the status 1 here and 127 on Linux?**
Because of the number in the `exit` line after `exec`. In the xv6 program it is `exit(1)`, and in the Linux program of Lab 7 it was `exit(127)`. The parent just prints the number that the child gave to `exit`: 1 in Figure 1.4 and 127 in Lab 7.


## Part 2. pingpong

### 2.1 The program user/pingpong.c

The parent sends one byte to the child through the first pipe. The child prints `<pid>: received ping`, writes the byte back through the second pipe and exits. The parent reads it, prints `<pid>: received pong` and waits for the child. I completed the TODO lines with `read(p2c[0], &b, 1)`, `write(c2p[1], &b, 1)`, `write(p2c[1], &b, 1)` and `read(c2p[0], &b, 1)`. I gave `b` the start value `'x'`, so the parent does not send an uninitialized byte.

```c
#include "kernel/types.h"
#include "user/user.h"
int
main(int argc, char *argv[])
{
  int p2c[2], c2p[2];                   // parent-to-child, child-to-parent
  char b = 'x';
  pipe(p2c);
  pipe(c2p);
  if(fork() == 0){
    close(p2c[1]); close(c2p[0]);       // the ends the child does not use
    read(p2c[0], &b, 1);
    printf("%d: received ping\n", getpid());
    write(c2p[1], &b, 1);
    exit(0);
  }
  close(p2c[0]); close(c2p[1]);         // the ends the parent does not use
  write(p2c[1], &b, 1);
  read(c2p[0], &b, 1);
  printf("%d: received pong\n", getpid());
  wait(0);
  exit(0);
}
```

<p align="center"><em>Figure 2.1 – Contents of user/pingpong.c</em></p>

### 2.2 Run it in xv6

The line `$U/_pingpong\` is in `UPROGS` (Figure 0.1).

```
vboxuser@Ubuntu:~/xv6-riscv$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ pingpong
4: received ping
3: received pong
$
```

<p align="center"><em>Figure 2.2 – pingpong: ping from the child comes first, then pong from the parent</em></p>

The line `ping` is printed first, by the child (pid 4). The line `pong` is printed second, by the parent (pid 3).

### Observe

**1. Why do you need TWO pipes, and not one for both directions?**
A pipe works in one direction only: what is written into the writing end is read from the reading end. If I used one pipe for both directions, the parent would write the byte and then read from the same pipe, and it could take its own byte back before the child does. The child could also not know whose data it reads. With two pipes every direction has its own channel: `p2c` (parent to child) and `c2p` (child to parent), and each process closes the two ends that it does not use.

**2. Nothing in your code says "wait until the child has replied". Which call makes the parent wait?**
The call `read(c2p[0], &b, 1)` in the parent. The pipe `c2p` is empty until the child writes the byte, so `read` blocks and the parent sleeps until the child sends the byte back. This is why `pong` always comes after `ping`. The last `wait(0)` makes the parent wait until the child process has ended.


## Conclusion

In this work I saw that the system calls that I used on Linux in Lab 7 are the same inside xv6: `fork`, `exec` and `wait` do the three steps of the shell, and the child becomes `echo`, `ls` or fails with the status that its own `exit` gives (1 here, 127 in Lab 7). I saw that the lines `hello from the child` and the listing are printed by the new program after `exec`, not by the code of `run`. With `pipe`, `read`, `write` and `close` I made two processes talk through two one-way pipes, and saw that the blocking `read` in the parent is what makes it wait for the answer of the child, so `ping` always comes before `pong`.
