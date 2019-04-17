# PlaidCTF - Plaid Planning Party III 2 (500pts, 8 solves)
*Writeup Author: hpmv*.

## Introduction

We played with HackingForSoju for the PlaidCTF. We got first blood on this challenge, which came out in the first few hours of the CTF but remained unsolved for more than a day.

This was a particularly interesting challenge that I spent a lot of time in, with a lot of interesting discoveries, so I decided to make a writeup to share the experience.

**If you just want to look at the correct solution, skip to Attempt 3.**

Also, I'll reduce clutter in the text part by putting all the code at the end.

## Reversing

I'll skip most of the details for the reversing part. It's not a particularly difficult binary to reverse, and the difficult part of the task isn't the reversing anyway.

We're given a binary that takes 16 command-line arguments, the first being either 1 or not 1, and the rest must be a permutation of the first 15 positive integers. They are subtracted by 1 and then assigned to 15 _persons_. Let's say for person $i$ the number assigned is $s_i$. I call it the _seed_.

Then for each person, we spawn a pthread to perform a sequence of tasks. Each task is one of the following:

* Lock mutex $R(s_i, k)$ where $s_i$ is the seed for person $i$ and $k$ is a hardcoded number, and then sleep for a unit of time.
* Unlock mutex $R(s_i, k)$, and then sleep for a unit of time.
* Sleep for a unit of time.
* Print a message.

Our goal is to find an input so that all threads will finish, and then it will use the input to compute a flag. The flag computation function appears chaotic, and in Plaid Planning Party III (part 1) the flag computation is not dependent on the input, so in this task we're very likely asked to find the right input instead of reversing the flag computation function.

Here is the lookup table for the $R$ function. The row is the seed given to a thread, and the column is the value of $k$. Note that for different $k$, the values are guaranteed to be different. We'll therefore refer to each column as a "mutex group". Notice also that some mutex groups have fewer mutexes, but there are 19 in total.


|        | 0   | 1   | 2   | 3   | 4   |
| ------ | --- | --- | --- | --- | --- |
| **0**  | 0   | 5   | 13  | 15  | 20  |
| **1**  | 0   | 5   | 10  | 15  | 20  |
| **2**  | 1   | 5   | 10  | 15  | 20  |
| **3**  | 1   | 5   | 11  | 15  | 20  |
| **4**  | 1   | 6   | 11  | 15  | 21  |
| **5**  | 2   | 6   | 11  | 15  | 21  |
| **6**  | 2   | 6   | 11  | 16  | 21  |
| **7**  | 2   | 6   | 12  | 16  | 21  |
| **8**  | 3   | 6   | 12  | 16  | 22  |
| **9**  | 3   | 7   | 12  | 16  | 22  |
| **10** | 3   | 7   | 12  | 16  | 23  |
| **11** | 4   | 7   | 12  | 16  | 23  |
| **12** | 4   | 7   | 13  | 16  | 24  |
| **13** | 4   | 7   | 13  | 15  | 24  |
| **14** | 0   | 5   | 13  | 15  | 24  |

Also, here is the sequence of operations performed by each thread, where $+x$ denotes lock, $-x$ denotes unlock, and $\cdots$ denotes sleep.

| #      | Operation Sequence                                                                                        |
| ------ | --------------------------------------------------------------------------------------------------------- |
| **0**  | $\cdots , +4 , +3 , -4 , +0 , -3 , -0 , +2 , +1 , -2 , -1 , +0 , +2 , -0 , +0 , -2 , -0$                  |
| **1**  | $\cdots , +0 , +4 , +2 , +1 , -0 , -1 , +3 , -2 , -3 , +1 , -1 , -4$                                      |
| **2**  | $\cdots , +2 , +1 , -2 , +2 , -2 , +0 , -0 , +4 , -4 , +3 , -3 , -1 , +4 , -4$                            |
| **3**  | $\cdots , +4 , +3 , -4 , +0 , -3 , -0 , +2 , +1 , -2 , +4 , -1 , -4$                                      |
| **4**  | $\cdots , +4 , -4 , +0 , -0 , +1 , +2 , +3 , -1 , -3 , -2 , +2 , +3 , -3 , -2 , +2 , -2$                  |
| **5**  | $\cdots , +4 , +3 , -4 , +0 , -0 , +2 , -3 , +1 , -1 , -2 , +0 , +2 , -0 , -2$                            |
| **6**  | $\cdots , +0 , +2 , -0 , +1 , -1 , -2 , +3 , +0 , -0 , +4 , -3 , -4$                                      |
| **7**  | $\cdots , +4 , -4 , +0 , -0 , +1 , +3 , +2 , -3 , -2 , -1 , +0 , -0$                                      |
| **8**  | $+0 , +2 , -2 , -0 , +0 , -0 , +4 , -4 , +2 , +1 , -1 , -2 , +0 , -0 , +3 , +4 , -4 , -3 , +0 , -0$       |
| **9**  | $\cdots , +3 , +1 , -1 , +4 , -4 , -3 , +0 , +2 , -0 , +1 , -2 , -1$                                      |
| **10** | $\cdots , +1 , +3 , +2 , -2 , -1 , +0 , -3 , -0 , +4 , -4$                                                |
| **11** | $\cdots , +4 , +1 , \cdots , +2 , -1 , -4 , +1 , \cdots , -1 , -2 , +0 , +1 , \cdots , +2 , -0 , -1 , -2$ |
| **12** | $\cdots , +0 , -0 , +0 , +1 , -0 , +2 , +3 , -2 , -3 , -1 , +0 , -0 , +1 , +3 , -1 , -3 , +4 , -4$        |
| **13** | $\cdots , +0 , +2 , -0 , +1 , -1 , -2 , +0 , +4 , -0 , -4$                                                |
| **14** | $+3 , +1 , +2 , -3 , -2 , -1 , +1 , +2 , -2 , +2 , -2 , -1$                                               |



## Attempt 1: Turn-Based Deadlock-Free Constraint Solving

It turns out that knowing *what* to do was perhaps the most difficult part of the challenge, at least for me.

We're clearly asked to find an input that does not lead to deadlocks, but the exact semantics of that is unclear. We first thought that the goal was to find a way to guarantee no deadlocks, but the intuition of deadlock-free code is to order your locks, yet there are individual threads that grab locks in inconsistent order even within themselves.

So I thought that the problem has something to do with the sleeps. Whenever a thread performs an action other than printing, it will sleep for 1 unit of time. Some threads actually sleep for 2 units of time at the beginning. I did not check exactly how long 1 unit is, but it is a constant number and is quite long compared to mutex overhead. What this sleeping effectively does is to turn the game into a turn-based one: At every turn, each thread will perform its next thing, and be blocked (and forced to skip the turn) if it is waiting on a lock someone else is holding. Our goal is to find a way to assign the locks to the threads so that they don't end up in a deadlock situation.

