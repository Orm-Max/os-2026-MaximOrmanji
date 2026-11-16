# Lab 12: Producer, Consumer and the Other Tools

**MINISTRY OF EDUCATION, CULTURE AND RESEARCH**  
**OF THE REPUBLIC OF MOLDOVA**  

**Technical University of Moldova**  
**Faculty of Computers, Informatics and Microelectronics**  
**Department of Software Engineering and Automation**  


**MAXIM ORMANJI FAF-242**

Checked by: Reițman Petricia

Chișinău, 2026

## Purpose of the work

The purpose of this work is to make the tools of Lab 11 work together. I solved the two classic problems of operating systems, producer and consumer, and readers and writers. I also met the condition variable as another way to wait for an event, and I built a deadlock on purpose and then removed it.


## Before you start

I work on the same laptop and with the same tools as in Lab 11 (Ubuntu, 2 CPU cores). I created a new folder. Every program is compiled with `-pthread`.

```
vboxuser@Ubuntu:~$ mkdir -p ~/os-lab12 && cd ~/os-lab12
```


## Part 1. Producer and consumer

A producer puts items into a buffer of 4 slots, a slow consumer takes them out. Three tools work together: `empty` counts the free slots (starts at 4), `full` counts the filled slots (starts at 0), and a mutex protects the buffer itself.

```c
/* prodcons.c - producer and consumer around a buffer of 4 slots.
 * empty = free slots (starts at SIZE), full = filled slots (starts at 0), lock = the buffer itself.
 * Build: gcc -Wall -pthread -o prodcons prodcons.c
 */
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#define SIZE 4
#define ITEMS 8
int buf[SIZE];
int in = 0, out = 0, count = 0; /* count only for printing */
sem_t empty, full;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
void *producer(void *arg) {
    for (int item = 1; item <= ITEMS; item++) {
        sem_wait(&empty); /* wait for a free slot */
        pthread_mutex_lock(&lock);
        buf[in] = item;
        in = (in + 1) % SIZE;
        count++;
        printf("put %d (in buffer: %d)\n", item, count);
        pthread_mutex_unlock(&lock);
        sem_post(&full); /* one more filled slot */
    }
    return NULL;
}
void *consumer(void *arg) {
    for (int k = 0; k < ITEMS; k++) {
        sem_wait(&full); /* wait for something to take */
        pthread_mutex_lock(&lock);
        int item = buf[out];
        out = (out + 1) % SIZE;
        count--;
        printf("    take %d (in buffer: %d)\n", item, count);
        pthread_mutex_unlock(&lock);
        sem_post(&empty); /* one more free slot */
        usleep(300000); /* the consumer is slow */
    }
    return NULL;
}
int main(void) {
    sem_init(&empty, 0, SIZE);
    sem_init(&full, 0, 0);
    pthread_t p, c;
    pthread_create(&c, NULL, consumer, NULL);
    pthread_create(&p, NULL, producer, NULL);
    pthread_join(p, NULL);
    pthread_join(c, NULL);
    sem_destroy(&empty);
    sem_destroy(&full);
    return 0;
}
```

```
vboxuser@Ubuntu:~/os-lab12$ gcc -Wall -pthread -o prodcons prodcons.c && ./prodcons
put 1 (in buffer: 1)
put 2 (in buffer: 2)
put 3 (in buffer: 3)
put 4 (in buffer: 4)
    take 1 (in buffer: 3)
put 5 (in buffer: 4)
    take 2 (in buffer: 3)
put 6 (in buffer: 4)
    take 3 (in buffer: 3)
put 7 (in buffer: 4)
    take 4 (in buffer: 3)
put 8 (in buffer: 4)
    take 5 (in buffer: 3)
    take 6 (in buffer: 2)
    take 7 (in buffer: 1)
    take 8 (in buffer: 0)
```

<p align="center"><em>Figure 1.1 – Producer and consumer</em></p>

The output is the same as the expected one: the producer fills the buffer to 4 and then waits, and after that it puts one item each time the slow consumer takes one. At the end the consumer empties the buffer (8 items put, 8 items taken).

### Observe

