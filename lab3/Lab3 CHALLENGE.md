# Lab3 CHALLENGE报告
**组员：梁锦洲，赵乐**

## 设计思路
LRU算法是根据时间局部性来选择被置换出的页面。根据这个规则及实验框架，按照以下思路设计：  
- 双向链表按照页面被访问的顺序排列，最近访问的页面在链表的头部，最久未访问的页面在链表的尾部。  
- 当一个页面被访问时，如果它已经在内存中，就将其节点移动到链表头部。  
- 当需要替换页面时，移除链表尾部的节点，即最久未访问的页面。

## 源代码
### swap_lru.h
```c
#ifndef __KERN_MM_SWAP_LRU_H__
#define __KERN_MM_SWAP_LRU_H__

#include <swap.h>
extern struct swap_manager swap_manager_lru;

#endif
```
### swap_lru.c
```c
static list_entry_t pra_list_head, *curr_ptr, *curr;

static int
_lru_init_mm(struct mm_struct *mm)
{     
     /*LAB3 CHALLENGE: 2212221 2113874*/ 
     // 初始化pra_list_head为空链表
     list_init(&pra_list_head);
     // 初始化当前指针curr_ptr指向pra_list_head，表示当前页面替换位置为链表头
     curr_ptr = &pra_list_head;
     curr = &pra_list_head;
     // 将mm的私有成员指针指向pra_list_head，用于后续的页面替换算法操作
     mm->sm_priv = &pra_list_head;
     return 0;
}
```
```c
static int
_lru_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && curr_ptr != NULL);
    //record the page access situlation
    /*LAB3 CHALLENGE: 2212221 2113874*/ 
    // link the most recent arrival page at the back of the pra_list_head qeueue.
    // 如果页面在链表头部，则不做处理
    if(list_prev(&pra_list_head) == entry){
    	return 0;
    }
    //遍历链表判断页面是否在链表中
    while(list_next(curr) != &pra_list_head){
    	if(curr == entry){
    	list_del(curr);
    	// 将页面移动到链表的头部，表示它是最近访问的
    	list_add_before(&(pra_list_head), entry);
    	return 0;
    	}
    }
    // 将页面移动到链表的头部，表示它是最近访问的
    list_add_before(&(pra_list_head), entry);
    return 0;
}
```
以上函数操作是在遍历已映射的页面链表，首先判断是否以及存在，若存在则删除页面位置，将其移到链表头；若不存在直接加到链表头，表示为最近访问的页面。
```c
static int
_lru_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
     assert(head != NULL);
     assert(in_tick==0);
    /*LAB3 CHALLENGE: 2212221 2113874*/
    while (1) {
	if (curr_ptr == head){
	curr_ptr = list_next(curr_ptr);
	cprintf("curr_ptr %p\n", (void*)curr_ptr);
	}
        // 获取当前页面对应的Page结构指针
        struct Page *page = le2page(curr_ptr, pra_page_link);
        cprintf("curr_ptr %p\n", (void*)curr_ptr);
        // 如果当前页面是链表的最后一个，即最近最久未访问的页面
        if (list_next(curr_ptr) == head) {
            // 将该页面从页面链表中删除
            list_del(&(page->pra_page_link));
            // 将该页面指针赋值给ptr_page作为换出页面
            *ptr_page = page;
            return 0;
        }
        // 将curr_ptr指向下一个，以便在下一次迭代中检查
        curr_ptr = list_next(curr_ptr);
    }
    return 0;
}
```
以上函数实现了置换操作，实际上由于在插入链表时已经做了链表更新操作，此时最近被访问的页面在链表头，最不常访问的在链表尾部，直接删除尾部元素即可。
```c
//测试函数
static int
_lru_check_swap(void) {
#ifdef ucore_test
    cprintf("this is lru test!!!");
    //no write
    int score = 0, totalscore = 5;
    cprintf("%d\n", &score);
    ++ score; cprintf("grading %d/%d points", score, totalscore);
    *(unsigned char *)0x3000 = 0x0c;
    assert(pgfault_num==4);
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==4);
    *(unsigned char *)0x4000 = 0x0d;
    assert(pgfault_num==4);
    *(unsigned char *)0x2000 = 0x0b;
    ++ score; cprintf("grading %d/%d points", score, totalscore);
    assert(pgfault_num==4);
    *(unsigned char *)0x5000 = 0x0e;
    assert(pgfault_num==5);
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==5);
    ++ score; cprintf("grading %d/%d points", score, totalscore);
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==5);
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==5);
    *(unsigned char *)0x3000 = 0x0c;//逐次出现缺页异常
    assert(pgfault_num==6);
    ++ score; cprintf("grading %d/%d points", score, totalscore);
    *(unsigned char *)0x4000 = 0x0d;
    assert(pgfault_num==7);
    *(unsigned char *)0x5000 = 0x0e;
    assert(pgfault_num==8);
    assert(*(unsigned char *)0x1000 == 0x0a);
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==9);
    ++ score; cprintf("grading %d/%d points", score, totalscore);
#else 
    *(unsigned char *)0x3000 = 0x0c;
    assert(pgfault_num==4);
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==4);
    *(unsigned char *)0x4000 = 0x0d;
    assert(pgfault_num==4);
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==4);
    *(unsigned char *)0x5000 = 0x0e;
    assert(pgfault_num==5);
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==5);
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==5);
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==5);
    *(unsigned char *)0x3000 = 0x0c;
    assert(pgfault_num==6);
    *(unsigned char *)0x4000 = 0x0d;
    assert(pgfault_num==7);
    *(unsigned char *)0x5000 = 0x0e;
    assert(pgfault_num==8);
    assert(*(unsigned char *)0x1000 == 0x0a);
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==9);
#endif
    return 0;
}


static int
_lru_init(void)
{
    return 0;
}

static int
_lru_set_unswappable(struct mm_struct *mm, uintptr_t addr)
{
    return 0;
}

static int
_lru_tick_event(struct mm_struct *mm)
{ return 0; }


struct swap_manager swap_manager_lru =
{
     .name            = "lru swap manager",
     .init            = &_lru_init,
     .init_mm         = &_lru_init_mm,
     .tick_event      = &_lru_tick_event,
     .map_swappable   = &_lru_map_swappable,
     .set_unswappable = &_lru_set_unswappable,
     .swap_out_victim = &_lru_swap_out_victim,
     .check_swap      = &_lru_check_swap,
};
```

