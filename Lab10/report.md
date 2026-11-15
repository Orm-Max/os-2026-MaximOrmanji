# Lab 10: Add Your Own System Call: trace

**MINISTRY OF EDUCATION, CULTURE AND RESEARCH**  
**OF THE REPUBLIC OF MOLDOVA**  

**Technical University of Moldova**  
**Faculty of Computers, Informatics and Microelectronics**  
**Department of Software Engineering and Automation**  


**MAXIM ORMANJI FAF-242**

Checked by: Reițman Petricia

Chișinău, 2026

## Purpose of the work

The purpose of this work is to change the kernel itself for the first time. I added a new system call `trace` to xv6. It makes a process print every system call it makes (or only the chosen ones). I followed one number, 23, from a user program through the `ecall` instruction down to a function in the kernel. In Lab 9 `strace` did this for me, and now I built the same idea myself.


## Before you start

I used my `~/xv6-riscv` tree. In our version of xv6 the number 22 is already used by `sync`, and the kernel print function is called `printk`, so I used the number 23 and `printk` (the MIT instructions for 6.1810 say 22 and `printf`).


## Part 1. The number and the user side

First the call gets a number, then the user side learns to ask for it. I made five small edits.

### 1.1 kernel/syscall.h: the number

```
vboxuser@Ubuntu:~/xv6-riscv$ printf '\n#define SYS_trace 23\n' >> kernel/syscall.h
```

```c
#define SYS_sync  22
#define SYS_trace 23 // new
```

### 1.2 user/user.h: the declaration

```
vboxuser@Ubuntu:~/xv6-riscv$ sed -i 's/^int sync(void);/&\nint trace(int);/' user/user.h
```

```c
int sync(void);
int trace(int); // new
```

### 1.3 user/usys.pl: the stub that runs ecall

```
vboxuser@Ubuntu:~/xv6-riscv$ printf '\nentry("trace");\n' >> user/usys.pl
```

```perl
entry("sync");
entry("trace"); # new
```

At build time this script writes `user/usys.S`, where `trace` becomes three instructions: `li a7, SYS_trace`, `ecall`, `ret`.

### 1.4 user/trace.c: the program

```
vboxuser@Ubuntu:~/xv6-riscv$ cat > user/trace.c <<'EOF'
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if (argc < 3) {
    fprintf(2, "usage: trace mask command [args...]\n");
    exit(1);
  }
  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "trace: failed\n");
    exit(1);
  }
  exec(argv[2], argv + 2);
  fprintf(2, "trace: exec %s failed\n", argv[2]);
  exit(1);
}
EOF
```

The program sets the mask on itself and then becomes the command with `exec`. The mask stays, because `exec` keeps the same process.

### 1.5 Makefile: put trace on the xv6 disk

```
vboxuser@Ubuntu:~/xv6-riscv$ sed -i 's|^\t\$U/_zombie\\$|&\n\t$U/_trace\\|' Makefile
vboxuser@Ubuntu:~/xv6-riscv$ grep -n "trace" kernel/syscall.h user/user.h user/usys.pl
kernel/syscall.h:25:#define SYS_trace 23
user/user.h:28:int trace(int);
user/usys.pl:47:entry("trace");
vboxuser@Ubuntu:~/xv6-riscv$ grep -n -P '^\t\$U/_(zombie|trace)\\$' Makefile
154:	$U/_zombie\
155:	$U/_trace\
```

<p align="center"><em>Figure 1.1 – The edits of Part 1</em></p>


The Makefile line starts with a TAB and nothing follows the backslash; `grep -P` with `\t` confirmed it.

### Check 1: the number reaches the kernel

```
vboxuser@Ubuntu:~/xv6-riscv$ make qemu

xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ trace 32 grep hello README
3 trace: unknown sys call 23
trace: failed
```

<p align="center"><em>Figure 1.2 – The kernel receives the number 23, but has no function for it yet</em></p>


This message is good news: my program ran `ecall` with `a7 = 23` and the kernel received it. The kernel only did not know a function for 23, so `trace` returned -1 and the program printed `trace: failed`.


## Part 2. The kernel side

### 2.1 kernel/proc.h: a place for the mask

Each process remembers its own mask. I added a field to `struct proc`, after the `name` field:

```
vboxuser@Ubuntu:~/xv6-riscv$ sed -i '/char name\[16\];/a\  int trace_mask;               // new: which calls to print' kernel/proc.h
```

```c
  char name[16];               // Process name (debugging)
  int trace_mask;              // new: which calls to print
```

### 2.2 kernel/sysproc.c: the handler

