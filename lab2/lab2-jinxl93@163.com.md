 学号：2012080056 姓名：金贤林 班级：计23
[练习1] 你的first fit算法是否有进一步的改进空间？
设计思路：
static void 
default_init(void):
	用list_init函数初始化free_list。
	nr_free(表示空区域的个数）设置为0。

static void 
default_init_memmap(struct Page*base, size_t n):
	从base~base+n,对每个页，p->property设置为0,p->flags设置为PG_property,p->ref也设置为0。用list_add_before函数把每个空闲页加到free_list里面。
	最后nr_free增加n,base->property设置为n。

static struct Page*
default_alloc_pages(size_t n)（找到第一个合适的空闲页，并再调整剩下的空闲页）
	用list_next()函数遍历free_list的所有空闲区，直到找到大小不小于n的空闲区。找到相应的空闲区以后，n个空闲页的PG_reserved =1, PG_property =0，然后用list_del函数把该n个空闲页从free_list删除掉。
	如果空闲区的大小大于n，剩下的区域也构成一个空闲区保存到free_list里面。
	nr_free减少n。

static void
default_free_pages(struct Page *base, size_t n)	（把小块合成大块）
	先用list_add_before函数把n个页加到free_list（这时要注意一定要保持地址顺序)。
	设置相应n个空闲区的属性（p->ref,p->flags等等）
	如果n个空闲区后面还有连续的空闲区，把两个空闲区合成一个大的空闲区。

改进的办法（自己随便想的）：对空闲区的初始化和删除等操作的时候，一定要设置所有页的属性（比如flags,ref等等，加或删除的时候对每个页都一一操作，会耗很多時間），这种方法会影响算法的效率。 可以只对一个空闲快的第一个页进行操作（本人试过这种办法，但是最终没有实现）。
[练习2]

[练习2.1] 请描述页目录项(Page Director Entry)和页表(Page Table Entry)中每个组成部分的含义和以及对ucore而言的潜在用处。
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

[练习2.2] 如果ucore执行过程中访问内存,出现了页访问异常,请问硬件要做哪些事情?
_如果ucore执行过程中访问内存, 出现了页访问异常, 请问硬件要做哪些事情_

出现了页访问异常, 硬件会将eflags, cs寄存器的值, eip的值, 错误码压入栈中, 如果涉及到了用户态到内核态的切换, 那么还会在压入前面几个值之前, 更换一个栈, 并且压入ss的值和esp的值. 

另外, 会将引起异常的访问地址存在cr2寄存器当中. 


[练习3] 

[练习3.1] 数据结构Page的全局变量(其实是一个数组)的每一项与页表中的页目录项和页表项有无对应关系?如果有,其对应关系是啥?

page结构体是物理页面的管理者, 因此和系统中的物理页面是一一对应的, 因此其排列顺序也是按照其对应的物理页面地址进行的, 所以可以使用`pte2page`函数从页表项找到相应的物理页面的Page结构体, `pte2page`实现如下: 

```
#define PTE_ADDR(pte)   ((uintptr_t)(pte) & ~0xFFF)

#define PPN(la) (((uintptr_t)(la)) >> PTXSHIFT)

pa2page(uintptr_t pa) {
    if (PPN(pa) >= npage) {
        panic("pa2page called with invalid pa");
    }
    return &pages[PPN(pa)];
}

static inline struct Page *
pte2page(pte_t pte) {
    if (!(pte & PTE_P)) {
        panic("pte2page called with invalid pte");
    }
    return pa2page(PTE_ADDR(pte));
}
``` 

可以看出page结构体和对应的物理页面的首地址是一一对应的, 换句话说, 第X个物理页面, 正好由pages[i]管理, 因此从pte到对应的page结构体.  
[练习3.2] 如果希望虚拟地址与物理地址相等,则需要如何修改lab2,完成此事? 鼓励通过编程来具体完成这个问题。