**1. The producer is fast, yet "in buffer" never goes above 4. Which call stops it?**
The call `sem_wait(&empty)`. The semaphore `empty` counts the free slots and starts at 4. Every `put` takes one unit. When the buffer is full the value is 0, and the producer sleeps in `sem_wait(&empty)` until the consumer takes an item and calls `sem_post(&empty)`.

**2. Who waits at the start: the consumer or the producer? Which call makes it wait?**
The consumer waits. It is created first and at once calls `sem_wait(&full)`. The semaphore `full` starts at 0 (nothing to take), so the consumer sleeps there until the producer puts the first item and calls `sem_post(&full)`. The producer does not wait at the start, because `empty` is 4.

**3. In `producer()`, swap the first two lines: lock the mutex BEFORE `sem_wait(&empty)`. Run it. What happens when the buffer is full, and why?**
I made a copy of the program with the two lines swapped and ran it with `timeout`, so the terminal comes back (the exit code 124 means that `timeout` had to kill the program):

```
vboxuser@Ubuntu:~/os-lab12$ sed '/sem_wait(&empty);/{N;s/\(.*\)\n\(.*\)/\2\n\1/}' prodcons.c > prodcons_swap.c && grep -n -A1 'pthread_mutex_lock(&lock);' prodcons_swap.c | head -4 && gcc -Wall -pthread -o prodcons_swap prodcons_swap.c && timeout 5 ./prodcons_swap; echo "exit code: $?"
17:        pthread_mutex_lock(&lock);
18-        sem_wait(&empty); /* wait for a free slot */
--
31:        pthread_mutex_lock(&lock);
put 1 (in buffer: 1)
put 2 (in buffer: 2)
put 3 (in buffer: 3)
put 4 (in buffer: 4)
exit code: 124
```

<p align="center"><em>Figure 1.2 – Mutex before sem_wait: the program hangs</em></p>

The `grep` shows that in `producer()` the line `pthread_mutex_lock(&lock)` (line 17) is now before `sem_wait(&empty)` (line 18). After four `put` lines nothing is printed any more. The producer takes the mutex, then calls `sem_wait(&empty)` for the fifth item, but the buffer is full, so it sleeps, and it sleeps while it holds the mutex. The consumer, to take an item and call `sem_post(&empty)`, must first take the same mutex, but the producer has it. Each one waits for the other, so this is a deadlock. In the right order (`sem_wait` first, mutex after) the producer waits for a place without holding the mutex, and the consumer can work. I put the lines back.


## Part 2. The condition variable

The problem of `order.c` from Lab 11 (B must read only after A has written), solved in another way. The state is a normal variable `ready`, protected by a mutex. The condition variable is only the place where B sleeps until A says "something changed". `pthread_cond_wait` releases the mutex, sleeps, and takes the mutex again before it returns.

```c
/* condvar.c - the order.c problem, solved with a mutex and a condition variable.
 * Build: gcc -Wall -pthread -o condvar condvar.c
 */
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
char message[32];
int ready = 0; /* the condition itself */
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t changed = PTHREAD_COND_INITIALIZER;
void *writer(void *arg) {
    sleep(1);
    pthread_mutex_lock(&lock);
    strcpy(message, "hello");
    ready = 1;
    printf("A: message written\n");
    pthread_cond_signal(&changed); /* wake one waiter */
    pthread_mutex_unlock(&lock);
    return NULL;
}
void *reader(void *arg) {
    pthread_mutex_lock(&lock);
    while (!ready) { /* while, not if: re-check after waking */
        printf("B: not ready, I sleep\n");
        pthread_cond_wait(&changed, &lock); /* unlocks, sleeps, locks again */
    }
    printf("B: I read \"%s\"\n", message);
    pthread_mutex_unlock(&lock);
    return NULL;
}
int main(void) {
    pthread_t a, b;
    pthread_create(&b, NULL, reader, NULL);
    pthread_create(&a, NULL, writer, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    return 0;
}
```

```
vboxuser@Ubuntu:~/os-lab12$ gcc -Wall -pthread -o condvar condvar.c && ./condvar
B: not ready, I sleep
A: message written
B: I read "hello"
```

