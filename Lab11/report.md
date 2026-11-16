# Lab 11: The Race and the Semaphore

**MINISTRY OF EDUCATION, CULTURE AND RESEARCH**  
**OF THE REPUBLIC OF MOLDOVA**  

**Technical University of Moldova**  
**Faculty of Computers, Informatics and Microelectronics**  
**Department of Software Engineering and Automation**  


**MAXIM ORMANJI FAF-242**

Checked by: Reițman Petricia

Chișinău, 2026

## Purpose of the work

The purpose of this work is to see what happens when two threads touch the same variable at the same time, and then to fix it. First I watched a race with my own eyes (a counter that loses additions), then I fixed it with a mutex and with a semaphore that starts at 1. After that I used the same semaphore in two more ways: as a signal (it starts at 0) and as a counter (the parking lot).


## Before you start

I work on my own Linux laptop (Ubuntu, 2 CPU cores). I created a new folder for the lab. Every program is compiled with `-pthread` and without `-O2`, because the optimizer would hide the race of Part 1.

```
vboxuser@Ubuntu:~$ mkdir -p ~/os-lab11 && cd ~/os-lab11
vboxuser@Ubuntu:~/os-lab11$ nproc
2
```


## Part 1. See the race

Two threads each add 1 to the same counter, one million times. The right answer is 2 000 000. There is no protection at all.

```c
/* race.c - two threads add to the same counter, with no protection.
 * Build: gcc -Wall -pthread -o race race.c (no -O2: keep counter++ in memory)
 * Run: ./race (several times)
 */
#include <stdio.h>
#include <pthread.h>
#define N 1000000
long counter = 0; /* shared by both threads */
void *add(void *arg) {
    for (int i = 0; i < N; i++)
        counter++; /* read, add 1, write back: three steps */
    return NULL;
}
int main(void) {
    pthread_t a, b;
    pthread_create(&a, NULL, add, NULL);
    pthread_create(&b, NULL, add, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    printf("counter = %ld (expected %d)\n", counter, 2 * N);
    return 0;
}
```

I compiled it and ran it three times:

```
vboxuser@Ubuntu:~/os-lab11$ gcc -Wall -pthread -o race race.c && for i in 1 2 3; do ./race; done
counter = 2000000 (expected 2000000)
counter = 1000000 (expected 2000000)
counter = 2000000 (expected 2000000)
```

<p align="center"><em>Figure 1.1 – Three runs of race</em></p>

Two of the three runs gave the right number by luck: the first thread finished its million additions before the second one really started, so they did not meet. The sheet says "never 2 000 000", but on my machine the threads sometimes do not overlap, so I ran the program 20 times and counted the different results:

```
vboxuser@Ubuntu:~/os-lab11$ for i in $(seq 20); do ./race; done | sort | uniq -c
      1 counter = 1000000 (expected 2000000)
      1 counter = 1000148 (expected 2000000)
      1 counter = 1000480 (expected 2000000)
      1 counter = 1007291 (expected 2000000)
      1 counter = 1007694 (expected 2000000)
      1 counter = 1007917 (expected 2000000)
      1 counter = 1008190 (expected 2000000)
      1 counter = 1016217 (expected 2000000)
      1 counter = 1021523 (expected 2000000)
      1 counter = 1028057 (expected 2000000)
      1 counter = 1030083 (expected 2000000)
      1 counter = 1037545 (expected 2000000)
      1 counter = 1040211 (expected 2000000)
      1 counter = 1041265 (expected 2000000)
      1 counter = 1063828 (expected 2000000)
      1 counter = 1075888 (expected 2000000)
      1 counter = 1151209 (expected 2000000)
      1 counter = 1575681 (expected 2000000)
      1 counter = 1764439 (expected 2000000)
      1 counter = 1909101 (expected 2000000)
```

<p align="center"><em>Figure 1.2 – 20 runs of race: no run gave 2 000 000, and almost all numbers are different</em></p>

In these 20 runs the result was never 2 000 000, and it changed from 1 000 000 to 1 909 101: a lot of additions were lost.

### Observe

**1. `counter++` looks like one step. Write the three steps the CPU really does.**
(1) Load the value of `counter` from memory into a register. (2) Add 1 to the register. (3) Store the register back into `counter` in memory.