This seems like a pretty messy problem to solve analytically, so I turned to constraint solving, with z3. For that, I need to formalize the problem as variables and constraints (this was inspired by an article about [solving nonograms with z3](https://dzone.com/articles/solving-nonograms-with-ruby-and-z3).)

* **Variables - mutex ownership**: we'll use a two-dimensional grid whose values $g_{m, x}$ denote the owner of mutex $m$ at time $x$. The owner is the thread number if the mutex is held, or -1 if not. The time axis will have some reasonable length, like 80. There are 19 different mutexes. Note that this representation inherently implies that a mutex can only be held by one owner at once.
* **Variables - seed assignment**: for thread $0\le i<15$ the seed is $s_i$.
* **Variables - lock assignment**: for thread $0\le i<15$ and lock number $0\le k<5$, the actual lock ID assigned to it is $r_{i, k}$.
* **Variables - thread actions**: for each thread $i$ and lock number $k$, there is a list of *regions* during which the thread holds $k$. For region #$j$, variable $t_{i,k,j}$ denotes the turn number when the thread obtained the lock, and $e_{i,k,j}$ denotes the turn number when the thread released the lock. Also, we'll use $c_{i,l}$ to denote the time of the $l$-th action (except printing) performed by $i$ (these variables alias $t_{i,k,j}$ and $e_{i,k,j}$ unless it is a sleep).
* **Constraint - seeds**: $0 <= s_i < 15$ for all $i$, and all $s_i$ are distinct.
* **Constraint - lock assignment**: $r_{i, k} = R(s_i, k)$, represented as $\bigvee_{a=0}^{14} r_{i,k}=R(a, k) \wedge s_i=a$.
* **Constraint - mutex ownership**: for each thread $i$ and mutex $m$, $g_{m, x} = i \iff \bigvee_{j} t_{i,k,j} \le x < e_{i,k,j} \wedge m=r_{i,k}$, where $k$ is the mutex group of $m$. This essentially says that the mutex is owned by a thread if and only if the time is within a lock-unlock region of that thread and that the mutex is assigned to that thread.
* **Constraint - tight execution**: threads will not sit there doing nothing; whenever it is free it will always perform the next action. There are two parts to this:
  * Unlock and sleep will always happen immediately. If action $c_{i,j}$ is followed by an action $c_{i, j+1}$ which is not a locking operation, then $c_{i,j+1}=c_{i,j}+1$.
  * Locking only blocks if there is another owner. If action $c_{i,j}$ is followed by an action $c_{i, j+1}$ which is a locking operation on mutex group $k$, then for every turn $x$ such that $c_{i,j} < x < c_{i, j+1}$ and mutex $m$ in the group, $r_{i,k} = m \implies g_{m,x} \ne -1$.
* **Constraint - ricky must not have cheese**: $r_{6, 1} \ne 6$. This is just some arbitrary constraint imposed in the code; it will abort if this isn't satisfied.

We solve these constraints to get a schedule of execution and corresponding inputs $s_i$. We get a solution within a few seconds, here's an example:

```
  0   1   2   3   4   5   6   7  10  11  12  13  15  16  20  21  22  23  24
  .   .   .   8   .   .   .   .   .   .   .   .  14   .   .   .   .   .   .
  .   .  12   8  13   .  14  10   .   .   8   2  14   .  11   4   0   .   5
  .   .   .   8  13  11  14  10   .  14   .   2  14   0  11   .   0   .   5
  .   .   4   .  13  11  14  10   .  14   .   2   9   0  11   .   .   .   5
  .   .  12   0  13  11  14  10  11   .   .   2   9   0  11   .   .   .   5
  .   .  12   0  13   2   4  10  11   .   .   2   9  10  11   .   .   .   5
  .   .  12   8  13   2   4  10  11   4  10  13   9  10   7   .   .   .   5
  .   .  12   .   6   2   4  10  11   4   0  13   9  10   3   .   .   .   5
  7   .  12   .   6   2   4  13  11   4   0  13   9  10   3   .   .   8   5
  .   .  12  10   6   2   4   .  11   4   0  13   9  10   3   .   .   .   5
  .   .  12  10   6   2   4   .  11   4   0   2   9   4   3   .   .   .   5
  .   .  12   .   6   2  14   .  11   4   0   .   9   4   3   .   .   .   5
  2   .  12   .   6   2  14   .  11   4   0   .   9   .   3   .  10   .   5
  .   .  12   .   6   2  14   .  11  14   0   .   9   .   3   .   .   .   5
  .   .  12   .   6   2  14   .  11   4   0   .   9   .   3   .   .   .   5
  .   .  12   .   6   2  14   .  11   4   0   .   9   4   3   .   .   .   5
  .   .  12   .   6   2  14   .  11   4   0   .   9   .   3   .   .   .   5
  .   .  12   .   6   2  14   .  11  14   0   .   9   .   3   .   .   .   5
  .   .  12   .   6   2  14   .  11   4   0   .   9   .   3   .   .   .   5
  .   .  12   .   6   2   0   .  11   .   0   .   9   .   3   .   .   .   5
  .   .  12   .   6   2   0   .  11   .   8   .   9   .   3   .   .   .   5
  .   .  12   .   6   2  12   8  11   .   8   .   9   .   3   .   .   .   5
  .   .   .   0   6   2  12   .  11   .   8   .   9   .   3   .   .   .   5
  .   .   .   0   6   2  12   .  11   .   0   .   9   .   3   .   .   .   5
  .   .   .   8   6   2  12   .  11   .   0   .   9   .   3   .   .   .   5
  .   .   .   0   6   2  12   .  11   .   0   .   9   .   3   .   .   .   5
  .   .   .   0   6   2  12   .  11   .   6   .   9   8   3   .   .   .   5
  .   .   .   .  13   2  12   .  11   .   6   .   9   8   3   .   .   8   5
  .   .   .   .  13   2  12   6  11   .   6   .   9   8   3   .   .   .   5
  .   .   .   .  13   2  12   .  11   .   6   .   9   .   3   .   .   .   5
  .   .   .   8  13   2  12   .  11   .  12   .   9   .   3   .   .   .   5
  .   .   .   .  13   2  12   .  11   .  12   .   9  12   3   .   .   .   5
  .   .   .   .  13   2  12   .  11   .   .   .   9  12   3   .   .   .   5
  .   .   .   .  13   2  12   .  11   .   .   .   9   6   3   .   .   .   5
  .   .   .   .  13   2   9   .  11   .   .   .   9   6   3   .   .   .   5
  .   .  12   .  13   2   .   .  11   .   .   .   9   6   3   .   .   .   5
  .   .   .   .  13   2   .   .  11   .   .   .   9   6   3   9   .   .   5
  .   .   .   .  13   2  12   .  11   .   .   .   9   6   3   .   .   .   5
  .   .   .   .  13   2  12   .  11   .   .   .   3   6   3   .   .   .   5
  .   9   .   .  13   2  12   .  11   .   .   .   3   6   2   .   .   .   5
  .   9   .   .  13   2  12   .  11   9   .   .   3   6   .   .   .   .   5
  .   3   .   .  13   2  12   .  11   9   .   .   3   6   .   .   .   .   5
  .   3   .   .  13   2  12   .  11   9   .   .   5   6   .   .   .   .   5
  .   .   .   .  13   2  12   .  11   9   .   .   5   6   .   .   .   .  13
  5   .   .   .   6   2  12   .  11   9   .   .   5   6   .   .   .   .  13
  .   .   .   .   1   2  12   .  11   9   .   .   5   6   .   .   .   .   .
  .   .   .   .   1   2  12   .  11   9   .   5   5   6   .   .   .   6   1
  .   .   .   .   1   2  12   .  11   9   .   5   2  12   .   .   .   6   1
  .   .   .   .   1   2   9   .  11   9   .   5   .  12   .   .   .   .   1
  .   .   .   .   1  11   9   .  11   3   .   5   .   .   .   .   .   .   1
  .   .   .   .   1  11   .   .  11   3   .   5   .   .   2  12   .   .   1
  .   .   .   .   1   7   .   .  11   3   .   5   .   .   .   .   .   .   1
  .   .   .   .   1   7   .   .   .   3   .   5   7   .   .   .   .   .   1
  .  11   .   .   1   7   .   .   7   3   .   5   7   .   .   .   .   .   1
  .  11   .   .   1   7   .   .   7   3   .   5   .   .   .   .   .   .   1
  .  11   .   .   1   7   .   .   .   3   .   5   .   .   .   .   .   .   1
  .  11   .   .   1  11   .   .   .   3   .   5   .   .   .   .   .   .   1
  7  11   .   .   1  11   .   .   .   3   .   5   .   .   .   .   .   .   1
  .  11   .   .   1  11   .   .  11   3   .   5   .   .   .   .   .   .   1
  .   .   .   .   1  11   .   .  11   3   .   5   .   .   .   .   .   .   1
  .   .   .   .   1   5   .   .  11   3   .   5   .   .   .   .   .   .   1
  .   .   .   .   1   3   .   .   .   3   .   5   .   .   .   .   .   .   1
  .   .   .   .   1   3   .   .   .   .   .   1   .   .   .   .   .   .   1
  5   .   .   .   1   3   .   1   .   .   .   1   .   .   3   .   .   .   1
  5   .   .   .   .   .   .   1   .   .   .   1   .   .   3   .   .   .   1
  5   .   .   .   .   .   .   .   .   .   .   1   .   .   .   .   .   .   1
  5   .   .   .   .   .   .   .   .   .   .   1   1   .   .   .   .   .   1
  5   .   .   .   .   .   .   .   .   .   .   5   1   .   .   .   .   .   1
  .   .   .   .   .   .   .   .   .   .   .   5   .   .   .   .   .   .   1
  .   .   .   .   .   .   .   1   .   .   .   .   .   .   .   .   .   .   1
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   1
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
  .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .   .
9 14 1 4 7 15 12 2 11 5 10 3 8 13 6
```

If we plug this into the program, like `./ppp2 1 9 14 1 4 7 15 12 2 11 5 10 3 8 13 6`, it will actually hang.

I set up a gdb session with some `dprint`s to figure out what's going on. Essentially, the problem is that our constraint solver only finds *one* possible outcome of the execution; when two threads try to lock the mutex in the same turn, it's not predictable who will end up grabbing it first.

## Attempt 1.5: Various Additional Mutex Constraints

I realized the problem is actually much harder at this point. We can't just find one interleaving that works, we have to make sure it will always work. So I thought, *maybe the deal is to avoid ambiguous situations* when we may have two threads competing for the same lock at the same time. So I added a new constraint:

* **Constraint - No ambiguity**: Two threads will not start locking on the same mutex in the same turn. Formally, for two threads $i < j$, if action $c_{i, l}$ is followed by action $c_{i, l+1}$ that is a locking operation on mutex group $k$, and action $c_{j, l}$ is followed by action $c_{j, l+1}$ that is also a locking operation on mutex group $k$, then $r_{i, k} = r_{j, k} \implies c_{i, l} \ne c_{j, l}$.

Note that, the interval between $c_{i, l}$ (exclusive) and $c_{i, l+1}$ (exclusive) when $c_{i,l+1}$ is a locking operation is the time the thread spends waiting for the lock. We'll call such an interval a **waiting region** on mutex group $k$ for later discussion. This interval is empty if the mutex is acquired immediately.

Sadly, with this additional constraint, we get no solutions. In fact, we can eyeball that conclusion by noticing that in the second turn, there are 6 threads (0, 3, 4, 5, 7, 11) locking on mutex group $k=4$, but there are only 5 mutexes in that group. A race will always happen.

I didn't give up here, though, because when I debugged many sessions, I noticed that during the first round, the mutexes always get acquired by the thread with the lower thread number. Perhaps that's because these threads get created in that order and the kernel has some predictable scheduling going on. Also, with even further debugging, I conjectured that the mutexes were fair, i.e. if threads $A$ and $B$ lock the mutex in that order, then they will get it in that order. Just to be clear to the readers, both of these end up being *wrong* about pthread and pthread mutexes, but these were what I had assumed at the time. So the new constraints are:

* **Constraint - No ambiguity V2**: For two waiting regions $(c_{i, l}, c_{i, l+1})$ and $(c_{j, l}, c_{j, l+1})$ of threads $i< j$ on the same mutex group $k$, $r_{i, k} = r_{j, k} \implies \textrm{if } c_{i,l} = c_{j, l} = 0 \textrm{ then } c_{i, l+1} < c_{j, l+1} \textrm{ else }  c_{i, l} \ne c_{j, l}$.
* **Constraint - Mutex fairness**: For two waiting regions $(c_{i, l}, c_{i, l+1})$ and $(c_{j, l}, c_{j, l+1})$ of threads $i\ne j$ on the same mutex group $k$, $r_{i,k}=r_{j,k} \implies c_{i,l}<c_{j, l} \implies c_{i,l+1}<c_{j,l+1}$. (If they start waiting in that order, they acquire in that order.)

Debugging solutions from these constraints, I realized that the mutex is actually not fair. So then, it isn't sufficient to say that two threads do not wait on the same mutex at the same time; we need a stronger constraint that when a mutex is released, no two threads can be already waiting on the mutex, because then it's ambiguous which one will get the mutex. In other words, the waiting regions for the same mutex never overlap:

* **Constraint - No ambiguity V3**: For two waiting regions $(c_{i, l}, c_{i, l+1})$ and $(c_{j, l}, c_{j, l+1})$ of threads $i< j$ on the same mutex group $k$, $r_{i, k} = r_{j, k} \implies \textrm{if } c_{i,l} = c_{j, l} = 0 \textrm{ then } c_{i, l+1} < c_{j, l+1}$ $\textrm{ else } (c_{i, l} \ne c_{j, l} \wedge (c_{i,l} \ge c_{j, l+1} \vee c_{j,l} \ge c_{i, l+1}))$.

This gives solutions such as `3 6 12 4 14 15 10 1 9 7 13 2 11 8 5` that will most likely succeed in printing a flag, but the flag is garbage.

## Attempt 2: Turn-Based Deadlock-Free Guarantee

At this point I reached out to LarsH from HackingForSoju to bounce some ideas. A couple of things he suggested are to maybe solve for deadlock-free guarantee using things such as partial order of locks, and to also write a simulator so we can all play with it.

It still seemed to me at this point that it was important that the threads took actions in steps by sleeping, since it was clear that a partial ordering is not possible due to inconsistent ordering even between the same thread. (In this end this turned out to be the wrong insight on my part.)

So now the problem I'm trying to tackle is, rather than avoiding ambiguity, whenever two threads wait on the same lock, we will consider both outcomes and make sure both of them will not deadlock.

So I wrote a simulator to check this. By the way, at this time the "official" simulator coded in the binary ("Checking the dinner...", which executes whenever the first argument is not 1) was not functional yet, so I never bothered to understand what it tried to do. If I did, I think I would've abandoned the turn-based idea more quickly.

The simulator simulates turns, each turn is decomposed into two parts, *during* and *after*:

* During the turn: each thread which is not *suspended* performs its next action. If the action is to unlock, it clears the owner of the mutex. If it's to sleep, it does nothing. If it is to lock, it adds itself to the *waiting queue* of that mutex and marks itself as *suspended*.
* After the turn: for each mutex without an *owner*, for each waiter in its waiting queue, fork off to a new branch in the search tree where we remove the waiter from the queue and make it the owner of the mutex. This is recursive so we exponentially enumerate all possibilities.

A search state is successful if all threads are complete. It is a deadlock if all threads are either complete or suspended and there is at least one suspended thread.

We carry through this search to explore all possible branches; if we find no deadlock in any state, then we're good.

This simulator works pretty quickly and I was able to find deadlocks easily with the solutions I got before with z3. But the problem is, how do we find an answer that passes the simulator? z3 is sure to not work here, because we need to explore an exponential number of branches, not just 1.

After playing around a few ideas, I eventually decided to run a genetic algorithm. I start with an initial input (like 2 3 4 5 6 7 1 8 9 10 11 12 13 14), and then mutate it:

* Swap pairs: take all pairs of positions and swap them.
* Rotate triplets: randomly take three positions and rotate them by 1 step.
* Rotate quadruplets: randomly take four positions and rotate them by 1 step.
* Randomize: pick a new permutation completely randomly.

The fitness metric for the genetic algorithm is how many branches are successful as opposed to arriving at a deadlock. A problem here is that for some inputs, there are hundreds of thousands of branches so it takes a long time to search through all of them; so instead, I sample them with a poor-man's version of Monte Carlo Tree Search to estimate the success ratio.

After many iterations on the code I eventually found a good solution that got 100% success in the sampled branches, so I turned off the sampling and ran another round of mutations to really find an input that would avoid deadlock in all branches. Here's one input: `14 7 9 12 8 11 10 2 4 6 15 1 3 5 13`. This input has 134334 possible successful outcomes, and never deadlocks.

Sadly, that still didn't print the right flag, and I could never get this input to actually produce a deadlock, so my simulator was probably not wrong. Still, the problem is I could find quite a few deadlock-free solutions, and all of them print garbage flags. That doesn't feel right.

## Attempt 3: Deadlock-Free without Turns

At this point I was convinced that I needed even more constraints, but the only part I could still put a constraint on is to ignore the sleeps; in other words, I need to guarantee that the threads will never deadlock even if I remove the nanosleeps and consider all interleavings. 

In fact, this is now a pure algorithm problem, and quite a nice one. After the CTF I spent some time to rephrase it to something more intuitive, so here it goes:

### The Dinner Problem

Problem Statement: At a dinner, there are 15 diners, and 19 plates. The plates are divided (unevenly) into 5 categories (think carbohydrates, vegetables, meats, etc.; I don't know how Indian food works ;) )

There are 15 menus, each naming 5 plates, one of each category. Each diner will get one menu, and the plates listed on the menu will be the only plates they will eat during the dinner.

Each diner has a specific eating habit, which is a linear sequence of actions. Each action is either "grab my plate for category X" or "put back my plate for category X". The diner will follow this sequence, but if the current action is to grab a plate and that plate is currently in someone else's possession, then they will sit there and wait until the plate is put back.

We're given the contents of each menu and the eating habit of each diner, and we need to find a way to assign the menus to diners so that everyone is guaranteed to finish their meal.

### Deadlock-Free Condition

The first thing I wanted to find out is, given a number of diners, how to determine whether they will ever deadlock. I tried to find some research literature around this topic, but all I could find is either runtime deadlock detection (which covers only one execution path) or static lock ordering enforcement. Indeed, the simple way of avoiding deadlocks is to have a partial order of the locks and never grab them out of order, but as I mentioned earlier, even the same diner grabs plates in inconsistent orders. This problem is a little different.

Actually, grabbing locks out of order is *not* a guarantee for race condition. Suppose we have two threads:

* Thread 1: Lock C; Lock A; Lock B; Unlock all
* Thread 2: Lock C; Lock B; Lock A; Unlock all

These threads grab A and B in the opposite order, but they *don't* have a deadlock, because the mutex C prevents them from ever getting into a deadlock situation. This is what I missed before which kept me looking for a turn-based solution - we *don't* need a partial order!

In fact, the following is a necessary condition for a deadlock to be possible:

* Let $L(i, a)$ be the set of mutexes that thread $i$ would already be holding when it is about to execute action #$a$.
* Let $l_{i, a}$ be the mutex that thread $i$ would be trying to acquire as part of action #$a$ (which must be a locking action).
* Necessary condition for deadlock existence: There exists a sequence of threads $i_1, i_2, ..., i_n$ and for each thread $i_j$ an index $a_j$ into its action sequence, such that
  *  The held locks, $L(i_j, a_j)$, are pairwise disjoint; and
  *  For each thread $i_j$, it is waiting on a lock held by $i_{j+1}$ (or $i_1$ if $j=n$), i.e. $l_{i_j, a_j} \in L(i_{j+1}, a_{j+1})$.

We call the pair of such sequences $(\{i_j\}, \{a_j\})$ a **deadlock point**.

Proving this necessary condition is trivial: if a deadlock exists, at the time of a deadlock, we can start from one thread and follow its wait chain to arrive at a cycle of threads, which form a deadlock point.

Is this a *sufficient* condition? In other words, if the program is deadlock free, are we guaranteed there are no deadlock points?

I never managed to prove this part, but I opted for speed and gave it a try anyway. I wrote a brute-force search which tries to find a wait cycle:

* First, precompute $L(i, a)$ for each thread $i$ and action index $a$.
* Precompute $S(l)$ for each mutex $l$, defined as the set of tuples $(i, x, y)$ such that thread $i$ holds mutex $l$ from action index $x$ (inclusive) to action index $y$ (exclusive).
* Initialize the search state: a sequence $A$ of pairs $(i, a)$ representing thread $i$ waiting on action $a$, initialized to empty, and a set of all held locks $H$, initialized to empty.
* $\textrm{function } \textrm{FindDeadlock}(l, A, H):$
  * For each $(i, x, y) \in S(l)$:
    * If there exists $(i, a)$ in $A$:
      * If $x\le a < y$, we have found a deadlock; return $A$.
      * Otherwise thread $i$ is already used, skip this case.
    * For each $x \le a < y$ such that action $a$ of thread $i$ is a locking operation:
      * Append $(i, a)$ to $A$
      * Add the set of held locks $L(i, a)$ to $H$
      * $\textrm{FindDeadlock}(l_{i, a}, A, H)$
      * Remove $L(i, a)$ to $H$
      * Remove $(i, a)$ from $A$

On the input I obtained earlier, `14 7 9 12 8 11 10 2 4 6 15 1 3 5 13`, this code found a deadlock instantly, and I verified it by hand.

### Searching for the Final Answer

Now that we can find deadlocks, it's time to find an assignment of menus that avoids deadlocks. I did struggle on this bit, but in the end I realized it isn't all that difficult. If we look at the menus again, they have pretty similar entries - here are the top 4 rows again:

|       | 0   | 1   | 2   | 3   | 4   |
| ----- | --- | --- | --- | --- | --- |
| **0** | 0   | 5   | 13  | 15  | 20  |
| **1** | 0   | 5   | 10  | 15  | 20  |
| **2** | 1   | 5   | 10  | 15  | 20  |
| **3** | 1   | 5   | 11  | 15  | 20  |
| **4** | 1   | 6   | 11  | 15  | 21  |

If we try to assign diners to menus (as opposed to menus to diners), then it's likely there aren't that many choices among the first four menus, because they share a lot of mutexes (for groups 1, 3, 4, in particular). We can check for deadlocks even when we have just 2 threads, because if we can get a deadlock with just a few threads, we can get the same deadlock with more threads. (This seems obvious, but it is important, because earlier when we were looking at the turn-based version, a deadlock with fewer threads does *not* necessarily imply a deadlock with more threads; the other threads could've delayed one of the deadlocking threads to avoid the deadlock.)

The algorithm is then to brute-force all possible assignments of diners to menus, stopping at each step to check for deadlocks. I'll omit the algorithm description here as it's quite standard, like the N-Queens problem. It turns out that the deadlock check prunes out a lot of the search tree and we only end up with 1.3 million nodes, which, even with Python, finished within a few minutes.

The search yielded four answers, one of which failed the "ricky must not have cheese" rule; the remaining three were:

```
8 14 4 11 5 12 10 2 1 9 3 13 6 15 7
8 14 4 12 5 11 10 2 1 9 3 13 6 15 7
12 14 4 8 5 11 10 2 1 9 3 13 6 15 7
```

The last one printed the correct flag :)

## Epilogue

After submitting the flag we saw that the binary was patched to fix the dinner checker. I'm not sure if that would've made it easier; I was generally confused by that code anyway, and didn't spend much time on it as a result. Apparently, the intention of that code was to make the goal clearer, which unfortunately I had missed.

However, the challenge was a great exercise for using z3 and a good refresher on algorithms.  I also found the collaboration with HackingForSoju helpful, particularly in finding new alternative interpretations of the challenge.

I still haven't proven my claim on the deadlock-free condition, though. If anyone knows of a proof or has relevant literature to share, please let me know (hpmv on freenode IRC).

## Code

I only wrote some comments, moved blocks of code around, and fixed some completely broken code after the CTF, so most of the code is still a bit messy, but here it is, the script that contains everything.

```python
import random
import itertools
import sys
from z3 import *

total_time = 80
total_threads = 15
total_locks = 19

# What each thread does. positive = lock (off by 1),
# negative = unlock (off by 1), 0 = sleep.
schedules = [
    [0, 5, 4, -5, 1, -4, -1, 3, 2, -3, -2, 1, 3, -1, 1, -3, -1],
    [0, 1, 5, 3, 2, -1, -2, 4, -3, -4, 2, -2, -5],
    [0, 3, 2, -3, 3, -3, 1, -1, 5, -5, 4, -4, -2, 5, -5],
    [0, 5, 4, -5, 1, -4, -1, 3, 2, -3, 5, -2, -5],
    [0, 5, -5, 1, -1, 2, 3, 4, -2, -4, -3, 3, 4, -4, -3, 3, -3],
    [0, 5, 4, -5, 1, -1, 3, -4, 2, -2, -3, 1, 3, -1, -3],
    [0, 1, 3, -1, 2, -2, -3, 4, 1, -1, 5, -4, -5],
    [0, 5, -5, 1, -1, 2, 4, 3, -4, -3, -2, 1, -1],
    [1, 3, -3, -1, 1, -1, 5, -5, 3, 2, -2, -3, 1, -1, 4, 5, -5, -4, 1, -1],
    [0, 4, 2, -2, 5, -5, -4, 1, 3, -1, 2, -3, -2],
    [0, 2, 4, 3, -3, -2, 1, -4, -1, 5, -5],
    [0, 5, 2, 0, 3, -2, -5, 2, 0, -2, -3, 1, 2, 0, 3, -1, -2, -3],
    [0, 1, -1, 1, 2, -1, 3, 4, -3, -4, -2, 1, -1, 2, 4, -2, -4, 5, -5],
    [0, 1, 3, -1, 2, -2, -3, 1, 5, -1, -5],
    [4, 2, 3, -4, -3, -2, 2, 3, -3, 3, -3, -2],
]

# The following is used for implementing the R function.
mealr = [0, 3, 6, 9, 12, 1, 6, 11, -1, -1, 1, 4,
         8, 14, -1, 2, 8, -1, -1, -1, 1, 5, 9, 10, 13]


def rbox(a, b):
    x = 2*(a-b+15) % 30
    y = (2*(b-a+15)+1) % 30
    return min(x, y)


condensed_l = [0, 1, 2, 3, 4, 5, 6, 7, 10,
               11, 12, 13, 15, 16, 20, 21, 22, 23, 24]

possible_locks = [[0, 1, 2, 3, 4], [5, 6, 7], [
    8, 9, 10, 11], [12, 13], [14, 15, 16, 17, 18]]

# The R function.


def pick_meal(pr, t):
    best = t*5
    for i in range(t*5 + 1, t*5 + 5):
        if mealr[i] >= 0:
            if rbox(pr, mealr[i]) < rbox(pr, mealr[best]):
                best = i
    return best


# The lookup table for precomputed R function outputs.
pick_table = [[condensed_l.index(pick_meal(pr, t))
               for t in range(5)] for pr in range(15)]

'''
pick_table:
      0  1  2  3  4
  0   0  5 13 15 20
  1   0  5 10 15 20
  2   1  5 10 15 20
  3   1  5 11 15 20
  4   1  6 11 15 21
  5   2  6 11 15 21
  6   2  6 11 16 21
  7   2  6 12 16 21
  8   3  6 12 16 22
  9   3  7 12 16 22
 10   3  7 12 16 23
 11   4  7 12 16 23
 12   4  7 13 16 24
 13   4  7 13 15 24
 14   0  5 13 15 24
'''

###############################################################################
######## ATTEMPT 1. z3 solver for turn-based problem ##########################
###############################################################################


def pick_table_emu(s, lock_index, r):
    """Writes a constraint that says R(s, lock_index) = r."""
    res = None
    for i in range(total_threads):
        cond = And(s == i, r == pick_table[i][lock_index])
        if res is None:
            res = cond
        else:
            res = Or(res, cond)
    return res


def solve_with_z3():
    """Attempt 1, z3 solver."""
    G = [[Int('G(%s,%s)' % (L, X)) for X in range(total_time)]
         for L in range(total_locks)]

    solver = Solver()

    for L in range(total_locks):
        for X in range(total_time):
            solver.add(G[L][X] >= -1, G[L][X] < total_threads)

    S = [Int('S(%s)' % T) for T in range(total_threads)]

    for v in S:
        solver.add(v >= 0, v < total_threads)

    solver.add(Distinct(S))

    all_groups = []
    all_waitings = []
    all_actual_l = []

    for T in range(total_threads):
        schedule = schedules[T]
        groups = [[] for i in range(5)]
        all_groups.append(groups)
        waitings = [[] for i in range(5)]
        all_waitings.append(waitings)
        pending = [None for i in range(5)]
        prev_var = -1
        sleeps = 0
        for event in schedule:
            if event == 0:
                event_var = Int('Sleep(%s)#%s' % (T, sleeps))
                sleeps += 1
                solver.add(event_var >= 0, event_var < total_time)
                solver.add(event_var == prev_var + 1)
                prev_var = event_var
                continue
            lock = abs(event) - 1
            is_lock = event > 0
            event_var = Int('%s(%s, %s)#%s' % (
                'Lock' if is_lock else 'Unlock', T, lock, len(groups[lock])))
            solver.add(event_var >= 0, event_var < total_time)
            if not is_lock:
                solver.add(event_var == prev_var + 1)
            solver.add(event_var > prev_var)
            if is_lock:
                assert pending[lock] is None, (T, lock)
                pending[lock] = event_var
                waitings[lock].append((prev_var + 1, event_var))
            else:
                assert pending[lock] is not None, (T, lock)
                groups[lock].append((pending[lock], event_var))
                pending[lock] = None
            prev_var = event_var
        actual_ls = []
        all_actual_l.append(actual_ls)
        for lock in range(5):
            assert pending[lock] is None, (T, lock)
            ActualL = Int('ActualLock(%s,%s)' % (T, lock))
            actual_ls.append(ActualL)
            solver.add(pick_table_emu(S[T], lock, ActualL))
            for X in range(total_time):
                is_in_range = False
                for (lock_time, unlock_time) in groups[lock]:
                    in_range = And(X >= lock_time, X < unlock_time)
                    if is_in_range is False:
                        is_in_range = in_range
                    else:
                        is_in_range = Or(is_in_range, in_range)
                for L in possible_locks[lock]:
                    cond = (G[L][X] == T) == And(is_in_range, ActualL == L)
                    solver.add(cond)

                is_waiting = False
                for (prev, lock_time) in waitings[lock]:
                    waiting = And(X >= prev, X < lock_time)
                    if is_waiting is False:
                        is_waiting = waiting
                    else:
                        is_waiting = Or(is_waiting, waiting)
                for L in possible_locks[lock]:
                    cond = Or(Not(And(ActualL == L, is_waiting)),
                              G[L][X] != -1)
                    solver.add(cond)

    solver.add(Or(S[6] < 4, S[6] > 8))

    if False:
        difficulty = 0
        for t_a in range(total_threads):
            for t_b in range(t_a):
                for L in range(5):
                    for l_a in range(len(all_groups[t_a][L])):
                        for l_b in range(len(all_groups[t_b][L])):
                            (wait_a, _) = all_waitings[t_a][L][l_a]
                            (wait_b, _) = all_waitings[t_b][L][l_b]
                            actual_lock_a = all_actual_l[t_a][L]
                            actual_lock_b = all_actual_l[t_b][L]
                            (lock_a, _) = all_groups[t_a][L][l_a]
                            (lock_b, _) = all_groups[t_b][L][l_b]
                            prio_a, prio_b = t_a, t_b
                            # a bunch of mess when trying different things here.
                            #
                            # cond = If(actual_lock_a == actual_lock_b,
                            #           If(And(And(wait_a == wait_b), wait_b < 2),
                            #              lock_a > lock_b if prio_a > prio_b else lock_b > lock_a,
                            #              # True,
                            #              And(wait_a != wait_b,
                            #                  If(wait_a < wait_b, lock_a < lock_b, lock_b < lock_a))
                            #              ), True)
                            # cond = If(actual_lock_a == actual_lock_b,
                            #           And(wait_a != wait_b,
                            #               If(wait_a < wait_b, lock_a < lock_b, lock_b < lock_a)), True)
                            diff = If(And(actual_lock_a == actual_lock_b,
                                          wait_a == wait_b), 1, 0)
                            difficulty = diff + difficulty
                            # print cond
        solver.add(difficulty < 10)

    print 'Solving...'
    while solver.check() == sat:
        model = solver.model()

        for j in range(total_locks):
            print '%3s' % condensed_l[j],
        print

        for i in range(total_time):
            for j in range(total_locks):
                if model[G[j][i]] == -1:
                    print '  .',
                else:
                    print '%3s' % model[G[j][i]],
            print

        for i in range(total_threads):
            print int(str(model[S[i]])) + 1,
        print
        print
        sol = [int(str(model[S[i]])) for i in range(total_threads)]

        # blacklist that solution to find a new one next.
        solver.add(Not(And(*[S[i] == sol[i] for i in range(total_threads)])))


if len(sys.argv) == 2 and sys.argv[1] == '1':
    solve_with_z3()
    quit()


###############################################################################
######## ATTEMPT 2. Genetic algorithm to find deadlock-free inputs ############
###############################################################################

def choices_helper(variations, i, prefix):
    """Given [(1, [100, 101]), (2, [200, 201])], returns combinations:
    [[(1, 100), (2, 200)], [(1, 101), (2, 200)], ...]"""
    if i >= len(variations):
        yield prefix
        return
    (index, possibles) = variations[i]
    for item in possibles:
        for res in choices_helper(variations, i + 1, prefix + [(index, item)]):
            yield res


def choices(variations):
    return choices_helper(variations, 0, [])


def remove_waiter(q, waiter):
    return [x for x in q if x != waiter]


def sim_end_of_turn(seeds, pos, locks):
    # Simulates the end of turn semantics. For each lock without owner, we'll
    # try all possibilities for promoting a waiter to the owner.
    #
    # pos: for each thread, (index of action the thread will take next,
    #                        whether thread is blocked on lock)
    # locks: for each lock, (owner, list of waiters)
    # returns a generator of states of (pos, locks)
    variations = []

    for i in range(total_locks):
        (owner, waiters) = locks[i]
        if owner is None and len(waiters) > 0:
            variations.append((i, waiters))
    all_variations = list(choices(variations))
    random.shuffle(all_variations)
    for variation in all_variations:
        pos_copy = pos[:]
        locks_copy = locks[:]
        for (lock_index, waiter) in variation:
            locks_copy[lock_index] = (waiter, remove_waiter(
                locks_copy[lock_index][1], waiter))
            (task_index, blocked) = pos_copy[waiter]
            pos_copy[waiter] = (task_index, False)
        yield (pos_copy, locks_copy)


def simulate_during_turn(seeds, pos, locks):
    """Simulates all threads' actions during a turn."""
    pos_copy = pos[:]
    new_waiters = [[] for l in range(total_locks)]
    locks_copy = locks[:]
    for (thread, (action, blocked)) in enumerate(pos):
        if not blocked and action < len(schedules[thread]):
            event = schedules[thread][action]
            if event == 0:
                pos_copy[thread] = (action + 1, False)
            else:
                lock_k = abs(event) - 1
                actual_lock = pick_table[seeds[thread]][lock_k]
                is_lock = event > 0
                if is_lock:
                    new_waiters[actual_lock].append(thread)
                    pos_copy[thread] = (action + 1, True)
                else:
                    assert locks[actual_lock][0] == thread, (
                        thread, actual_lock, locks[actual_lock])
                    locks_copy[actual_lock] = (
                        None, locks_copy[actual_lock][1])
                    pos_copy[thread] = (action + 1, False)

    for (lock_index, new) in enumerate(new_waiters):
        if new:
            (owner, waiters) = locks_copy[lock_index]
            locks_copy[lock_index] = (owner, waiters + new)
    return (pos_copy, locks_copy)


def simulate_one_turn(seeds, pos, locks):
    """Simulates one turn completely, returns generator of (A, H)."""
    (pos, locks) = simulate_during_turn(seeds, pos, locks)
    return sim_end_of_turn(seeds, pos, locks)


def is_complete(pos):
    return all(index == len(schedules[i]) and not blocked
               for (i, (index, blocked)) in enumerate(pos))


def is_deadlock(pos):
    return not any(index != len(schedules[i]) and not blocked
                   for (i, (index, blocked)) in enumerate(pos))


class DeadlockException(Exception):
    """Actually, this is used to just break out of the search."""
    pass


def simulate_from(seeds, pos, locks, results, limit, kill_if_dead, report_back):
    """DFS from an initial state to simulate all possible outcomes onward.

    Stops if limit number of outcomes are reached.

    If report_back is true, will return the immediate children instead of 
    recursively visiting them.

    If kill_if_dead, will return immediately when finding any deadlock."""
    if is_complete(pos):
        results[0] += 1
        if results[0] + results[1] > limit:
            raise DeadlockException()
        return []
    if is_deadlock(pos):
        # print 'Deadlock'
        results[1] += 1
        if kill_if_dead or results[0] + results[1] > limit:
            raise DeadlockException()
        return []
    next_states = []
    for (new_pos, new_locks) in simulate_one_turn(seeds, pos, locks):
        if report_back:
            next_states.append((new_pos, new_locks))
        else:
            simulate_from(seeds, new_pos, new_locks, results,
                          limit, kill_if_dead, False)
    return next_states


def simulate(seeds, limitless=False):
    """Simulates with a poor-man's Monte Carlo Tree Search.

    Basically, we first BFS to expand to 25 nodes, and then DFS from each of 
    these nodes with a max-nodes limit. Also, every time we expand a node of
    the search tree, we visit its children randomly. So this gives a pretty
    good approximation to a real Monte Carlo search.

    limitless is a hack that will make the search exhaustively check all
    nodes, essentially disabling sampling."""
    good = 0
    bad = 0
    complete = True
    pool = [([(0, False)] * total_threads,
             [(None, []) for i in range(total_locks)])]
    while len(pool) > 0 and len(pool) < 25:
        new_pool = []
        for (pos, locks) in pool:
            results = [0, 0]
            new_pool.extend(simulate_from(
                seeds, pos, locks, results, 1000000, False, True))
            good += results[0]
            bad += results[1]
        pool = new_pool
    for (pos, locks) in pool:
        results = [0, 0]
        try:
            simulate_from(seeds, pos, locks, results,
                          10000000 if limitless else 25, limitless, False)
        except DeadlockException as e:
            complete = False
        good += results[0]
        bad += results[1]
    return (good, bad, complete)


# print simulate([1, 13, 3, 9, 11, 10, 6, 7, 5, 4, 0, 14, 12, 2, 8], limitless=True)

#simulate([2, 13, 1, 4, 6, 10, 9, 3, 8, 12, 5, 11, 0, 14, 7])
#simulate([2, 5, 11, 3, 13, 14, 9, 0, 8, 6, 12, 1, 10, 7, 4])
#simulate([7, 8, 13, 6, 11, 5, 0, 14, 2, 12, 4, 1, 10, 3, 9])

#master_pool = [[7, 8, 13, 6, 11, 5, 0, 14, 2, 12, 4, 1, 10, 3, 9]]
#master_pool = [[7, 6, 13, 8, 11, 5, 0, 14, 2, 12, 4, 1, 10, 3, 9]]
#master_pool = [[2, 5, 11, 3, 13, 14, 9, 0, 8, 6, 12, 1, 10, 7, 4]]
master_pool = [[1, 2, 3, 4, 5, 6, 0, 7, 8, 9, 10, 11, 12, 13, 14]]
#master_pool = [[7, 6, 12, 14, 9, 5, 0, 13, 2, 8, 1, 3, 10, 4, 11]]
#master_pool = [[1, 13, 3, 9, 11, 10, 2, 7, 6, 4, 0, 14, 12, 5, 8]]
#master_pool = [[13, 6, 8, 11, 7, 10, 9, 1, 0, 5, 14, 3, 2, 4, 12]]
new_pool = []

if len(sys.argv) == 2 and sys.argv[1] == '2':
    while True:
        for master in master_pool:
            for (i, j) in itertools.combinations(range(15), 2):
                sol = master[:]
                (sol[i], sol[j]) = (sol[j], sol[i])
                if sol[6] >= 4 and sol[6] <= 8:
                    continue
                (good, bad, complete) = simulate(sol)
                new_pool.append(
                    (tuple(sol), good*1.0/(good+bad), good, bad, complete))
            for i in [0]:
                sol = master[:]
                (good, bad, complete) = simulate(sol)
                new_pool.append(
                    (tuple(sol), good*1.0/(good+bad), good, bad, complete))
            for _ in range(50):
                while True:
                    sol = master[:]
                    i = random.randint(0, 14)
                    j = random.randint(0, 14)
                    k = random.randint(0, 14)
                    if i == j or j == k or k == i:
                        continue
                    if sol[6] >= 4 and sol[6] <= 8:
                        continue
                    (sol[i], sol[j], sol[k]) = (sol[j], sol[k], sol[i])
                    (good, bad, complete) = simulate(sol)
                    new_pool.append(
                        (tuple(sol), good*1.0/(good+bad), good, bad, complete))
                    break
            for _ in range(50):
                while True:
                    sol = master[:]
                    i = random.randint(0, 14)
                    j = random.randint(0, 14)
                    k = random.randint(0, 14)
                    l = random.randint(0, 14)
                    if len(set([i, j, k, l])) < 4:
                        continue
                    if sol[6] >= 4 and sol[6] <= 8:
                        continue
                    (sol[i], sol[j], sol[k], sol[l]) = (
                        sol[j], sol[k], sol[l], sol[i])
                    (good, bad, complete) = simulate(sol)
                    new_pool.append(
                        (tuple(sol), good*1.0/(good+bad), good, bad, complete))
                    break
            for _ in range(50):
                sol = range(15)
                random.shuffle(sol)
                if sol[6] >= 4 and sol[6] <= 8:
                    continue
                (sol[i], sol[j], sol[k]) = (sol[j], sol[k], sol[i])
                (good, bad, complete) = simulate(sol)
                new_pool.append(
                    (tuple(sol), good*1.0/(good+bad), good, bad, complete))

        new_pool = sorted(set(new_pool), key=lambda x: (-x[1], x[2]))[:10]

        print '====================================='
        for (sol, ratio, good, bad, complete) in new_pool:
            print 'Good: %s, Bad: %s, Complete: %s, Ratio: %s, Sol: %s' % (
                good, bad, complete, good * 1.0 / (good + bad), sol)
        master_pool = [list(item[0]) for item in new_pool]
    quit()


###############################################################################
######## ATTEMPT 3. Solving for true deadlock-free inputs #####################
###############################################################################

class Lock:
    def __init__(self, id):
        self.id = id

    def __str__(self):
        return '+' + str(self.id)

    __repr__ = __str__


class Unlock:
    def __init__(self, id):
        self.id = id

    def __str__(self):
        return '-' + str(self.id)

    __repr__ = __str__


class FoundDeadlockException(Exception):
    pass


def find_deadlock(seeds):
    """Initial version of deadlock finding code which takes input seeds and
    prints a deadlock situation if we find one."""
    actual_schedules = []
    lock_regions = [[] for i in range(total_locks)]
    held_locks_at = []

    for (thread, schedule) in enumerate(schedules):
        seed = seeds[thread]
        actual_schedule = []
        for action in schedule:
            if action == 0:
                continue
            lock_index = abs(action) - 1
            actual_lock = pick_table[seed][lock_index]
            is_lock = action > 0
            if is_lock:
                actual_schedule.append(Lock(actual_lock))
            else:
                actual_schedule.append(Unlock(actual_lock))
        actual_schedules.append(actual_schedule)

    for (thread, schedule) in enumerate(actual_schedules):
        lock_time = [-1] * total_locks
        locks_held = set()
        held_locks_at_one = []
        for (index, action) in enumerate(schedule):
            held_locks_at_one.append(set(locks_held))
            if isinstance(action, Lock):
                assert lock_time[action.id] == -1
                lock_time[action.id] = index
                locks_held.add(action.id)
            else:
                assert lock_time[action.id] != -1
                if index > lock_time[action.id] + 1:
                    lock_regions[action.id].append(
                        (thread, lock_time[action.id] + 1, index))
                lock_time[action.id] = -1
                locks_held.remove(action.id)
        held_locks_at.append(held_locks_at_one)

    for (i, s) in enumerate(actual_schedules):
        print i, s
    thread_pos = [-1] * total_threads
    locks_held = set()
    found = set()

    def helper(lock, indent=0):
        for (thread, start, end) in lock_regions[lock]:
            if thread_pos[thread] != -1:
                if thread_pos[thread] >= start and thread_pos[thread] < end:
                    print 'Found deadlock'
                    found.add(tuple(thread_pos))
                    raise FoundDeadlockException()
                continue
            for i in range(start, end):
                action = actual_schedules[thread][i]
                if not isinstance(action, Lock):
                    continue
                held = held_locks_at[thread][i]
                if len(locks_held.intersection(held)) == 0:
                    # print '  '*(indent + 1) + "Position", i
                    locks_held.update(held)
                    thread_pos[thread] = i
                    helper(action.id, indent + 2)
                    thread_pos[thread] = -1
                    for h in held:
                        locks_held.remove(h)
        # print '  '*indent + 'Done trying lock', lock

    try:
        for lock_id in range(total_locks):
            helper(lock_id)
    except FoundDeadlockException as e:
        print thread_pos, locks_held


#find_deadlock([11, 13, 3, 7, 4, 10, 9, 1, 0, 8, 2, 12, 5, 14, 6])
# find_deadlock(range(15))
# quit()


def is_deadlock_free(actual_schedules):
    """Given a list of schedules of Lock and Unlock actions,
    determine if it's impossible to get into a deadlock situation.

    Returns False if we found a deadlock."""
    lock_regions = [[] for i in range(total_locks)]
    held_locks_at = []
    num_threads = len(actual_schedules)

    for (thread, schedule) in enumerate(actual_schedules):
        lock_time = [-1] * total_locks
        locks_held = set()
        held_locks_at_one = []
        for (index, action) in enumerate(schedule):
            held_locks_at_one.append(set(locks_held))
            if isinstance(action, Lock):
                assert lock_time[action.id] == -1
                lock_time[action.id] = index
                locks_held.add(action.id)
            else:
                assert lock_time[action.id] != -1
                if index > lock_time[action.id] + 1:
                    lock_regions[action.id].append(
                        (thread, lock_time[action.id] + 1, index))
                lock_time[action.id] = -1
                locks_held.remove(action.id)
        held_locks_at.append(held_locks_at_one)

    thread_pos = [-1] * num_threads
    locks_held = set()
    found = set()

    def helper(lock, indent=0):
        for (thread, start, end) in lock_regions[lock]:
            if thread_pos[thread] != -1:
                if thread_pos[thread] >= start and thread_pos[thread] < end:
                    raise FoundDeadlockException()
                continue
            for i in range(start, end):
                action = actual_schedules[thread][i]
                if not isinstance(action, Lock):
                    continue
                held = held_locks_at[thread][i]
                if len(locks_held.intersection(held)) == 0:
                    # print '  '*(indent + 1) + "Position", i
                    locks_held.update(held)
                    thread_pos[thread] = i
                    helper(action.id, indent + 2)
                    thread_pos[thread] = -1
                    for h in held:
                        locks_held.remove(h)

    try:
        for lock_id in range(total_locks):
            helper(lock_id)
    except FoundDeadlockException as e:
        return False
    return True

# print is_deadlock_free([
#     [Lock(1), Lock(2), Unlock(1), Unlock(2)],
#     [Lock(2), Lock(1), Unlock(1), Unlock(2)],
# ])


def get_actual_schedule(assi):
    """Given a list of (seed, thread) assignments, return the list of
    schedules of actions."""
    actual = []
    for (seed, t) in enumerate(assi):
        schedule = schedules[t]
        actual_schedule = []
        for action in schedule:
            if action == 0:
                continue
            lock_index = abs(action) - 1
            actual_lock = pick_table[seed][lock_index]
            is_lock = action > 0
            if is_lock:
                actual_schedule.append(Lock(actual_lock))
            else:
                actual_schedule.append(Unlock(actual_lock))
        actual.append(actual_schedule)
    return actual


mycount = 0


def find_assignment(arr, i):
    """Main brute-force search with pruning. This gives the final answer."""
    global mycount
    if i == 15:
        print arr
        return
    for j in range(15):
        if j in arr:
            continue
        arr[i] = j
        actual = get_actual_schedule(arr[:i+1])
        mycount += 1
        if mycount % 1000 == 0:
            print mycount, 'Trying', arr[:i+1]
        if is_deadlock_free(actual):
            find_assignment(arr, i+1)
        arr[i] = -1


find_assignment([-1]*15, 0)
```
