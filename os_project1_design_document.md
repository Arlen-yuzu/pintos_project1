# Design Review for Project 1: Threads 

## Group Members

- 杨寒梅 <11611712@mail.sustc.edu.cn>

## Task 1: Efficient Alarm Clock

### Data structures and functions

in `thread.h`:

* add `int64_t sleeping_ticks` into struct `thread` in order to store ticks that this thread can wake up

in `thread.c`:

* add `static struct list sleeping_list` that keeps track of the threads which are sleeping, note that this is an ascending order list which is sorted by `sleeping_ticks`
* add `void sleeping_list_handle(void)` to check `sleeping_list` from its head to see if any threads can be unblocked and removed from the list
* modify `void thread_tick (void)`, add function `sleeping_list_handle()`  in it so as to execute this function each timer tick

in `timer.c`:

- modify `void timer_sleep (int64_t ticks)`, instead of using a while loop busy waiting, I'll disable interrupts, block the thread and add it into the `sleeping_list` according to the rank of `sleeping_ticks`. 

### Algorithms 

In function `timer_sleep(int64_t ticks)`:

1. Check whether parameter `ticks` is larger than 0
2. Set the thread’s `sleeping_ticks` as the sum of the given ticks and the current ticks 
3. Disable interrupts 
4. Insert the current thread into the `sleeping_list` according to its `sleeping_ticks` to make sure the `sleeping_ticks` is in an ascending order.
5. Block the thread
6. Enable interrupts and reset to the old one

In function `sleeping_list_handle(void)`:

1. Check the `sleeping_list`  from its head to see if any thread's `sleeping_ticks` less than the global ticks
2. If any, reset its `sleeping_ticks`
3. Disable interrupts
4. Remove it from `sleeping_list` and unblock it
5. Unblock the thread 
6. Enable interrupts and reset to the old one
7. Repeat the step 1-6 until the `sleeping_list` is empty or a thread's `sleeping_ticks` is larger than the gloabl ticks

### Synchronization

As you can see from *Algorithms* section, interrupts are disabled for steps 4-5 in `timer_sleep` and `sleeping_list_handle`. Since lists in Pintos are not thread-safe and `ready_list` and `sleeping_list` are modified in step 4-5 for each function,  they should disable interrupts in this process. 

### Rationale

At the begining, I choose to use `sleeping_ticks` to store the ticks how long it can sleep and decrease it using ` thread_foreach` each time. This solution don't need to keep the `sleeping_list` as a sorted list, but when there are many sleeping threads, it will iterate the whole list to decrease the `sleeping_ticks`, which is ineffiecient. 

Therefore, I use  `sleeping_ticks` to store the ticks at which the thread will be waked up and keep the  `sleeping_list` sorted. The complexity of the inserting to a list is O(n), which n denotes the length of the list. And the unblock process also takes O(n), but in most of time, it will leave the loop earlier.

As I listed in the *Data structures and functions* and *Algorithms* section, there are only few functions to implement and modify, so it won't take too much time.



## Task 2: Priority Scheduler

### Data structures and functions

in `thread.h`:

* add the following codes into struct `thread` 

  ```C
  int base_priority; // store the original priority before donation
  struct list donators; // a list of threads waiting on the locks this thread has
  struct lock *lock_blocked_by; // the thread blocked by the lock
  ```

* add the following definition

  ```c
  #define LOCK_LEVEL 8 // lock depth
  ```

in `thread.c`:

- add `bool less_priority(const struct list_elem *a, const struct list_elem *b, void * aux)`  to compare two threads' priority
- modify the following functions, replace `list_push_back` with `list_insert_ordered` using `less_priority`
  - thread_unblock
  - thread_creat
  - thread_yield

in `synch.h`:

* add the following codes into struct `lock` 

  ```C
  struct list_elem elem; // list element for lock to track the priority donation
  int max_priority; // the highest priority among the threads waiting on the lock.
  ```

in `synch.c`:

- modify `void sema_up (struct semaphore *sema)` and `void sema_down (struct semaphore *sema)` functions which can unblock a waiter with highest priority
- modify `void cond_signal (struct condition *cond, struct lock *lock UNUSED)` which can pick a waiter with highest priority
- modify `bool lock_acquire (struct lock *lock)` which can donate priority from the caller to the lock holder 
- modify `void lock_release (struct lock *lock)` which can restore owner's priority

### Algorithms 

Here list two core functions' algorithms design. There are many other functions to be modified, most of them are list in *Data structures and functions* section and they just change the original FIFO list to a priority queue.

In function `lock_acquire()`:

1. Disable interrupts

2. Donation

   2.1 IF `lock->holder` is not NULL, compare lock holder’s priority with current thread’s priority

   2.1.1 IF lock holder’s priority < current thread’s priority

   2.1.1.1 Set lock holder’s priority to current thread’s priority

   2.1.1.2 Add the current thread to the lock holder's donators list

   2.1.1.3 Insert the current thread to the `ready_list` if its status is ready

   2.1.1.4 Replace the holder's thread with `thread->lock_blocked_by`

   2.1.1.4 Repeat step 2.1.1.1-2.1.1.4 until the holder thread is NULL or it reaches the `LOCK_LEVEL` defined before.

   2.1.2 ENDIF

   2.2 ENDIF

   2.3 Execute `sema_down`: if sema value is 0, put all threads acquiring this lock into the sema’s waiters list until sema value becomes positive 

   2.4 Set the current thread to this lock’s holder

