---
layout: post
title: "Learn C++: memory order and atomic variables"
date: 2023-09-05
categories: CATEGORY-1 CATEGORY-2
---
## why memory order is needed
Because of the performance gap between CPU and memory, cache is employed. In multi-core machine, every core has its exclusive L1 cache, which causes another core may not see the memory change immediately after the write.  
Additionally, while compiler optimizes the program, it may reorder the code according to "As-if rule", namely the code will act exactly as the that without reordering in **single thread**. It causes problem in multi-thread program.  
When we use `mutex`, the above problems should not be concerned because `mutex` implementation has solved them. ([how?](todo))  
When we use atomic variable, memory order can be specific in every atomic operation, so that we can opitimize further.  
## memory order
Here is the memory order enum in atomic operation function.
1. `memory_order_relaxed`: No constrain on synchronization and code reorder. Only promise the operation's atomicity and **modification order consistency**.  
2. `memory_order_consume`: Similar to `memory_order_acquire` but looser.  
3. `memory_order_acquire`: Use in **load** or **RMW** operation. Prevent the code after the operation reorder to the front.  
4. `memory_order_release`: Use in **store** or **RMW** operation. Prevent the code before the operation reorder to the back.  
5. `memory_order_acq_rel`: Use in **RMW** operation. Prevent any reorder that cross the operation.  
6. `memory_order_seq_cst`: The default order. Prevent any reorder across. And read from(load) or write to(store) the global memory (instead of cache). This order SC-DRF is the same as sequence order when the program has no race condition.  

Note that not every operation is allowed to use any memory order enum. When the not allowed memory order is used, MS STL implemented it as `memory_order_seq_cst`.  
And when we assign or read atomic variable directly use `=` operator, we actually call the `=` overload and use the default `memory_order_seq_cst`.  


## SC-DRF(Sequential consistency for data race free) model
example:  
```cpp
std::atomic<bool> x, y;
std::atomic<int> z;

// thread 1
void write_x() { x.store(true, std::memory_order_seq_cst); }

// thread 2
void write_y() { y.store(true, std::memory_order_seq_cst); }

// thread 3
void read_x_then_y() {
    while (!x.load(std::memory_order_seq_cst))
        ;
    if (y.load(std::memory_order_seq_cst))
        ++z;
}

// thread 4
void read_y_then_x() {
    while (!y.load(std::memory_order_seq_cst))
        ;
    if (x.load(std::memory_order_seq_cst))
        ++z;
}
```
In this example, z cannot be 0 after 4 threads finish. If thread3 if condition is not met, thread 4 should spin in the while loop. And since thread 3 while loop is break, `x` must be `true`. So thread 4 if condition must be met.  
The memory order should only be **seq_cst** because the **load** in if condition should load what is actually in global memory.

## acquire release model
example:  
```cpp
// thread1
void write_y_then_x() {
    y.store(1, std::memory_order_relaxed);
    x.store(true, std::memory_order_release);
}

// thread2
void read_x_then_y() {
    while (!x.load(std::memory_order_acquire))
        ;
    if (y.load(std::memory_order_relaxed) == 0) {
        // this can not happen
        std::cout << "NO!!!\n";
    } else {
        // std::cout << "YES\n";
    }
}
```
The **x store** in thread1 synchronized with the **x load** in thread1. And the **y store** in thread1 is not allowed to reorder to back and the **y load** is not allowed to reorder to front. And because the modification order consistency. The condition cannot be meet forever.