**2. Describe, with those steps, how two threads can lose one addition.**
Let `counter` be 5. Thread A loads 5 into its register. Before A stores anything, thread B also loads 5. A adds 1 and stores 6. B adds 1 to its own old 5 and also stores 6. Two additions were made, but the counter went only from 5 to 6, so one addition is lost. The same happens in the real program many times.

**3. Why is the result different at every run?**
Because the two threads run on different cores, and the exact moments when one thread reads and the other one writes depend on the scheduler and on the timing of the CPU, which is a little different every time. So the number of lost additions is different in each run (from about 90 000 to about 1 000 000 in my 20 runs). In two runs of the first three the threads did not overlap at all, so no addition was lost and the answer was exactly 2 000 000.


## Part 2. Fix it: mutex, then semaphore

### 2.1 A mutex around counter++

The line between `lock` and `unlock` is the critical section: only one thread may be inside.

```c
/* mutex.c - the same counter, now protected by a mutex.
 * Build: gcc -Wall -pthread -o mutex mutex.c
 */
#include <stdio.h>
#include <pthread.h>
#define N 1000000
long counter = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
void *add(void *arg) {
    for (int i = 0; i < N; i++) {
        pthread_mutex_lock(&lock); /* only one thread past this line */
        counter++; /* the critical section */
        pthread_mutex_unlock(&lock); /* let the other one in */
    }
    return NULL;
}
int main(void) {
    pthread_t a, b;
    pthread_create(&a, NULL, add, NULL);
    pthread_create(&b, NULL, add, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    printf("counter = %ld (expected %d)\n", counter, 2 * N);
    return 0;
}
```

### 2.2 The same lock, made from a semaphore

A semaphore is a number that never goes below 0. `sem_wait` takes one unit (and waits if there is none), `sem_post` gives one back. Started at 1, it behaves like a mutex.

```c
/* sem.c - the same counter, protected by a semaphore that starts at 1.
 * Build: gcc -Wall -pthread -o sem sem.c
 */
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#define N 1000000
long counter = 0;
sem_t s;
void *add(void *arg) {
    for (int i = 0; i < N; i++) {
        sem_wait(&s); /* 1 -> 0: I go in; at 0: wait */
        counter++;
        sem_post(&s); /* 0 -> 1: the next one may go in */
    }
    return NULL;
}
int main(void) {
    sem_init(&s, 0, 1); /* 0 = shared between threads, 1 = start value */
    pthread_t a, b;
    pthread_create(&a, NULL, add, NULL);
    pthread_create(&b, NULL, add, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    printf("counter = %ld (expected %d)\n", counter, 2 * N);
    sem_destroy(&s);
    return 0;
}
```

I compiled both and ran each of them three times:

```
vboxuser@Ubuntu:~/os-lab11$ gcc -Wall -pthread -o mutex mutex.c && gcc -Wall -pthread -o sem sem.c && for p in mutex sem; do for i in 1 2 3; do echo "run $i of $p:"; ./$p; done; done
run 1 of mutex:
counter = 2000000 (expected 2000000)
run 2 of mutex:
counter = 2000000 (expected 2000000)
run 3 of mutex:
counter = 2000000 (expected 2000000)
run 1 of sem:
counter = 2000000 (expected 2000000)
run 2 of sem:
counter = 2000000 (expected 2000000)
run 3 of sem:
counter = 2000000 (expected 2000000)
```

<p align="center"><em>Figure 2.1 – mutex and sem, three runs each</em></p>

### 2.3 The price of the lock

```
vboxuser@Ubuntu:~/os-lab11$ for p in race mutex sem; do echo "== time of $p"; ( time ./$p ) 2>&1 | grep -E 'counter|real'; done
== time of race
counter = 1839922 (expected 2000000)
real	0m0.020s
== time of mutex
counter = 2000000 (expected 2000000)
real	0m0.096s
== time of sem
counter = 2000000 (expected 2000000)
real	0m0.173s
```

<p align="center"><em>Figure 2.2 – Time of race, mutex and sem</em></p>

`race` is the fastest (0.020 s) but its answer is wrong, `mutex` takes 0.096 s and `sem` 0.173 s. The order is the same as on the reference machine (race, then mutex, then sem), only the numbers are different.

### Observe

**1. Run `mutex` and `sem` three times each. Is the answer always 2 000 000?**
Yes. All six runs (three of `mutex` and three of `sem`) gave exactly `counter = 2000000`. Only one thread at a time is inside the critical section, so no addition is lost.

