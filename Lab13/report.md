# Lab 13: Semaphores in Your Own Kernel (xv6)

**MINISTRY OF EDUCATION, CULTURE AND RESEARCH**  
**OF THE REPUBLIC OF MOLDOVA**

**Technical University of Moldova**  
**Faculty of Computers, Informatics and Microelectronics**  
**Department of Software Engineering and Automation**

**MAXIM ORMANJI FAF-242**

Checked by: Reițman Petricia

Chișinău, 2026

## Purpose of the work

In Labs 11 and 12 I used semaphores from the C library. In this work I built them myself, inside the xv6 kernel, as three new system calls: `sem_init`, `sem_down` and `sem_up` (numbers 24, 25, 26). A semaphore turned out to be only a number, a spinlock that protects the number, and the kernel's `sleep` and `wakeup`. A process that waits really sleeps and uses no CPU. At the end I used a semaphore as a lock to keep two processes from mixing their output.

## Before you start

I work in the same virtual machine as before, in my tree `~/xv6-riscv` (with `trace` from Lab 10). Numbers 22 (`sync`) and 23 (`trace`) are already used, so the new calls get 24, 25 and 26. Our kernel sleeps in two steps, `sleep_prepare(chan)` and then `sleep()`, not with `sleep(chan, lock)` as in the xv6 book, so I used the code from the lab text.

## Part 1. The numbers and the user side

The recipe is the same as for `trace` in Lab 10, three times: a number, a declaration, a stub.

```
vboxuser@Ubuntu:~/xv6-riscv$ printf '#define SYS_sem_init 24\n#define SYS_sem_down 25\n#define SYS_sem_up 26\n' >> kernel/syscall.h && sed -i 's/^int sync(void);/&\nint sem_init(int, int);\nint sem_down(int);\nint sem_up(int);/' user/user.h && printf '\nentry("sem_init");\nentry("sem_down");\nentry("sem_up");\n' >> user/usys.pl && grep -n "sem_" kernel/syscall.h user/user.h user/usys.pl
kernel/syscall.h:26:#define SYS_sem_init 24
kernel/syscall.h:27:#define SYS_sem_down 25
kernel/syscall.h:28:#define SYS_sem_up 26
user/user.h:28:int sem_init(int, int);
user/user.h:29:int sem_down(int);
user/user.h:30:int sem_up(int);
user/usys.pl:49:entry("sem_init");
user/usys.pl:50:entry("sem_down");
user/usys.pl:51:entry("sem_up");
```

<p align="center"><em>Figure 1.1 – Numbers, declarations and stubs</em></p>

There are three lines in each of the three files, and the numbers 24-26 come right after 23 (`trace`), so nothing overlaps.

Then I created the two user programs. `semtest.c` has two tests. In test 1 the child must wait for the parent (semaphore 0 starts at 0). In test 2 `ping` and `pong` take turns with two semaphores that pass the turn to each other.

```c
// semtest.c: two tests for the kernel semaphores (Lab 13).
#include "kernel/types.h"
#include "user/user.h"
int
main(void)
{
  // Test 1: order. Semaphore 0 starts at 0, so the child must wait.
  sem_init(0, 0);
  if (fork() == 0) {
    sem_down(0);
    printf("child: I ran after the parent\n");
    exit(0);
  }
  pause(10);
  printf("parent: done, I let the child go\n");
  sem_up(0);
  wait(0);
  // Test 2: ping-pong. Semaphore 1 = ping's turn, semaphore 2 = pong's turn.
  sem_init(1, 1);
  sem_init(2, 0);
  if (fork() == 0) {
    for (int i = 1; i <= 3; i++) {
      sem_down(2);
      printf("  pong %d\n", i);
      sem_up(1);
    }
    exit(0);
  }
  for (int i = 1; i <= 3; i++) {
    sem_down(1);
    printf("ping %d\n", i);
    sem_up(2);
  }
  wait(0);
  printf("semtest: OK\n");
  exit(0);
}
```

`mix.c` starts two processes that print five lines each, one letter per `write()`. With the argument `lock` it uses semaphore 3 (started at 1) as a lock around one line. I need it in Part 3.

```c
// mix.c: two processes print five lines each, one letter per write().
//   mix        no lock: the letters of the two processes can mix
//   mix lock   semaphore 3 used as a lock: every line stays whole
#include "kernel/types.h"
#include "user/user.h"
void
lines(char *s, int uselock)
{
  for (int k = 0; k < 5; k++) {
    if (uselock)
      sem_down(3);
    for (char *p = s; *p; p++)
      write(1, p, 1);
    if (uselock)
      sem_up(3);
  }
}
int
main(int argc, char *argv[])
{
  int uselock = (argc > 1);
  sem_init(3, 1);
  if (fork() == 0) {
    lines("bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb\n", uselock);
    exit(0);
  }
  lines("AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\n", uselock);
  wait(0);
  exit(0);
}
```