3. Enable interrupts and reset to the old one

In function `lock_release()`:

1. Make sure this thread is the holder of this lock

2. Disable interrupts.

3. Set the lock holder to NULL

4. Execute `sema_up`: increase the sema value by 1, which means this lock can be get by its `semaphore.waiters` or any thread is going to acquire it

5. Reset the lock_holder’s original priority value
   5.1 IF no donation happened

   5.1.1 Set lock holder’s priority value to its original priority value

   5.2 ELSE

   5.2.1  IF original lock holder holds only this lock

   5.2.1.1  Set lock holder’s priority value to its original priority value

   5.2.2  ELSE

   5.2.2.1  Set lock holder’s priority to the highest priority in its donators' list.

6. Enable interrupts and reset to the old one

### Synchronization

When donating the priority, the lock holder’s priority can be set by it’s donator, and it also can be changed by itself. If the donators and the thread itself set the priority in a different order, that may cause a different result. To solve this problem, the priority of the lock holder is saved before donation, so that it can be restored after it releases the lock. If the a holder tries to lower its priority after donation, a change is only made to the saved priority `base_priority`, leaving the current priority unchanged.

### Rationale

Initially, I just think if a new thread acquire a lock held by a lower priority thread, then the thread trying to acquire the lock will donate its priority to the holder and finally wait for the lock. However, this design can't handle the case of nested donation in spite of less time overhead. The current design works fine with nested donation cases and can completely eliminates the priority inversion, though it involves the complexity of maintaining the donators' list. 

After organizing all ideas discussed above, this task is not so hard, but there are much more details than task 1 that I must take careful of in my implementations.

## Task 3: Multi-level Feedback Queue Scheduler (MLFQS)

### Data structures and functions

in `thread.h`:

- add the following codes into struct `thread` 

  ```C
  int nice; // the thread's current nice value
  fixed_t recent_cpu; // most recently calculated cpu value
  ```

in `thread.c`:

- add the following variable into  `thread.c` 

  ```C
  fixed_t load_avg; // most recently calculated load average value
  ```

- modify `void thread_tick (void)` according to the following logic

  - if current thread is not idle thread, increase its `recent_cpu` by 1
  - if `ticks` reaches multiple of `4`, recalculate the priority of current thread according to the formula 1 shows in the *Algorithms* section
  - if `ticks` reaches multiple of `TIMER_FREQ`, recalculate `load_avg`, iterate each thread and update `recent_cpu` and their priority, remove them from their old priority queue and push them into their new priority queue

- implement functions `int thread_get_nice (void) `, `void thread_set_nice (int nice UNUSED)`, `int thread_get_load_avg (void) ` and `int thread_get_recent_cpu (void) ` according to the functions and variables defined before

### Algorithms 

Refer to the document, we have three formulas:

formula 1:

$𝑝𝑟𝑖𝑜𝑟𝑖𝑡𝑦 = 𝑃𝑅𝐼\_𝑀𝐴𝑋 − (𝑟𝑒𝑐𝑒𝑛𝑡\_𝑐𝑝𝑢/4) − (𝑛𝑖𝑐𝑒 × 2)​$

formula 2:

$recent\_cpu = (2*load\_avg)/(2*load\_avg+1)*recent\_cpu+nice$

formula 3:

$load\_avg = (59/60) × load\_avg + (1/60) × ready\_threads$

In function `thread_tick (void)`:

1. IF current thread is not idle thread

   1.1 increase its `recent_cpu` by 1

2. IF `ticks` reaches multiple of `4`

   2.1 IF current thread is not idle thread

   2.1.1 recalculate the priority of current thread according to the formula 1

3. IF `ticks` reaches multiple of `TIMER_FREQ`

   3.1 IF current thread is not idle thread, add `ready_threads` 1 

   3.1 recalculate `load_avg` according to the formula 3

   3.2 iterate each thread

   3.2.1 update `recent_cpu` according to formula 2, then update `priority` according to the formula 1

   3.2.2 remove the thread from its old priority queue and push it into their new priority queue

### Synchronization

Most of the modifications are protected by disabling interrupts, so they will not cause race conditions problem. In `thread_set_nice`, I not only modify a thread's nice value, but also its priority which leads to the change of thread queues. If a thread switch occurs, this may cause a problem, so it's better to disable interrupts completely for this method.

### Rationale

If only use one queue to store the ready threads, every time we insert a thread, it will cost linear time to keep it ordered, after recalculations of the each thread's priority, we need to sort the queue again, which will cost O(nlogn) time. Compared to using an array of 64 queues, the time complexity to insert a thread is constant, and a linear time for recalculations.

# Additional Questions