<p align="center"><em>Figure 2.1 – condvar</em></p>

The output is the same as the expected one: B goes to sleep, one second later A writes the message and wakes B, and B reads "hello".

### Observe

**1. In `order.c` (Lab 11) the semaphore itself remembered that the message was ready. Here, which variable remembers it?**
The variable `ready`. The condition variable `changed` remembers nothing: it is only the place where B sleeps and the way for A to wake it. The fact "the message is ready" is stored in the normal variable `ready` (protected by the mutex).

**2. Replace `while (!ready)` with `if (!ready)`, compile, run. Does it still work here?**

```
vboxuser@Ubuntu:~/os-lab12$ sed 's/while (!ready)/if (!ready)/' condvar.c > condvar_if.c && grep -n 'if (!ready)' condvar_if.c && gcc -Wall -pthread -o condvar_if condvar_if.c && ./condvar_if
24:    if (!ready) { /* while, not if: re-check after waking */
B: not ready, I sleep
A: message written
B: I read "hello"
```

<p align="center"><em>Figure 2.2 – condvar with if instead of while</em></p>

Yes, it works and the output is the same. In this program there is one reader, one writer and exactly one `signal`, so after the wake-up `ready` is really 1. The `if` checks the condition only once, and here this is enough, but it is only luck of this small program.

**3. Then why do we write `while`? Give one reason why a thread can wake up and still find `ready == 0`.**
Because the wake-up does not guarantee that the condition is true. One reason is a spurious wakeup: `pthread_cond_wait` is allowed to return even if nobody called `signal`. Another reason is that with several waiters another thread can take the mutex first and use up (change back) the condition before the awakened thread gets the mutex again. With `while` the thread checks the condition again after every wake-up and goes back to sleep if it is false. With `if` it would go on with a wrong state.


## Part 3. Readers and writers

Many readers may read together, a writer must be alone. Two semaphores: `room` (1 = the room is free for a writer) and `rlock` (a lock around the counter `readers`). The first reader in locks the room, the last one out unlocks it.

```c
/* readers.c - readers and writers with two semaphores.
 * Many readers may read together; a writer must be alone.
 * Build: gcc -Wall -pthread -o readers readers.c
 */
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#define READERS 3
#define WRITERS 2
int value = 0; /* the shared data */
int readers = 0; /* how many are reading now */
sem_t rlock; /* protects the counter readers */
sem_t room; /* 1 = the room is free for a writer */
void *reader(void *arg) {
    long id = (long)arg;
    sem_wait(&rlock);
    if (++readers == 1) /* the first reader in locks out the writers */
        sem_wait(&room);
    sem_post(&rlock);
    printf("reader %ld reads %d (readers inside: %d)\n", id, value, readers);
    usleep(200000);
    sem_wait(&rlock);
    if (--readers == 0) /* the last reader out lets the writers in */
        sem_post(&room);
    sem_post(&rlock);
    return NULL;
}
void *writer(void *arg) {
    long id = (long)arg;
    sem_wait(&room); /* alone in the room */
    value += 10;
    printf("writer %ld writes %d (alone)\n", id, value);
    usleep(200000);
    sem_post(&room);
    return NULL;
}
int main(void) {
    sem_init(&rlock, 0, 1);
    sem_init(&room, 0, 1);
    pthread_t r[READERS], w[WRITERS];
    pthread_create(&w[0], NULL, writer, (void *)1L);
    for (long i = 0; i < READERS; i++)
        pthread_create(&r[i], NULL, reader, (void *)(i + 1));
    pthread_create(&w[1], NULL, writer, (void *)2L);
    for (int i = 0; i < READERS; i++) pthread_join(r[i], NULL);
    for (int i = 0; i < WRITERS; i++) pthread_join(w[i], NULL);
    return 0;
}
```

```
vboxuser@Ubuntu:~/os-lab12$ gcc -Wall -pthread -o readers readers.c && ./readers
writer 1 writes 10 (alone)
reader 1 reads 10 (readers inside: 1)
reader 2 reads 10 (readers inside: 2)
reader 3 reads 10 (readers inside: 3)
writer 2 writes 20 (alone)
```