**2. What would happen in `sem.c` with `sem_init(&s, 0, 2)`? Try it and explain.**
I made a copy of the program with the start value 2 and ran it five times:

```
vboxuser@Ubuntu:~/os-lab11$ sed 's/sem_init(&s, 0, 1)/sem_init(\&s, 0, 2)/' sem.c > sem2.c && gcc -Wall -pthread -o sem2 sem2.c && for i in 1 2 3 4 5; do ./sem2; done
counter = 2000000 (expected 2000000)
counter = 1836428 (expected 2000000)
counter = 1873839 (expected 2000000)
counter = 1885232 (expected 2000000)
counter = 1902567 (expected 2000000)
```

<p align="center"><em>Figure 2.3 – sem with the start value 2</em></p>

The race comes back: four of the five runs lost additions. A semaphore that starts at 2 lets two threads pass `sem_wait` at the same time (2 to 1 and 1 to 0), so both are inside the "critical section" together and the protection is gone. The first run gave 2 000 000 only by luck, as in `race`. To work as a lock a semaphore must start at 1.

**3. Why is the locked version slower?**
Every addition now needs a `lock` and an `unlock` (or `sem_wait` and `sem_post`): two million pairs of calls instead of two million plain `counter++`. When one thread is inside, the other one must wait, so the two threads can no longer work in parallel on the counter. In my measurement `sem` (0.173 s) is also slower than `mutex` (0.096 s): the semaphore is a more general tool than the mutex. The price is small, and it buys the right answer.


## Part 3. A semaphore as a signal

Now the problem is not two threads at once, but the order: B must read the message only after A has written it. A semaphore that starts at 0 is a closed door: B waits at `sem_wait` until A calls `sem_post`.

```c
/* order.c - thread B must wait until thread A has written the message.
 * A semaphore that starts at 0 is a signal: sem_wait blocks until someone posts.
 * Build: gcc -Wall -pthread -o order order.c
 */
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
char message[32];
sem_t ready; /* starts at 0: nothing is ready yet */
void *writer(void *arg) {
    sleep(1); /* pretend the work takes a while */
    strcpy(message, "hello");
    printf("A: message written\n");
    sem_post(&ready); /* 0 -> 1: wake up B */
    return NULL;
}
void *reader(void *arg) {
    printf("B: waiting for the message...\n");
    sem_wait(&ready); /* blocks while the value is 0 */
    printf("B: I read \"%s\"\n", message);
    return NULL;
}
int main(void) {
    sem_init(&ready, 0, 0);
    pthread_t a, b;
    pthread_create(&b, NULL, reader, NULL); /* B starts first, on purpose */
    pthread_create(&a, NULL, writer, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    sem_destroy(&ready);
    return 0;
}
```

```
vboxuser@Ubuntu:~/os-lab11$ gcc -Wall -pthread -o order order.c && ./order
B: waiting for the message...
A: message written
B: I read "hello"
```

<p align="center"><em>Figure 3.1 – order with the semaphore at 0</em></p>

B printed its first line at once, then there was a pause of one second, and then A wrote the message and B read it. The output is the same as the expected one.

### Observe

**1. Change the start value to 1 (`sem_init(&ready, 0, 1)`), compile, run. What does B read now, and why?**

```
vboxuser@Ubuntu:~/os-lab11$ sed 's/sem_init(&ready, 0, 0)/sem_init(\&ready, 0, 1)/' order.c > order1.c && gcc -Wall -pthread -o order1 order1.c && ./order1
B: waiting for the message...
B: I read ""
A: message written
```

<p align="center"><em>Figure 3.2 – order with the semaphore at 1</em></p>

B reads an empty string. With the start value 1, the door is already open: `sem_wait` in B takes the unit at once (1 to 0) and does not wait. So B prints `message` before A has written anything, and the global array `message` is still full of zeros, which is an empty string. A writes "hello" only one second later, too late for B.

**2. Put it back to 0. In one sentence: what does a semaphore that starts at 0 mean?**
A semaphore that starts at 0 means "nothing is ready yet": whoever calls `sem_wait` must sleep until another thread says "it is ready" with `sem_post`, so it works as a signal.


## Part 4. A semaphore as a counter: the parking lot

Six cars, three places. The semaphore starts at 3 and counts the free places. The mutex is there only so that the variable `inside` and the `printf` stay together.

