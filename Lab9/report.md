# Lab 9: Watching System Calls with strace

**MINISTRY OF EDUCATION, CULTURE AND RESEARCH**  
**OF THE REPUBLIC OF MOLDOVA**  

**Technical University of Moldova**  
**Faculty of Computers, Informatics and Microelectronics**  
**Department of Software Engineering and Automation**  


**MAXIM ORMANJI FAF-242**

Checked by: Reițman Petricia

Chișinău, 2026

## Purpose of the work

The purpose of this work is to look at the door between a program and the kernel: the system call. I used `strace` to see the system calls of a real command (`ls -l`), then I wrote a small C program that uses `open`, `write`, `read` and `close` directly, and I watched how these calls look from the outside. At the end I counted only one kind of call (`openat`) and compared two commands.


## Before you start

I work on my own Linux laptop (Ubuntu). I created a separate folder for the lab and checked that `strace` is installed.

```
vboxuser@Ubuntu:~$ mkdir -p ~/os_lab9 && cd ~/os_lab9
vboxuser@Ubuntu:~/os_lab9$ sudo apt install -y strace
[sudo: authenticate] Password:
strace is already the newest version (6.19+ds-0ubuntu5).
strace set to manually installed.
Summary:
  Upgrading: 0, Installing: 0, Removing: 0, Not Upgrading: 0
```

<p align="center"><em>Figure 0.1 – Working folder and strace</em></p>


## Part 1. See the calls of a real command

### The calls of ls -l

`strace` runs a command and prints every system call beside the normal output. I asked it to show only `openat`, `read`, `write` and `close` (the output is shortened, the lines I left out are marked with `...`):

```
vboxuser@Ubuntu:~/os_lab9$ strace -f -e trace=openat,read,write,close ls -l
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libsystemd.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
close(3)                                = 0
...
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0 \250\2\0\0\0\0\0"..., 832) = 832
close(3)                                = 0
...
openat(AT_FDCWD, "/usr/share/coreutils/locales/ls/en-US.ftl", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
...
openat(AT_FDCWD, "/etc/passwd", O_RDONLY|O_CLOEXEC) = 4
read(4, "root:x:0:0:root:/root:/bin/bash\n"..., 4096) = 2823
close(4)                                = 0
openat(AT_FDCWD, "/etc/group", O_RDONLY|O_CLOEXEC) = 4
read(4, "root:x:0:\ndaemon:x:1:\nbin:x:2:\ns"..., 4096) = 1184
close(4)                                = 0
...
close(3)                                = 0
write(1, "total 24\n-rw-rw-r-- 1 vboxuser v"..., 190total 24
-rw-rw-r-- 1 vboxuser vboxuser  2466 Nov  2 13:44 output.txt
-rwxrwxr-x 1 vboxuser vboxuser 16264 Nov  2 13:44 syscalls
-rw-rw-r-- 1 vboxuser vboxuser   735 Nov  2 13:43 syscalls.c
) = 190
+++ exited with 0 +++
```

<p align="center"><em>Figure 1.1 – System calls of ls -l (shortened)</em></p>


First `ls` loads the shared libraries (`openat`, `read`, `close` for each of them, for example `libc.so.6`), then it reads some files it needs (`/etc/passwd` and `/etc/group` to change the numbers of the owner and the group into names), then it opens the directory `.` and only at the end it prints the result with one `write`. The only error is `ENOENT` for the file `ls/en-US.ftl`: this file of translations does not exist, and the program just continues.

### How many system calls does one ls make?

I counted the lines of the full trace (all calls, without the normal output) and then asked `strace` for a summary with the option `-c`:

```
vboxuser@Ubuntu:~/os_lab9$ strace -f ls -l 2>&1 >/dev/null | wc -l
281
vboxuser@Ubuntu:~/os_lab9$ strace -c -f ls -l >/dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ------------------
 14.55    0.002333          72        32           mmap
 12.29    0.001971          56        35           close
 11.04    0.001770          52        34           getdents64
 10.74    0.001722          57        30           fstat
  9.51    0.001525          47        32         1 openat
  8.07    0.001294          58        22           read
  3.82    0.000612          68         9           mprotect
  3.43    0.000550         550         1           execve
...
  0.23    0.000037          37         1           write
------ ----------- ----------- --------- --------- ------------------
100.00    0.016033          57       279        14 total
```

<p align="center"><em>Figure 1.2 – Number of system calls of one ls -l</em></p>


One simple `ls -l` makes about **280 system calls**: the summary says 279 calls (14 of them with an error), and the full trace has 281 lines (the lines with `+++ exited +++` and similar notes are not calls). The most frequent ones are `close` (35), `getdents64` (34, it reads the directory), `mmap` (32), `openat` (32) and `fstat` (30). `write` is called only once.

### Observe

**Which system call does `ls` use to print to the screen? To which descriptor does it write?**
`ls` uses the system call `write`. In the trace it is the line `write(1, "total 24\n-rw-rw-r-- 1 vboxuser v"..., 190) = 190`: the first argument is the descriptor `1`, which is the standard output (stdout), and the call returned 190, the number of bytes written. In the `strace` listing the text of `ls` is mixed with the trace line, because both go to the same terminal.


## Part 2. A program that calls the kernel directly

### The program

I typed the program `syscalls.c`. It opens the file `note.txt`, writes one line in it, closes it, opens it again and reads the line back, using `open`, `write`, `read` and `close`. `printf` is used only for the messages on the screen.

```
vboxuser@Ubuntu:~/os_lab9$ cat > syscalls.c <<'EOF'
/* syscalls.c - open, write, read and close, using only system calls. */
#include <fcntl.h>   /* open */
#include <unistd.h>  /* write, read, close */
#include <string.h>  /* strlen */
#include <stdio.h>   /* printf */

int main(void) {
    const char *msg = "hello, kernel\n";

    int fd = open("note.txt", O_CREAT | O_WRONLY | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return 1; }
    printf("open returned descriptor %d\n", fd);

    ssize_t n = write(fd, msg, strlen(msg));
    printf("write sent %zd bytes\n", n);
    close(fd);

    fd = open("note.txt", O_RDONLY);
    char buf[64];
    n = read(fd, buf, sizeof(buf) - 1);
    buf[n] = '\0';
    printf("read got %zd bytes: %s", n, buf);
    close(fd);
    return 0;
}
EOF
```

<p align="center"><em>Figure 2.1 – Contents of syscalls.c</em></p>


### Compile and run

```
vboxuser@Ubuntu:~/os_lab9$ gcc -Wall -Wextra -o syscalls syscalls.c
vboxuser@Ubuntu:~/os_lab9$ ./syscalls
open returned descriptor 3
write sent 14 bytes
read got 14 bytes: hello, kernel
```

<p align="center"><em>Figure 2.2 – Compiling and running the program</em></p>


The program compiled without warnings and the output is the same as the expected one: the descriptor is 3, 14 bytes were written (`hello, kernel` has 13 characters and the end of the line `\n` is the 14th) and the same 14 bytes were read back.

### Watching the program with strace

```
vboxuser@Ubuntu:~/os_lab9$ strace -e trace=openat,write,read,close ./syscalls
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0 \250\2\0\0\0\0\0"..., 832) = 832
close(3)                                = 0
openat(AT_FDCWD, "note.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 3
write(3, "hello, kernel\n", 14)         = 14
close(3)                                = 0
openat(AT_FDCWD, "note.txt", O_RDONLY)  = 3
read(3, "hello, kernel\n", 63)          = 14
close(3)                                = 0
write(1, "open returned descriptor 3\nwrite"..., 80open returned descriptor 3
write sent 14 bytes
read got 14 bytes: hello, kernel
) = 80
+++ exited with 0 +++
```

<p align="center"><em>Figure 2.3 – System calls of ./syscalls</em></p>