<p align="center"><em>Figure 3.1 – Readers and writers</em></p>

The output is the same as the expected one: writer 1 writes alone, then the three readers are inside at the same time and all read the same value 10, and at the end writer 2 writes alone.

### Observe

**1. Three readers are inside at the same time, but a writer is always alone. Which two lines of `reader()` make that possible?**
The lines `if (++readers == 1) sem_wait(&room);` and `if (--readers == 0) sem_post(&room);`. The first reader that comes in takes the room (`sem_wait(&room)`), so no writer can enter, and the next readers just go in without touching `room`. The last reader that goes out gives the room back (`sem_post(&room)`). A writer calls `sem_wait(&room)`, so it waits until there is no reader inside, and while the writer is inside the room is taken and the readers wait.

**2. Why does the counter `readers` need its own lock (`rlock`)? What could go wrong without it?**
Because `++readers` and `--readers` are not one step: like `counter++` in Lab 11 they are load, change, store. Without `rlock` two readers could change the counter at the same time and lose one change, and the check `== 1` or `== 0` could be wrong. Then the first reader might not lock the room (a writer would come in together with the readers), or the last reader might not unlock it (the writers would wait forever).

**3. Imagine readers that keep arriving without a pause. What happens to writer 2?**
There is always at least one reader inside, so `readers` never goes down to 0, nobody calls `sem_post(&room)`, and writer 2 waits in `sem_wait(&room)` without end. This problem is called starvation: the writer is not blocked by a deadlock, it just never gets its turn.


## Part 4. Build a deadlock, then remove it

Two threads, two mutexes, taken in the opposite order. Each one holds what the other needs, and both wait forever. I run it with `timeout`, so the terminal comes back after 5 seconds. I saved this version as `deadlock_orig.c`.

```c
/* deadlock.c - two threads, two mutexes, taken in opposite order.
 * Build: gcc -Wall -pthread -o deadlock deadlock.c
 * Run: timeout 5 ./deadlock; echo "exit code: $?" (124 = killed by timeout)
 */
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
pthread_mutex_t A = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t B = PTHREAD_MUTEX_INITIALIZER;
void *one(void *arg) {
    pthread_mutex_lock(&A);
    printf("one: I have A, I want B\n");
    sleep(1);
    pthread_mutex_lock(&B); /* waits for two */
    printf("one: I have both\n");
    pthread_mutex_unlock(&B);
    pthread_mutex_unlock(&A);
    return NULL;
}
void *two(void *arg) {
    pthread_mutex_lock(&B); /* the fix: lock A first here too */
    printf("two: I have B, I want A\n");
    sleep(1);
    pthread_mutex_lock(&A); /* waits for one */
    printf("two: I have both\n");
    pthread_mutex_unlock(&A);
    pthread_mutex_unlock(&B);
    return NULL;
}
int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, one, NULL);
    pthread_create(&t2, NULL, two, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("done\n");
    return 0;
}
```

```
vboxuser@Ubuntu:~/os-lab12$ gcc -Wall -pthread -o deadlock_orig deadlock_orig.c && timeout 5 ./deadlock_orig; echo "exit code: $?"
one: I have A, I want B
two: I have B, I want A
exit code: 124
```

<p align="center"><em>Figure 4.1 – The deadlock</em></p>

The two lines "I have A, I want B" and "I have B, I want A" are printed, then nothing happens, and after 5 seconds `timeout` kills the program (exit code 124).

### The fix

In `two()` I lock A first and then B, the same order as in `one()`, and I swap the two `unlock` lines too. This is the complete file `deadlock.c` that I hand in:

```c
/* deadlock.c - two threads, two mutexes, taken in the SAME order (fixed version).
 * Build: gcc -Wall -pthread -o deadlock deadlock.c
 * Run: timeout 5 ./deadlock; echo "exit code: $?" (124 = killed by timeout)
 */
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
pthread_mutex_t A = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t B = PTHREAD_MUTEX_INITIALIZER;
void *one(void *arg) {
    pthread_mutex_lock(&A);
    printf("one: I have A, I want B\n");
    sleep(1);
    pthread_mutex_lock(&B);
    printf("one: I have both\n");
    pthread_mutex_unlock(&B);
    pthread_mutex_unlock(&A);
    return NULL;
}
void *two(void *arg) {
    pthread_mutex_lock(&A); /* the fix: lock A first here too */
    printf("two: I have B, I want A\n");
    sleep(1);
    pthread_mutex_lock(&B);
    printf("two: I have both\n");
    pthread_mutex_unlock(&B);
    pthread_mutex_unlock(&A);
    return NULL;
}
int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, one, NULL);
    pthread_create(&t2, NULL, two, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("done\n");
    return 0;
}
```

```
vboxuser@Ubuntu:~/os-lab12$ gcc -Wall -pthread -o deadlock deadlock.c && timeout 5 ./deadlock; echo "exit code: $?"
two: I have B, I want A
two: I have both
one: I have A, I want B
one: I have both
done
exit code: 0
```

<p align="center"><em>Figure 4.2 – The fixed version</em></p>

Now the program ends with `done` and the exit code 0. This time thread `two` was the first one to get A, so its lines are first; the order of the threads is not fixed, and it does not matter. The text "I have B, I want A" in the `printf` of `two()` is left from the original on purpose: it is the same text as in the expected output of the lab sheet. After the fix the thread really has A and wants B, so the message is not exact any more, but it is only a message and does not change how the locks work.

### Observe

**1. In the stuck run, which thread holds which mutex, and which one is each thread waiting for? Draw the circle.**
Thread `one` holds A and waits for B. Thread `two` holds B and waits for A.

```
   one ──holds──> A <──waits for── two
    │                               │
    └────waits for──> B <──holds────┘
```

Each thread waits for the mutex that the other one holds, so the circle is closed and nobody can go on.

**2. Remove both `sleep(1)` lines from the ORIGINAL version and run it ten times. Does it still always hang? Why is that worse, not better?**

```
vboxuser@Ubuntu:~/os-lab12$ sed '/sleep(1);/d' deadlock_orig.c > deadlock_nosleep.c && gcc -Wall -pthread -o deadlock_nosleep deadlock_nosleep.c && for i in $(seq 10); do timeout 2 ./deadlock_nosleep > /dev/null; echo "run $i: exit code $?"; done
run 1: exit code 0
run 2: exit code 0
run 3: exit code 0
run 4: exit code 0
run 5: exit code 0
run 6: exit code 0
run 7: exit code 0
run 8: exit code 0
run 9: exit code 0
run 10: exit code 0
```

<p align="center"><em>Figure 4.3 – The original without sleep, ten runs</em></p>

In my ten runs it did not hang even once. Without `sleep(1)` each thread takes both mutexes in a very short time, before the other thread has even started, so the threads do not meet. But the order of the locks is still wrong, so the deadlock is still possible, it only happens rarely and by chance, when the two threads reach the locks at almost the same moment. This is worse: with `sleep(1)` the error is easy to see, but without it the program can pass all the tests and hang later in real work, and then it is very hard to repeat and to find.

**3. In one sentence: why can taking the locks in the same order never deadlock?**
If every thread takes the locks in the same order (A before B), a thread that holds B has already got A, so nobody can hold B and wait for A, and the circle of waiting can never close.


## Conclusion

In this work I used the tools of Lab 11 together. In the producer and consumer, the semaphore `empty` stops the producer when the buffer is full, the semaphore `full` makes the consumer wait for an item, and the mutex protects the buffer itself; when I took the mutex before `sem_wait(&empty)`, the program hung after four items, because the producer slept while it held the mutex. The condition variable is another way to wait for an event: the fact is kept in the variable `ready`, and the condition must be checked with `while`, because a thread can wake up while the condition is false. In readers and writers two semaphores let many readers in together and keep the writer alone, but a stream of readers can starve the writer. At last I built a deadlock (two threads, two mutexes, opposite order, exit code 124) and removed it by taking the locks in the same order in both threads. I also saw that without `sleep(1)` the same bad code did not hang in ten runs, so this kind of error can stay hidden.