```c
/* parking.c - 6 cars, 3 places. A semaphore that starts at 3 counts the free places.
 * Build: gcc -Wall -pthread -o parking parking.c
 */
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#define PLACES 3
#define CARS 6
sem_t places; /* free places */
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int inside = 0; /* only for printing */
void *car(void *arg) {
    long id = (long)arg;
    sem_wait(&places); /* take a place, or wait */
    pthread_mutex_lock(&lock);
    inside++;
    printf("car %ld enters (inside: %d)\n", id, inside);
    pthread_mutex_unlock(&lock);
    sleep(1); /* parked */
    pthread_mutex_lock(&lock);
    inside--;
    printf("car %ld leaves (inside: %d)\n", id, inside);
    pthread_mutex_unlock(&lock);
    sem_post(&places); /* give the place back */
    return NULL;
}
int main(void) {
    sem_init(&places, 0, PLACES);
    pthread_t t[CARS];
    for (long i = 0; i < CARS; i++)
        pthread_create(&t[i], NULL, car, (void *)(i + 1));
    for (int i = 0; i < CARS; i++)
        pthread_join(t[i], NULL);
    sem_destroy(&places);
    return 0;
}
```

```
vboxuser@Ubuntu:~/os-lab11$ gcc -Wall -pthread -o parking parking.c && ./parking
car 2 enters (inside: 1)
car 1 enters (inside: 2)
car 4 enters (inside: 3)
car 2 leaves (inside: 2)
car 5 enters (inside: 3)
car 1 leaves (inside: 2)
car 6 enters (inside: 3)
car 4 leaves (inside: 2)
car 3 enters (inside: 3)
car 5 leaves (inside: 2)
car 6 leaves (inside: 1)
car 3 leaves (inside: 0)
```

<p align="center"><em>Figure 4.1 – The parking lot with 3 places</em></p>

The order of the cars is different from the one in the sheet (the first car in is number 2), but this is allowed: the order depends on the scheduler. There are six entries and six exits, and at the end `inside` is 0.

### Observe

**1. What is the largest value of "inside" you see? Which line of code guarantees it?**
The largest value is **3**. It is guaranteed by `sem_wait(&places);` together with `sem_init(&places, 0, PLACES);` (`PLACES` is 3): the semaphore starts at 3, every car takes one unit before it goes in, and when the value is 0 the next car sleeps in `sem_wait` until some car leaves and calls `sem_post(&places)`. The mutex does not limit the number of cars, it only keeps `inside++` and the `printf` together.

**2. Set `PLACES` to 1. What does the semaphore become?**

```
vboxuser@Ubuntu:~/os-lab11$ sed 's/#define PLACES 3/#define PLACES 1/' parking.c > parking1.c && gcc -Wall -pthread -o parking1 parking1.c && ./parking1
car 1 enters (inside: 1)
car 1 leaves (inside: 0)
car 4 enters (inside: 1)
car 4 leaves (inside: 0)
car 6 enters (inside: 1)
car 6 leaves (inside: 0)
car 2 enters (inside: 1)
car 2 leaves (inside: 0)
car 3 enters (inside: 1)
car 3 leaves (inside: 0)
car 5 enters (inside: 1)
car 5 leaves (inside: 0)
```

<p align="center"><em>Figure 4.2 – The parking lot with 1 place</em></p>

With one place the semaphore becomes a lock (a mutex): `inside` is only 0 or 1, and every car goes in only after the previous one has left. This is the same thing as in Part 2, where the semaphore started at 1.


## Conclusion

In this work I saw a race with my own eyes: two threads that add to the same counter without protection lose additions, because `counter++` is three steps (load, add, store) and the threads can mix them. In 20 runs the result was never 2 000 000 and changed from 1 000 000 to 1 909 101. A mutex and a semaphore started at 1 fixed it (always 2 000 000), and the price was time: 0.096 s and 0.173 s against 0.020 s of the wrong version. With the start value 2 the semaphore stopped being a lock and the race came back. Then I used the semaphore in two other ways. Started at 0, it is a signal: B waits until A posts, and with the start value 1 B read an empty message. Started at 3, it is a counter of free places: the parking never had more than 3 cars, and with one place it became a lock again. So one tool, the semaphore, can be a lock, a signal and a counter, depending only on its start value.