**Q1 (This question uses the MLFQS scheduler.) Suppose threads A, B, and C have nice values 0, 1, and 2. Each has a recent_cpu value of 0. Fill in the table below showing the scheduling decision and the recent_cpu and priority values for each thread after each given number of timer ticks. We can use R(A) and P(A) to denote the recent_cpu and priority values of thread A, for brevity.**

| timer ticks | R(A) | R(B) | R(C) | P(A) | P(B) | P(C) | thread to run    |
| ----------- | ---- | ---- | ---- | ---- | ---- | ---- | ---------------- |
| 0           | 0    | 0    | 0    | 63   | 61   | 59   | A                |
| 4           | 4    | 0    | 0    | 62   | 61   | 59   | A                |
| 8           | 8    | 0    | 0    | 61   | 61   | 59   | B（Round robin） |
| 12          | 8    | 4    | 0    | 61   | 60   | 59   | A                |
| 16          | 12   | 4    | 0    | 60   | 60   | 59   | B（Round robin） |
| 20          | 12   | 8    | 0    | 60   | 59   | 59   | A                |
| 24          | 16   | 8    | 0    | 59   | 59   | 59   | C（Round robin） |
| 28          | 16   | 8    | 4    | 59   | 59   | 58   | B                |
| 32          | 16   | 12   | 4    | 59   | 58   | 58   | A                |
| 36          | 20   | 12   | 4    | 58   | 58   | 58   | C（Round robin） |

**Q2 Did any ambiguities in the scheduler specification make values in the table (in the previous question) uncertain? If so, what rule did you use to resolve them?**

First, the question doesn't specify the `PRI_MAX` and the number of ticks per second. I assume that these values are based on the project, so `PRI_MAX = 63`, and that there are 100 ticks per second so we don't need to consider the change of  `load_avg` since there are only 36 ticks. 

Second, if there are two threads with the same priority, it's unclear which thread to run, so I assume they're ordered in a round-robin fashion, it'll choose the thread that run the least recently. (In my design, if a thread's priority is changed, it'll add to a new priority queue's tail, so the thread that run the least recently is in the head of the new priority queue, it's easy to implement)

**Q3  Tell us about how pintos start the first thread in its thread system (only consider the thread part)**

From the comment for function  `void thread_init (void) ` in `thread.c`, the first thread is created by transforming the code that is currently running into a thread.

> This can't work in general and it is possible in this case only because `loader.S` was careful to put the bottom of the stack at a page boundary.

**Q4 Consider priority scheduling, how does pintos keep running a ready thread with highest priority after its time tick reaching TIME_SLICE?**

The current code can't realize this, from the following image, we can see the next thread is pop from `ready_list`'s head. However, when current thread reaches TIME_SLICE, it'll insert into the `ready_list`'s tail. If we want to realize this function, we must keep the` ready_list` ordered, that is, the thread in it is according to a decreasing priority rank. In this case, the ready thread with highest priority will be inserted to the head of `ready_list` and it'll be pop up first in the next time slice.

![image-20190317210418269](/Users/nicole/Library/Application Support/typora-user-images/image-20190317210418269.png)



**Q5 What will pintos do when switching from one thread to the other? By calling what functions and doing what?**

Calling `schedule`, and this function will call `thread_schedule_tail`, which will destroy the current running thread if it's dying and mark the next thread as the running thread.  Set `thread_ticks = 0;` to start a new time slides.

**Q6 How does pintos implement floating point number operation**

All calculations on floating point number are simulated using integers. From `#define FP_SHIFT_AMOUNT 16` , we know that pintos use 16 bits for its fractional part, so if a number is of `fixed_t` type, we can treat the rightmost 16 bits as floating point number's fraction part, and the remining part is the floating point number's integer part.

Understanding the floating number's storage stategy, we can easily implement addition, substraction, multiplication and division among floating numbers or a floating number with an integer.

**Q7 What do priority-donation test cases(priority-donate-chain and priority-donate-nest) do and illustrate the running process**

**priority-donate-chain:**

*Function:* 

test the correction of program if there exists a priority donate chain 

*Running process：*

Set the current thread's priority to `PRI_MIN`, and acquire `locks[0]` , create seven (NESTING_DEPTH-1) threads with each thread's priority 3,6,9,12… and set each thread's execution function to `lock_pairs[i]`. Each time this thread creates, it will invoke `donor_thread_func` to adopt preemptive scheduling strategy. After creating this thread, each loop will create another thread with priority equals to `thread_priority-1`, this time it won't invoke `donor_thread_func`.

After original_thread releaase locks[0], thread 1 is waked up and output the message and then release the lock output the current priority. Because this thread is donated by other thread with higher priority, the priority is equal to 21, and then lead to run the next thread. Only we reached the last thread, it hasn't been donated, so its priority turns to original priority.



**priority-donate-nest:**

*Function:* 

test the correction of program if there exists a nested donation

*Running process：*

creat a thread with medium priority has two locks a and b, and a thread with high priority has a lock b. The lock a is hold by low thread, and the lock b is hold by medium thread. So when the high thread try to acquire lock b, the medium thread's priority is promoted, so the medium thread try to get lock a which is hold by low thread, so the low thread's priority is also promoted.