The first five lines are the setup: the dynamic loader opens `/etc/ld.so.cache` and the library `libc.so.6`. After them there are my six calls, and they are the same as in the task:

| My C code | System call in the trace | Result |
|---|---|---|
| `open("note.txt", O_CREAT \| O_WRONLY \| O_TRUNC, 0644)` | `openat(AT_FDCWD, "note.txt", O_WRONLY\|O_CREAT\|O_TRUNC, 0644)` | `= 3` |
| `write(fd, msg, strlen(msg))` | `write(3, "hello, kernel\n", 14)` | `= 14` |
| `close(fd)` | `close(3)` | `= 0` |
| `open("note.txt", O_RDONLY)` | `openat(AT_FDCWD, "note.txt", O_RDONLY)` | `= 3` |
| `read(fd, buf, sizeof(buf) - 1)` | `read(3, "hello, kernel\n", 63)` | `= 14` |
| `close(fd)` | `close(3)` | `= 0` |

Two small things are interesting. First, my `open` became `openat`: the library function `open` of `libc` calls the newer system call `openat` with `AT_FDCWD`, which means "relative to the current directory". Second, the three `printf` calls did not make three separate `write` calls. There is only one `write(1, ..., 80)` at the end: when stdout goes to a pipe or a file and not to a terminal, `printf` keeps the text in a buffer and sends it all at once when the program exits. The 80 bytes are 27 + 20 + 33, the lengths of the three messages. The second `open` got the descriptor 3 again, because the first `close(3)` freed it, and the kernel always gives the smallest free number.

### Observe

**The descriptor returned by the first `openat` is 3, not 0, 1 or 2. Why are those three already taken?**
Because every process starts with three open standard streams: `0` is the standard input (stdin), `1` is the standard output (stdout) and `2` is the standard error (stderr). The shell opens them (they point to the terminal) before it starts my program, and the program inherits them. The kernel gives to `open` the smallest descriptor number that is free, and the first free one is 3. In the trace this is also visible: the `printf` text goes to the descriptor `1`, the same one `ls` used in Part 1.


## Part 3. Count one kind of call

In Part 1 I counted all the calls together. Now I count only one kind: `openat`. I filter the trace with `-e trace=openat` and write it to a file with `-o`, so it is not mixed with the output of `ls`. Then I check the number with the summary table of `strace -c`.

```
vboxuser@Ubuntu:~/os_lab9$ strace -f -e trace=openat -o trace.txt ls -l > /dev/null; grep -c 'openat(' trace.txt; grep -c ENOENT trace.txt
32
1
vboxuser@Ubuntu:~/os_lab9$ strace -c -f ls -l > /dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ------------------
 12.63    0.001025          32        32         1 openat
 12.50    0.001014          29        34           getdents64
 12.21    0.000991          28        35           close
 11.93    0.000968          30        32           mmap
...
  0.30    0.000024          24         1           write
...
------ ----------- ----------- --------- --------- ------------------
100.00    0.008115          28       284        13 total
```

<p align="center"><em>Figure 3.1 – Counting openat in ls -l</em></p>


`ls -l` makes **32** calls of `openat`, and the row `openat` of the summary table says the same: 32. One of them failed (the column `errors` shows 1): it is the `ENOENT` for the missing `ls/en-US.ftl` file from Part 1. A failed call is still a call, so it is counted. In total this run has 284 calls, so `openat` is 32 / 284 ≈ 11.3% of them. The column `% time` (12.63%) is a share of time, not of the number of calls. The total is a little different from Part 1 (279 there), because the numbers change slightly from run to run.

### Comparing ls and ls -l

