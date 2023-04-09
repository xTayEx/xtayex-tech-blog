---
title: "CMU15418 Lecture 6"
date: 2023-04-09T22:58:02+08:00
draft: false
---

# Work Distribution and Scheduling

## Key goals

- Balance workload onto available execution resources
- Reduce communication
- Reduce extra work overhead performed to increase parallelism, manager assignment, reduce communication, etc.

## Distribution technique

### Static assignment

- What is it?
    
    Assignment of work to threads is pre-determined
    
- Advantages: simple, zero runtime overhead for distribution
- Use case
    - it is known up front that all work has the same cost
        
        ![](https://s2.loli.net/2023/04/09/nPS2OGfgzwHMVk9.png)
        
    - When work is predictable, but not all jobs have same cost
    
    ![](https://s2.loli.net/2023/04/09/irOJ3SMRmIG2pcq.png)
    

### Semi-static assignment

- Cost of work is predictable for near-term future
    - recent past good predictor of near future
- Application periodically profiles itself and re-adjusts assignment
    - Assignment is “static” for the interval between re-adjustments
    
    ![](https://s2.loli.net/2023/04/09/gyVvxW4foEipDZQ.png)
    

### Dynamic assignment

- Program determines assignment dynamically at runtime to ensure a well distributed load. (The execution time of tasks, or the total number of tasks, is unpredictable.)

### Choose a proper task size(or granularity)

- fine granularity partitioning
    - may cost high synchronization cost
    
    ![](https://s2.loli.net/2023/04/09/a8LS3lV5wruYqAI.png)
    
- coarse granularity partitioning
    - can decrease the synchronization cost
    - but may not enable good workload balance balance via dynamic assignment (smaller tasks make it easy for the system to split all tasks to some almost equal-sized task groups)
    - Ideal granularity depends on many factors(may be introduced in the later lectures?)
    

### Smarter task scheduling

- if the system assigns tasks to works from left to right, workload imbalance may occur.

![](https://s2.loli.net/2023/04/09/SavxNrtkqQK3dYp.png)

![](https://s2.loli.net/2023/04/09/LNXuGzMK5Bh7q4i.png)

- possible solutions:
    - Divide work into a larger number of smaller tasks (May not be possible (perhaps long task is fundamentally sequential))
    - Schedule long task first to reduce “slop” at end of computation (recall the event scheduling problem in algorithm class)
    
    ![](https://s2.loli.net/2023/04/09/OIk419KzpbyqneJ.png)
    

### Decreasing synchronization overhead using a distributed set of queues

- avoid need for all workers to synchronize on single work queue

![](https://s2.loli.net/2023/04/09/t7R2xlWhLDaeSAV.png)

### Common parallel programming patterns

- Data parallelism

Perform same sequence of operations on many data elements

- Fork-join pattern
    - pthread’s create and join
    - clik plus
        - `cilk_spawn foo(args)` → fork. invoke foo, but unlike standard function call, caller may continue executing asynchronously with execution of foo
        - `cilk_sync` → join: returns when all calls spawned by current function have completed. (“sync up” with the spawned calls)
        - some example for cilk plus
        
        ![](https://s2.loli.net/2023/04/09/EpYIvhtjmAZHFwz.png)
        
        - abstraction vs. implementation
            
            Notice that the `cilk_spawn` abstraction does not specify how or when spawned calls are scheduled to execute. But `cilk_sync` does serve as a constraint on scheduling
            

### More detail about clik plus

- `clik_spawn`
    - hide implementation, like `std::async`
    - run continuation first (child stealing): Caller thread spawns work for all iterations before executing any of it (BFS)
    - run child first (continuation stealing): Caller thread only creates
    one item to steal (continuation that represents all remaining
    iterations) (DFS)
- `clik_sync`
    - stall join
    - greedy policy: When thread that initiates the fork goes idle, it
    looks to steal new work. Last thread to reach the join point continues
    execution after sync
        - All threads always attempt to steal if there is nothing to do (thread only goes idle if no work to steal is present in system)
        - Worker thread that initiated spawn may not be thread that executes logic after `cilk_sync`
- Quick sort example (continuation stealing)

```cpp
void quick_sort(int *begin, int *end) {
  if (begin >= end - PARALLEL_CUTOFF) {
    std::sort(begin, end);
  } else {
    int *middle = partition(begin, end);
    cilk_spawn quick_sort(begin, middle);
    quick_sort(middle + 1, last);
  }
}
```

![](https://s2.loli.net/2023/04/09/w18uWKIB5O7PMSo.png)

![](https://s2.loli.net/2023/04/09/JBcILZGC4Ku6qOQ.png)

- Implementing work stealing: dequeue per worker
    - Local thread pushes/pops from the “tail” (bottom)
    - Remote threads steal from “head” (top)
    - Idle threads randomly choose a thread to attempt to steal from (random choice of victim)

### Details about cilk_sync

- When no stealing, there’s nothing to do at the sync point. `clik_sync` is no-op (all job is done by thread 0 itself)
- Otherwise, there are two policies
    - “stalling” join policy. Thread that initiates the fork must perform the sync.
        - A table (shown below) is used to keep the “stealing” information
        
        ![](https://s2.loli.net/2023/04/09/oR4aTJjEe2GiI5P.png)
        
    - “greedy” policy. When thread that initiates the fork goes idle, it looks to steal new work. Last thread to reach the join point continues execution after sync.
    - **Cilk uses greedy join scheduling**
