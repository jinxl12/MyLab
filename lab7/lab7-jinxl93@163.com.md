 学号：2012080056 姓名：金贤林 班级：计23

[练习1] 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

在这个过程中, 需要一个锁`mutex`来控制临界区的访问, 以及每个哲学家对应的信号量`s[]`来标示每个哲学家的状态, 对于每个哲学家来说, 有两个核心操作, 一个是`phi_take_forks_sema`用于得到叉子, 另一个是`phi_put_forks_sema`, 用于将叉子放回, 并且提醒相邻的两位哲学家.

`phi_take_forks_sema`会将哲学家的状态标示为`HUNGRY`, 并且尝试获得叉子, 如果相邻两位哲学家都没有在用叉子, 那么该哲学家获得叉子, 并将自己的状态标注为`EATING`, 并`up`自己的信号量, 尝试结束之后, 哲学家会`down`自己的信号量, 如果之前获得了叉子, 那么信号量的资源会被释放, 该进程继续运行, 反之, 该进程就会陷入等待之中, 进入对应的等待队列.

`phi_put_forks_sema`会让哲学家放下自己的叉子, 将哲学家的状态置为`THINKING`, 并且提醒相邻的哲学家尝试获得叉子, 如果相邻的哲学家(以下称作"哲学家2")获得叉子(前提是哲学家2很饿而且相邻的哲学家已经不用叉子了), 那么哲学家2对应的信号量就会加1, 这样就会哲学家2的进程被重新唤醒, 继续执行.

[练习1.1]内核级信号量的设计描述, 与大致执行流流程

信号量是基于`semaphore_t`类型的, `semaphore_t`类型包括两部分, 一个是`value`, 用于记住资源的数量, 另一个是一个`wait_queue`, 用来记录等待资源的队列.

对于一个信号量, 包含三个操作`up`, `down`, `try_down`, 三个操作都需要屏蔽中断, `up`操作会给相应的信号量资源加一, 并且将等待队列中最前的进程取出, 唤醒该进程, `down`操作会将信号量的资源是否为正, 如果为正, 则减一后返回, 否则就使进程进入"等待"状态, 兵加入到等待队列之中, 进行进程调度, `try_down`操作也会将信号量的资源减一, 但如果资源不为正的话, `try_down`什么也不会做, 直接返回.  

[练习1.2]用户态进程/线程信号量机制的设计方案

用户态的信号量机制, 只要对内核态下的信号量机制进行封装即可, 在内核维护一个信号量的数组, 用`sem_init`来得到一个内核信号量的id, 用`wait`操作来使用系统调用, 调用内核态下的`down`, 用`post`操作来, 通过系统调用调用内核中的`up`即可.

[练习2]内核级条件变量和基于内核级条件变量的哲学家就餐问题


在这个过程中, 需要一个管程管理进程间的关系, 对于每个哲学家来说, 同样有两个核心操作, 一个是`phi_take_forks_sema`用于得到叉子, 另一个是`phi_put_forks_sema`, 用于将叉子放回, 并且提醒相邻的两位哲学家.

`phi_take_forks_sema`会将哲学家的状态标示为`HUNGRY`, 并且尝试获得叉子, 如果相邻两位哲学家都没有在用叉子, 那么该哲学家获得叉子, 并将自己的状态标注为`EATING`, 并`signal`自己的条件变量, 尝试结束之后, 哲学家会`wait`自己的条件变量, 如果之前获得了叉子, 那么该进程继续运行, 反之, 该进程就会陷入等待之中, 进入对应的等待队列.

`phi_put_forks_sema`会让哲学家放下自己的叉子, 将哲学家的状态置为`THINKING`, 并且提醒相邻的哲学家尝试获得叉子, 如果相邻的哲学家(以下称作"哲学家2")获得叉子(前提是哲学家2很饿而且相邻的哲学家已经不用叉子了), 那么哲学家2就会`signal`自己的条件变量, 这样就会哲学家2的进程被重新唤醒, 继续执行.

但在管程中不同的是, 操作系统一定会响应完所有在等待队列中的进程的请求, 才会接受新的请求, 这是一个管程中有一个`next`和一个`mutex`所导致的.  

[练习2.1]内核级条件变量的设计描述与其大致执行流流程

条件变量一般在管程的持有之下使用, 一个管程包括一个保护临界区的信号量`mutex`(资源被初始化为1), 以及另一个信号量`next`(资源被初始化为0), 用于唤醒在等待队列中进程的锁, 以及一个32位的整数`next_count`, 用于存储该管程下所有等待进程的个数, 以及一个条件变量的指针`cv`, 用于管理该管程下所有的条件变量.

而条件变量本身, 包含以下的成员, 一个信号量`sem`, 一个32位整数`count`, 用于记录在自己等待队列中的进程数量, 以及一个指向自己管理者的管程指针`owner`.

对于条件变量来说, 有两个方法`cond_signal`, `cond_wait`, `cond_signal`会让对应的条件变量中的条件量的资源增加1, 同时, 也会让`next`的资源减1(即等待处于"等待"状态的进程出现), 而`cond_wait`会检查是否现在有处于"等待"状态的进程, 如果有让`next`的资源增加1, 允许"等待"进程进入调度, 如果没有"等待"进程, 就释放临界锁, 允许新的请求, 之后让等待的进程的信号量的资源减少1.

[练习2.1]用户态进程/线程条件变量机制的设计方案

用户态的条件变量, 对内核态的条件变量进行封装即可, 在内核当中维持条件变量的数组, 使用`get_cond`获得对应的条件变量的id, 使用`signal`和`wait`对相应的条件变量进行操作, 但唯一不同的是, 在这里是传入条件变量的id而非其地址. 另外, 需要使用系统调用来使`signal`和`wait`调用内核态下的`cond_wait`和`cond_signal`.