```
vboxuser@Ubuntu:~/xv6-riscv$ ls -l user/semtest.c user/mix.c
-rw-rw-r-- 1 vboxuser vboxuser 702 Nov 16 13:44 user/mix.c
-rw-rw-r-- 1 vboxuser vboxuser 809 Nov 16 13:44 user/semtest.c
```

<p align="center"><em>Figure 1.2 – The two user programs</em></p>

Both programs must be on the disk image, so I added them to `UPROGS` in the `Makefile`, after `_zombie` (the lines start with a TAB):

```
vboxuser@Ubuntu:~/xv6-riscv$ grep -n -P '^\t\$U/_(zombie|semtest|mix)\\$' Makefile
155:	$U/_zombie\
156:	$U/_semtest\
157:	$U/_mix\
```

<p align="center"><em>Figure 1.3 – Makefile: both programs in UPROGS</em></p>

The three lines `_zombie`, `_semtest`, `_mix` are in a row, with a TAB at the start.

### Check 1

Check 1 of the lab: the calls reach the kernel, but the kernel does not know them yet. To see it, I temporarily commented out the three entries `[SYS_sem_init]`, `[SYS_sem_down]`, `[SYS_sem_up]` in the table of `kernel/syscall.c`, rebuilt the kernel and ran `semtest`:

```
<<< PASTE HERE THE REAL OUTPUT OF `semtest` WITH THE THREE TABLE ENTRIES COMMENTED OUT >>>
<<< it must contain: "semtest: unknown sys call 24", "semtest: unknown sys call 25",
<<< and the line "child: ..." BEFORE the line "parent: ..." >>>
```

<p align="center"><em>Figure 1.4 – Check 1: unknown sys call 24 and 25</em></p>

The kernel does not have the numbers 24 and 25 in its table, so it prints `unknown sys call` and returns -1. Because of this `sem_down` does not wait at all, and the child prints its line BEFORE the parent: without semaphores the child ran first. This is exactly what Part 2 fixes. After the check I put the three table entries back (Part 2).

## Part 2. The kernel side

The file `kernel/sem.c` is new. There is a table of `NSEM = 8` semaphores, shared by all processes. Each one is a number `value` and a spinlock `lock` that protects it. `sem_down` is the heart: while the value is 0, the process registers the address of the semaphore as the thing it waits for (`sleep_prepare(s)`), lets go of the spinlock and sleeps (`sleep()`). `sem_up` adds 1 and wakes up everyone who sleeps on that address (`wakeup(s)`).

```c
// sem.c: counting semaphores inside the xv6 kernel (Lab 13).
// A small table of NSEM semaphores, shared by every process.
// Each one is a number plus a spinlock that protects the number.
#include "types.h"
#include "riscv.h"
#include "defs.h"
#include "param.h"
#include "spinlock.h"
#include "proc.h"
#define NSEM 8
struct sem {
  struct spinlock lock;   // a zeroed spinlock is a free spinlock
  int value;              // how many may still pass
};
static struct sem sems[NSEM];
// sem_init(id, value): give semaphore id its start value.
uint64
sys_sem_init(void)
{
  int id, value;
  argint(0, &id);
  argint(1, &value);
  if (id < 0 || id >= NSEM || value < 0)
    return -1;
  acquire(&sems[id].lock);
  sems[id].value = value;
  release(&sems[id].lock);
  return 0;
}
// sem_down(id): take one unit; if there is none, sleep until there is.
uint64
sys_sem_down(void)
{
  int id;
  argint(0, &id);
  if (id < 0 || id >= NSEM)
    return -1;
  struct sem *s = &sems[id];
  acquire(&s->lock);
  while (s->value == 0) {
    sleep_prepare(s);          // I will sleep on the address of s
    release(&s->lock);         // never sleep holding a spinlock
    sleep();                   // returns at once if sem_up came in between
    acquire(&s->lock);
    if (killed(myproc())) {    // kill must still work on a waiting process
      release(&s->lock);
      return -1;
    }
  }
  s->value--;
  release(&s->lock);
  return 0;
}
// sem_up(id): give one unit back and wake whoever sleeps on s.
uint64
sys_sem_up(void)
{
  int id;
  argint(0, &id);
  if (id < 0 || id >= NSEM)
    return -1;
  struct sem *s = &sems[id];
  acquire(&s->lock);
  s->value++;
  wakeup(s);
  release(&s->lock);
  return 0;
}
```

Then I put `sem.o` into the `OBJS` list of the `Makefile`, right after `sysproc.o`, and added the three calls to the table of system calls in `kernel/syscall.c`:

```
vboxuser@Ubuntu:~/xv6-riscv$ ls -l kernel/sem.c && grep -n 'sysproc.o\|sem.o' Makefile
-rw-rw-r-- 1 vboxuser vboxuser 1706 Nov 16 13:43 kernel/sem.c
20:  $K/sysproc.o \
21:  $K/sem.o \

vboxuser@Ubuntu:~/xv6-riscv$ grep -n 'sem' kernel/syscall.c
106:extern uint64 sys_sem_init(void);
107:extern uint64 sys_sem_down(void);
108:extern uint64 sys_sem_up(void);
137:  [SYS_sem_init] = sys_sem_init,
138:  [SYS_sem_down] = sys_sem_down,
139:  [SYS_sem_up] = sys_sem_up,
```