```
vboxuser@Ubuntu:~/xv6-riscv$ cat >> kernel/sysproc.c <<'EOF'

uint64
sys_trace(void)
{
  int mask;
  argint(0, &mask);
  myproc()->trace_mask = mask;
  return 0;
}
EOF
```

The handler reads argument 0 (register `a0`) with `argint` and stores it in the process.

### 2.3 kernel/syscall.c: the table

```
vboxuser@Ubuntu:~/xv6-riscv$ sed -i 's|^extern uint64 sys_sync(void);|&\nextern uint64 sys_trace(void); // new|' kernel/syscall.c
vboxuser@Ubuntu:~/xv6-riscv$ sed -i 's|^\(\s*\)\[SYS_sync\]\s*= sys_sync,|&\n\1[SYS_trace] = sys_trace, // new|' kernel/syscall.c
```

```c
extern uint64 sys_sync(void);
extern uint64 sys_trace(void); // new
...
  [SYS_sync]  = sys_sync,
  [SYS_trace] = sys_trace, // new
```

### 2.4 kernel/syscall.c: the names and the print

The kernel knows only numbers, so I added a table of names above the function `syscall()`:

```c
static char *syscallnames[] = {
  [SYS_fork] "fork", [SYS_exit] "exit", [SYS_wait] "wait",
  [SYS_pipe] "pipe", [SYS_read] "read", [SYS_kill] "kill",
  [SYS_exec] "exec", [SYS_fstat] "fstat", [SYS_chdir] "chdir",
  [SYS_dup] "dup", [SYS_getpid] "getpid", [SYS_sbrk] "sbrk",
  [SYS_pause] "pause", [SYS_uptime] "uptime", [SYS_open] "open",
  [SYS_write] "write", [SYS_mknod] "mknod", [SYS_unlink] "unlink",
  [SYS_link] "link", [SYS_mkdir] "mkdir", [SYS_close] "close",
  [SYS_sync] "sync", [SYS_trace] "trace",
};
```

Then, inside `syscall()`, right after the line that runs the call, I added two lines. They are after the call, so the return value in `a0` is already there:

```c
    p->trapframe->a0 = syscalls[num]();
    if ((p->trace_mask >> num) & 1) // new
      printk("%d: syscall %s -> %ld\n", p->pid, syscallnames[num], p->trapframe->a0); // new
```

### 2.5 kernel/proc.c: children inherit the mask

```
vboxuser@Ubuntu:~/xv6-riscv$ sed -i 's|^\(\s*\)safestrcpy(np->name, p->name, sizeof(p->name));|&\n\1np->trace_mask = p->trace_mask; // new|' kernel/proc.c
```

```c
  safestrcpy(np->name, p->name, sizeof(p->name));
  np->trace_mask = p->trace_mask; // new
```

I checked that all the edits of Part 2 are in the files:

```
vboxuser@Ubuntu:~/xv6-riscv$ grep -n "trace_mask" kernel/proc.h kernel/sysproc.c kernel/syscall.c kernel/proc.c
kernel/proc.h:104:  int trace_mask;               // new: which calls to print
kernel/sysproc.c:119:  myproc()->trace_mask = mask;
kernel/syscall.c:160:    if ((p->trace_mask >> num) & 1) // new
kernel/proc.c:291:  np->trace_mask = p->trace_mask; // new
vboxuser@Ubuntu:~/xv6-riscv$ grep -n "syscallnames\|sys_trace\|SYS_trace" kernel/syscall.c
106:extern uint64 sys_trace(void); // new
134:  [SYS_trace] = sys_trace, // new
138:static char *syscallnames[] = {
141:  [SYS_sync] "sync", [SYS_trace] "trace",
161:      printk("%d: syscall %s -> %ld\n", p->pid, syscallnames[num], p->trapframe->a0); // new
vboxuser@Ubuntu:~/xv6-riscv$ git diff --stat
 Makefile         |  9 +++++++++
 kernel/proc.c    |  1 +
 kernel/proc.h    |  1 +
 kernel/syscall.c | 15 +++++++++++++++
 kernel/syscall.h |  2 ++
 kernel/sysproc.c |  9 +++++++++
 user/user.h      |  1 +
 user/usys.pl     |  2 ++
 8 files changed, 40 insertions(+)
```

<p align="center"><em>Figure 2.1 – The edits of Part 2 in the source files</em></p>


### Check 2: trace works

```
vboxuser@Ubuntu:~/xv6-riscv$ make qemu

xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 965
3: syscall read -> 453
3: syscall read -> 0
$
```

<p align="center"><em>Figure 2.2 – trace 32 grep hello README</em></p>


The numbers are exactly the expected ones: 1023 + 965 + 453 = 2441, the size of `README`, read in three pieces, and then 0 for the end of the file.


