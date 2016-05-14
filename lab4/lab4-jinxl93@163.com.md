 学号：2012080056 姓名：金贤林 班级：计23

[练习1]

设计思路

在这里的练习只需要将proc_struct结构体中的成员变量作初始化即可. 

[练习1.1] 请说明proc_struct中`struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥?

`context`用于保存进程切换的上下文, `tf`用于构造中断帧, 具体的用途如下:

在拷贝线程的时候(do_fork当中), 会为`context.eip`和`context.esp`赋值, `eip`的值指向`forkret`, 而`esp`的值指向自己记录的trapframe. 

具体用到`context`的地方, 是在一个`switch_to`的汇编函数(见`switch.S`)当中, 通过汇编函数, 将当前的`%eip`, `%esp`, `%ebx`, `%ecx`, `%edx`, `%esi`, `%edi`, `%ebp`保存在`from`(`context`)之中, 将`to`中的几个寄存器的值存到现在的寄存器当中. 

这个函数(`switch_to`)准备返回的时候, 由于栈顶被压入了`to.eip`, 因此会直接跳到函数`forkret`, 并直接调用函数`forkrets`, 而`forkrets`函数是将栈顶指向当前进程储存的trapframe, 然后跳到`__trapret`之后再从异常处理中返回. 

[练习2]


设计思路

这部分需要填写`do_fork `的代码, 思路与实验指导书上的一致, 首先调用alloc_proc, 获得一块用户信息块, 然后为进程分配一个内核栈, 之后复制原进程的内存管理信息到新进程(但内核线程不必做此事), 接着复制原进程上下文到新进程, 然后将新进程添加到进程列表, 最后唤醒新进程并返回新进程号.


[练习2.1] 请说明ucore是否做到给每个新fork的线程一个唯一的id?

在ucore中`MAX_PROCESS`为4096, `MAX_PID`为`MAX_PROCESS`的两倍, 因此, id的数量是足够用于分配给进程的.

用于id分配的函数, 是在`proc.c`中的`get_pid`中实现的, `get_pid`的总体思路非常朴素, 设置一个计数器`last_pid`, 每次有进程申请id号, 就将`last_pid`加一之后返回, 如果`last_pid`超过了`MAX_PID`, 那么就将`last_pid`置为1之后返回, 这样的处理方式可能导致进程号重复, 以下是解决方法. 

最朴素的方法, 就是在每次分配id时遍历proc_list, 但这效率太低, 因此在遍历的时候, 确定一个`next_safe`值, 即当前进程队列中, 大于`last_pid`的最小pid, 这样就可以避免每次都遍历`proc_list`.

[练习3]

[练习3.1] 在本实验的执行过程中，创建且运行了几个内核线程?

创建了两个线程`idle`和`init`, 但真正运行的只有`init`. 

[练习3.2] 语句`local_intr_save(intr_flag);`....`local_intr_restore(intr_flag);`在这里有何作用?请说明理由

在这里, 这两句话是屏蔽中断和恢复中断的两句话, 切换进程的四个步骤, 是不能被中途打断的, 因为`current`指针, 栈顶指针, 页目录首地址和上下文, 必须匹配, 因此必须全部执行或全部不执行, 因此需要屏蔽中断. 
