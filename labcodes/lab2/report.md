# Lab2

## 练习0  

已完成合并。

## 练习1  

#### 1 实现  
对各函数的简要说明如下：
default_init
这里是对系统需要用到的数据结构的初始化，包括对free_list链表的初始化和nr_free空闲页个数置0的初始化。
default_init_memmap
这个函数是对内存中空闲页的初始化，即将它们加入到空闲页的列表中。  
主要代码如下：  
```
struct Page *p = base;
for (; p != base + n; p ++) {
    assert(PageReserved(p));
    p->flags = 0;
    // 设置标志位表明该页有效
    SetPageProperty(p);
    // property的值表示以该页开头有多少个连续的页，除了第一页外都为0
    p->property = 0;
    // 开始的时候页都没有引用
    set_page_ref(p, 0);
    // 将页按顺序加到链表的末尾
    list_add_before(&free_list, &(p->page_link));
}
// first block
base->property = n;
nr_free += n;
```
default_alloc_pages
这个函数的作用是配从base开始的n个连续的页。
```
if (n > nr_free) {
    return NULL;
}
struct Page *page = NULL, *tempp = NULL;
// Search through the list
list_entry_t *le = &free_list, *nle = NULL;
while ((le = list_next(le)) != &free_list) {
    struct Page *p = le2page(le, page_link);
    if (p->property >= n) {
        page = p;
        break;
    }
}
if (page != NULL) { // Found valid position at (le, page)
    int i = 0;
    for (; i<n; ++i) {
        nle = list_next(le);
        tempp = le2page(le, page_link);
        ClearPageProperty(tempp);
        SetPageReserved(tempp);
        list_del(le);
        le = nle;
    }
    // Recalculate the number of blocks
    if (page->property > n) {
        (le2page(le, page_link))->property = page->property - n;
    }
    nr_free -= n;
}
return page;
```
default_free_pages
这个函数的作用是释放从base开始的n个连续页，并对空闲块做合并工作。
```
list_entry_t *le = &free_list;
while ((le=list_next(le)) != &free_list) {                     //按顺序寻找，找到第一个地址大于base的页
    if ((le2page(le, page_link)) > base)                       //这一页的前面就是要释放的块应该插入的位置
        break;
}

struct Page *p = base;
for (; p < base + n; ++p) { // 将空间依次加入链表
    set_page_ref(p, 0);
    ClearPageReserved(p);
    SetPageProperty(p);
    p->property = 0; //先统一设置property为0
    list_add_before(le, &(p->page_link));
}
base->property = n;
p = le2page(le, page_link); // 检查后面的点
if (p == base+n && PageProperty(p)) { // 如果可以合并
    base->property += p->property; // 则合并
    p->property = 0;
}

// 向前试探
le = list_prev(&(base->page_link));
if (le2page(le, page_link) == base-1) { // 是连续的，则继续向前试探
    while (le != &free_list) {
        p = le2page(le, page_link);
        if (p + p->property == base) { // 找到了前一段空间的开头
            p->property += base->property; // 则合并
            base->property = 0;
            break;
        }
        le = list_prev(le);
    }
}
nr_free += n;
```
#### 2 改进空间  

现在的方法在空闲页列表中需要存储每一页，而遍历的时候则是一页页遍历，通过property的值来判断块的大小，这样一方面浪费存储空间，同时也增大了试探时的时间成本。如果有一个块很大，那么需要遍历很多个页。我觉得可以改进存储方式，即将多个空闲页用一个块来储存，而每一块中则只需要记录第一页的起始地址和块中页的个数，这样在遍历的时候可以节约很多空间。在前向合并空间时也可以节省试探的时间。

## 练习二  

#### 1 实现  

get_pte函数是通过PDT的基址pgdir和线性地址la来获取pte
```
pde_t *pdep = &(pgdir[PDX(la)]);   // (1) find page directory entry
struct Page *page = NULL;
if (!((*pdep) & PTE_P)) {              // (2) check if entry is not present
    if (!create) {                  // (3) check if creating is needed, then alloc page for page table
        return NULL;
    }
    page = alloc_page();
    set_page_ref(page, 1); // (4) set page reference
    uintptr_t pa = page2pa(page); // (5) get linear address of page
    memset(KADDR(pa), 0, PGSIZE); // (6) clear page content using memset, MUST use virtual address
    *pdep = (pa | PTE_U | PTE_P | PTE_W);                 // (7) set page directory entry's permission
}
//现在pde的值已经存在了
//需要返回的值是pte的指针，这里先将pde中的地址转化为程序可用的虚拟地址
//将这个地址转化为pte数据类型的指针，然后根据la的值索引出对应的pte表项
//最后通过&取得它的指针返回
return &(((pte_t*)KADDR(PDE_ADDR(*pdep)))[PTX(la)]);          // (8) return page table entry
```
#### 2  

PDE和PTE的组成（详见mmu.h文件）
```
#define PTE_P           0x001                   // 当前项是否存在，用于判断缺页
#define PTE_W           0x002                   // 当前项是否可写，标志权限
#define PTE_U           0x004                   // 用户是否可获取，标志权限
#define PTE_PWT         0x008                   // 写直达缓存机制,硬件使用Write Through
#define PTE_PCD         0x010                   // 禁用缓存，硬件使用Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // 页是否被修改（dirty)
#define PTE_PS          0x080                   // 页大小
#define PTE_MBZ         0x180                   // 必须为0的位
#define PTE_AVAIL       0xE00                   // 软件使用的位，可任意设置
```
#### 3  
硬件需要根据设置的IDT找到对应的异常处理例程的入口，然后跳转到该处理例程处理该异常。

## 练习三  
#### 1 实现  

page_remove_pte函数：释放la地址所指向的页，并设置对应的pte的值
```
if ((*ptep) & PTE_P) {                      //(1) check if this page table entry is present
    struct Page *page = pte2page(*ptep); //(2) find corresponding page to pte
    page_ref_dec(page);                          //(3) decrease page reference
    if (page_ref(page) == 0) {                       //(4) and free this page when page reference reaches 0
        free_page(page);
    }
    tlb_invalidate(pgdir, la);              //(6) flush tlb
    *ptep = 0;                          //(5) clear second page table entry
}
```
#### 2 Page的全局变量  

Page的全局变量为pages。
pages变量对应的是每一个页的虚拟地址，而页表项和页目录项都指向一个页，它们保存的是页的物理地址，通过pte2page、pde2page可以将pte、pte中保存的页映射到page中的虚拟地址对应的页。
#### 3 修改lab2  

需要修改pages变量的地址，如果把它的虚拟地址映射到0处就实现了page的虚拟地址等于物理地址。