## Part 3. Play with the mask

The mask is a number read in binary: the bit N on means "print the call number N". For example, 32 = 2^5, and 5 is `SYS_read`.

### 3.1 All the calls (all bits on)

```
$ trace 2147483647 grep hello README
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 965
4: syscall read -> 453
4: syscall read -> 0
4: syscall close -> 0
```

<p align="center"><em>Figure 3.1 – All system calls of grep hello README</em></p>


With all bits on I see the full life of the process: first the call `trace` itself (the mask is set during this call, so it is printed when the call returns), then `exec`, which started `grep`, then `open` of `README` (descriptor 3), the three `read` calls with 0 at the end, and `close`. This is the same sequence that `strace` showed in Lab 9, but much shorter, because xv6 has no dynamic libraries.

### 3.2 Only fork (bit 1, mask 2)

```
$ trace 2 usertests forkforkfork
usertests starting
test forkforkfork: 5: syscall fork -> 6
6: syscall fork -> 7
7: syscall fork -> 8
7: syscall fork -> 9
8: syscall fork -> 10
8: syscall fork -> 11
8: syscall fork -> 12
7: syscall fork -> 13
...
7: syscall fork -> 65
7: syscall fork -> -1
9: syscall fork -> -1
8: syscall fork -> 66
OK
ALL TESTS PASSED
```

<p align="center"><em>Figure 3.2 – Only fork calls in forkforkfork (shortened)</em></p>


The test forks until the process table is full. Many different pids print lines, and at the end `fork -> -1` shows that there is no free place for a new process. The test passed (`ALL TESTS PASSED`).

### 3.3 Only write (bit 16, mask 65536)

```
$ trace 65536 echo hi
hi67: syscall write -> 2

67: syscall write -> 1
```

<p align="center"><em>Figure 3.3 – Only write calls of echo hi</em></p>


### 3.4 Only open and close (mask 2129920)

```
$ trace 2129920 grep hello README
6: syscall open -> 3
6: syscall close -> 0
```

<p align="center"><em>Figure 3.4 – Only open and close</em></p>


### Observe

**1. In `trace 32 grep hello README` why do I see only `read`, and not `open` or `close`?**
The mask 32 is 2^5, so only the bit 5 is on. The number 5 is `SYS_read`. The number of `open` is 15 and of `close` is 21, and their bits are off. The kernel prints a call only if `(p->trace_mask >> num) & 1` is true, so `open` and `close` run, but they are not printed.

**2. In `forkforkfork`, three different pids print lines. Which line of code from Part 2 makes that possible?**
The line `np->trace_mask = p->trace_mask;` in `fork()` (`kernel/proc.c`, Part 2.5). The child copies the mask of its parent, so every new process is also traced. Without this line only the first process (the one I started with `trace`) would print its `fork` calls, because the children would start with the mask 0. In my output many pids (5, 6, 7, 8, 9, ...) print lines, and this is the proof.

**3. In `trace 65536 echo hi` the trace line is printed between "hi" and the new line. Why? How many times does `echo` call `write`?**
`echo` calls `write` two times: first it writes the text `hi` (2 bytes, the result is 2) and then it writes the new line character (1 byte, the result is 1). The kernel prints the trace line right after each call. So the screen shows `hi` (the first `write`), then the trace line `67: syscall write -> 2` (and it begins on the same screen line, just after `hi`), and only then the second `write` prints the new line. The trace line of the second call comes after it.

**4. Which mask traces both `open` (15) and `close` (21), and nothing else?**
The mask with only these two bits on: 2^15 + 2^21 = 32768 + 2097152 = **2129920**. I tested it with `trace 2129920 grep hello README` and the output has only `open -> 3` and `close -> 0`.


## Conclusion

In this work I added the system call `trace` to xv6 and followed the number 23 all the way. On the user side the declaration in `user.h` and the stub from `usys.pl` put 23 in the register `a7` and run `ecall`. On the kernel side the number is used as an index in the table `syscalls[]`, and the function `sys_trace` stores the mask in the new field `trace_mask` of the process. Check 1 showed that the call reaches the kernel even before the handler exists (the message `unknown sys call 23`), and Check 2 gave exactly the expected numbers 1023, 965, 453 and 0. In Part 3 I learned that the mask is a set of bits (bit N is the call number N), that I can build any mask by adding the powers of two (for example 2^15 + 2^21 = 2129920 for `open` and `close`), and that `fork` must copy the mask to the child, otherwise only one process would be traced. I also saw that the trace line is printed after the call, so the output of the traced program and of the kernel can mix in the terminal.