<p align="center"><em>Figure 2.1 – sem.o in OBJS and the three entries in the table</em></p>

There are three `extern` lines and three table entries. The line 146 of `syscall.c` (the names table for `trace` from Lab 10) is not changed.

### Check 2

I built the kernel, started xv6 and ran `semtest`:

```
vboxuser@Ubuntu:~/xv6-riscv$ make -s kernel/kernel fs.img; make qemu
...
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ semtest
parent: done, I let the child go
child: I ran after the parent
ping 1
  pong 1
ping 2
  pong 2
ping 3
  pong 3
semtest: OK
```

<p align="center"><em>Figure 2.2 – semtest</em></p>

The output is the same as the expected one. The parent waits 10 ticks before it lets the child go, yet the child prints second: it was asleep in `sem_down` all this time. Ping and pong alternate strictly, three times each, and the program ends with `semtest: OK`. The build only printed the usual warnings `LOAD segment with RWX permissions`, they do not change the result.

## Part 3. A semaphore as a lock: mix

`mix` starts two processes that each print five lines of 30 letters, one letter per `write()`. `make qemu` uses 3 CPUs, so the two processes really run at the same time. I ran it twice without a lock and once with the lock (semaphore 3, started at 1).

```
$ mix
AAbbbbbbbbbbbbbbbbAbbbAAAAAAbbbbAAAAbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbA
bAbAbAbAbAbAbAbAbAbAbAbbbbbbbbbbbbbbbbbbb
AAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
$ mix
AAAAAAAAAAAAAAAAAAAAbAbAbAAAAAAAA
AAAAAAAAAAAAAAAbbbbbbbbAAAAAAAAAAAAAAA
AAbbbbbAAbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
AAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

<p align="center"><em>Figure 3.1 – mix without the lock, two runs</em></p>

In both runs the letters of the two processes are mixed: there are lines that are too short or too long, and places where `A` and `b` alternate letter by letter (`bAbAbAbAbAbAbAb...`). Each time the mixing is different.

```
$ mix lock
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
bbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
```

<p align="center"><em>Figure 3.2 – mix with the lock</em></p>

With the lock every line is whole: first five lines of `A`, then five lines of `b`. The two processes do not mix any more, one of them finishes its line before the other starts the next one.

### Observe

**1. Without the lock, why do the letters mix? (How many system calls does one line cost?)**
One line has 30 letters and a newline, and every letter is a separate `write()` system call, so one line costs 31 system calls. The system can switch from one process to the other (or the other process, on another CPU, can call `write()` at the same time) between any two of these calls. Then the letters of the two processes go to the screen one after another, in a mixed order.

**2. With the lock every line is whole. Which two lines of `mix.c` make one line a critical section?**
The lines `if (uselock) sem_down(3);` before the loop with `write()`, and `if (uselock) sem_up(3);` after it. Semaphore 3 starts at 1, so only one process can be between them. The other process waits in `sem_down(3)` until the first one gives the unit back with `sem_up(3)`, and the whole line is printed without interruption.

**3. In `sem_down` the test is `while (s->value == 0)`, not `if`. Why can a process wake up and still find 0?**
Between the wake-up and the moment when the process takes the spinlock again, another process can run `sem_down` and take the unit. Also `sem_up` wakes everyone who sleeps on this semaphore, but it added only one unit, so only one of them can take it. The other processes wake up and find `value == 0` again, so they must go back to sleep. With `while` the value is checked again after every wake-up; with `if` the process would go on and make the value negative.

**4. `sem_down` calls `release(&s->lock)` BEFORE `sleep()`. What would go wrong if it slept holding the spinlock?**
The process would sleep with the lock taken. Then `sem_up` of the other process could not take the same lock (`acquire` would wait for it forever), so nobody could increase the value or call `wakeup`, and the sleeping process would never wake up: a deadlock, like in Lab 12. All other processes that use this semaphore would also stay in `acquire`. In addition, a spinlock keeps the interrupts off, and the kernel must not sleep with them off.

## Conclusion

In this work I built semaphores inside the xv6 kernel. I added three system calls (`sem_init`, `sem_down`, `sem_up` with numbers 24-26) in the same way as `trace` in Lab 10: a number, a declaration and a stub on the user side, and an entry in the table on the kernel side. A semaphore in the kernel is only an integer with a spinlock; `sem_down` sleeps while the value is 0 (and releases the spinlock before it sleeps), and `sem_up` adds one and wakes up the sleepers. `semtest` showed that the child really waits for the parent and that ping and pong alternate strictly, and it ended with `semtest: OK`. In `mix` the same semaphore, started at 1, worked as a lock: without it the letters of two processes mixed in both runs, and with it every line was whole. In Check 1, with the table entries removed, the kernel printed `unknown sys call 24` and `unknown sys call 25` and the child ran before the parent, which is the behaviour that the semaphores fix.