```
vboxuser@Ubuntu:~/os_lab9$ strace -f -e trace=openat -o t1.txt ls -l > /dev/null; strace -f -e trace=openat -o t2.txt ls > /dev/null; grep -c 'openat(' t1.txt t2.txt
t1.txt:32
t2.txt:12
vboxuser@Ubuntu:~/os_lab9$ grep -o '"[^"]*"' t2.txt
"/etc/ld.so.cache"
"/usr/lib/x86_64-linux-gnu/libsystemd.so.0"
"/usr/lib/x86_64-linux-gnu/libselinux.so.1"
"/usr/lib/x86_64-linux-gnu/libgcc_s.so.1"
"/usr/lib/x86_64-linux-gnu/libm.so.6"
"/usr/lib/x86_64-linux-gnu/libc.so.6"
"/usr/lib/x86_64-linux-gnu/libpcre2-8.so.0"
"/proc/filesystems"
"/proc/self/maps"
"/usr/share/coreutils/locales/uucore/en-US.ftl"
"/usr/share/coreutils/locales/ls/en-US.ftl"
"."
vboxuser@Ubuntu:~/os_lab9$ diff <(grep -o '"[^"]*"' t2.txt | sort) <(grep -o '"[^"]*"' t1.txt | sort)
1a2
> "/etc/group"
2a4,5
> "/etc/nsswitch.conf"
> "/etc/passwd"
12a16,32
> "/usr/share/zoneinfo"
> "/usr/share/zoneinfo/Africa"
> "/usr/share/zoneinfo/America"
> "/usr/share/zoneinfo/America/Argentina"
> "/usr/share/zoneinfo/America/Indiana"
> "/usr/share/zoneinfo/America/Kentucky"
> "/usr/share/zoneinfo/America/North_Dakota"
> "/usr/share/zoneinfo/Antarctica"
> "/usr/share/zoneinfo/Arctic"
> "/usr/share/zoneinfo/Asia"
> "/usr/share/zoneinfo/Atlantic"
> "/usr/share/zoneinfo/Australia"
> "/usr/share/zoneinfo/Etc"
> "/usr/share/zoneinfo/Etc/UTC"
> "/usr/share/zoneinfo/Europe"
> "/usr/share/zoneinfo/Indian"
> "/usr/share/zoneinfo/Pacific"
```

<p align="center"><em>Figure 3.2 – openat of ls and ls -l</em></p>


Plain `ls` makes 12 calls of `openat` and `ls -l` makes 32, so the option `-l` adds 20. The 12 calls of plain `ls` are the setup: `/etc/ld.so.cache`, 6 shared libraries (`libsystemd`, `libselinux`, `libgcc_s`, `libm`, `libc`, `libpcre2-8`), `/proc/filesystems`, `/proc/self/maps`, 2 files of messages (`en-US.ftl`) and the directory `.` itself (1 + 6 + 2 + 2 + 1 = 12). The `diff` shows only added lines, so `ls -l` opens everything that `ls` opens and 20 more files: `/etc/passwd` and `/etc/group` (to turn the numbers of the owner and the group into names), `/etc/nsswitch.conf` (it tells where to look for these names) and 17 paths in `/usr/share/zoneinfo` (3 + 17 = 20), which are used to print the dates in the local time zone.

### Observe

**Why is counting one kind of call better than counting all of them?**
The total (284 calls) only says that the command is busy, it does not say what it is doing. One kind of call answers a precise question: how many files does the command open? And the same count for two commands (32 against 12) shows exactly what an option changes, and the `diff` of the file names shows why.


## Conclusion

In this work I used `strace` to look at the system calls, the way a program asks the kernel to do something. With `ls -l` I saw that even a simple command makes about 280 system calls: most of them load the libraries and read the directory, and only one `write` to the descriptor 1 prints the result. Then I wrote a program with `open`, `write`, `read` and `close` and found its calls in the trace with the same arguments and results; I also saw that `open` of the C library really uses the system call `openat`, and that `printf` buffers the text when the output is not a terminal. I learned that a file descriptor is a small number given by the kernel, and that 0, 1 and 2 are already used for stdin, stdout and stderr, so the first file of my program gets the descriptor 3. In Part 3 I counted only the `openat` calls: `ls -l` makes 32 of them against 12 for plain `ls`, and the 20 extra files (users, groups, time zones) show what the option `-l` really costs.
