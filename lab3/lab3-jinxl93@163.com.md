 学号：2012080056 姓名：金贤林 班级：计23
[练习1]

设计思路

`do_pgfault`中已经有了差错控制, 因此只需要真正地处理页缺失. 先使用`get_pte`获得相应的页表项, 如果页表项为空, 表示对应的物理页面不存在, 则为该页表申请一个页面, 如果页表项不为空且现在允许页面换入, 则将页面换入, 并且调整页表项, 使其正确地指向页面, 要注意需要对对应的page结构体中的`Page::pra_vaddr`进行赋值(在这里的作用类似于初始化), 并且将该页面加入到交换链表中. 


[练习1.1] 请描述页目录项(Page Director Entry)和页表(Page Table Entry)中组成部分对ucore实现页替换算法的潜在用处.

页目录项是指向储存页表的页面的, 所以本质上与页表项相同, 结构也应该相同. 每个页表项的高20位, 就是该页表项指向的物理页面的首地址的高20位(当然物理页面首地址的低12位全为零), 而每个页表项的低12为, 则是一些功能位, 可以通过在`mmu.h`中的一组宏定义发现. 

```
#define PTE_P           0x001                   // Present 对应物理页面是否存在
#define PTE_W           0x002                   // Writeable 对应物理页面是否可写
#define PTE_U           0x004                   // User 对应物理页面用户态是否可以访问
#define PTE_PWT         0x008                   // Write-Through 对应物理页面在写入时是否写透(即向更低级储存设备写入)
#define PTE_PCD         0x010                   // Cache-Disable 对应物理页面是否能被放入高速缓存
#define PTE_A           0x020                   // Accessed 对应物理页面是否被访问
#define PTE_D           0x040                   // Dirty 对应物理页面是否被写入
#define PTE_PS          0x080                   // Page Size 对应物理页面的页面大小
#define PTE_MBZ         0x180                   // Bits must be zero 必须为零的部分
#define PTE_AVAIL       0xE00                   // Available for software use 用户可自定义的部分
```

PTE_P, PTE_W, PTE_U在ucore中已经用得很多, 就不再赘述, PTE_A和PTE_D可以用于虚拟内存管理和页面的置换算法, PTE_PWT和PTE_PCD可以用于影响cache的行为. 

[练习1.2] 如果ucore的缺页服务例程在执行过程中访问内存, 出现了页访问异常, 请问硬件要做哪些事情？

出现了页访问异常, 硬件会将eflags, cs寄存器的值, eip的值, 错误码压入栈中, 如果涉及到了用户态到内核态的切换, 那么还会在压入前面几个值之前, 更换一个栈, 并且压入ss的值和esp的值. 

另外, 会将引起异常的访问地址存在cr2寄存器当中. 

以上结论来自ucore的`trap.c`文件中. 

[练习2]

实现思路

在这里实现思路非常简单, 如果head是swap链表头, entry待插入的页, ptr_page是指向被换出页面的指针, 由于swap链表是一个双向链表, 因此对于换入, 只需要

```
list_add_before(head, entry);
```

换出

```
*ptr_page = le2page(list_next(head), pra_page_link);
list_del(list_next(head));
```

简单说就是, 换入时放在对尾, 同时也是链表头的前驱, 换出时返回后继. 


[练习2.1]如果要在ucore上实现"extended clock页替换算法"请给你的设计方案, 现有的swap_manager框架是否足以支持在ucore中实现此算法?如果是, 请给你的设计方案.

在ucore中实现"extended clock页替换算法"是完全可行的, 由于现有的swap队列已经是一个环了, 因此可以不做改动, 只对`map_swappable`和`swap_out_vistim`进行修改即可. 

由于扩展时钟算法需要一个"时针", 因此需要在struct mm里添加一项指向swap链表中某项的指针. 

由于扩展时钟算法需要物理页面对应页表项里的一些位的信息, 因此需要的到页面对应的页表项, 方法是

```
get_pte(mm->pagir, page->pra_vaddr, 0);
```

在扩展时钟算法中, 需要考察页表项的PTE_A位和PTE_D位, 在这里都能满足, 算法的具体流程, 就不再赘述. 

[练习2.1.1] 需要被换出的页的特征是什么?

PTE_A和PTE_D都被置为了零. 

[练习2.1.2] 在ucore中如何判断具有这样特征的页

判断页面对应页表项的第5位和第6位(最低是第0位)

[练习2.1.3] 何时进行换入和换出操作?

只要发生缺页异常, 且页表项存在, 且允许换入(在ucore中是`swap_init_OK`不为零)即进行换入操作, 在尝试换入时, 如果物理页面已经用尽, 或是达到某一阈值, 则应该换出, 但在ucore中似乎没有体现出这一点. 
