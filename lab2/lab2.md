# Lab2

## 练习1：理解first-fit连续物理内存分配算法

### 结构体介绍

```c
struct list_entry
{
    struct list_entry *prev, *next;
};
```

结构体`list_entry`，内有两个`list_entry`指针，这两个指针，它的`typedef`为`list_entry_t`。

```c
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // number of free pages in this free list
} free_area_t;
```

结构体`free_area_t`，双向链表结构。
其中`free_list`是一个`list_entry`结构的双向链表变量。
`nr_free`记录当前空闲页的个数。

```c
struct Page {
    int ref;                        // page frame's reference counter
    uint64_t flags;                 // array of flags that describe the status of the page frame
    unsigned int property;          // the num of free block, used in first fit pm manager
    list_entry_t page_link;         // free list link
};
```

结构体`Page`，是物理页的属性结构。
`ref`表示该页被页表引用的计数，即映射到该物理页的虚拟页个数。
`flags`是该物理页的状态标记，有两个标记位，第一个表示是否被保留，如果被保留则为1。第二个表示此页是否是`free`的，如果为`free`，则设置为1，表示该页可以被分配。反之则是已被分配。
`property`表示某个连续内存空闲块的大小，该变量被使用时表示该`Page`为连续内存块的开始地址，即第一页的地址。
`page_link`用以将多个空闲内存块链接到一起的双向链表指针。

### 函数介绍

+ `default_init`函数用于初始化空闲内存块的链表和空闲内存块的数量。
它会调用`list_init`,`list_init`会使用`free_list`作为参数，然后将该参数的两个指针都指向参数本身，作为双向链表的初始化，然后将`nr_free`赋值为0，设置现在可用的页表为0。

+ `default_init_memmap`函数用于初始化一个空闲内存块，初始化空闲页链表。它会遍历给定的内存块，初始化每一个空闲页，然后计算空闲页的总数。
  传入参数为物理页的基地址和物理页个数（该数量应大于0）。首先对每一个物理页进行设置：先判断是否为保留页，不是则将标志位清0，连续空页个数也设置为0。修改物理基地址的`property`为n。设置该页的标志位为1，计算所有空闲页的个数。然后判断`free_list`是否为空，如果为空，则将当前物理基地址的`page_link`加入`free_list`中。如果不为空，则需要遍历已有的`free_list`，然后按照内存的地址将物理基地址插入到`free_list`对应的位置。

+ `default_alloc_pages`函数用于寻找第一个能够满足需求的内存块。它会遍历空闲内存块的链表，如果找到对应位置，就将它进行截断取出。
  传入需要的页数`n`，如果需要的页数大于已有的可使用的页数`nr_free`，则直接返回`NULL`。从链表的头指针开始遍历，直到再次回到头指针。如果找到第一个能够提供`n`个页的空闲内存块，就中断循环。如果找到了合适的空闲内存块，先将原本的指针删除，然后对该内存块进行页面初始化，生成新的指针指向剩余的内存，并将它接入链表内。重新计算空闲页数并进行赋值，然后将新页基址的标志位置1，表示该页可以进行分配。

+ `default_free_pages`函数用于释放指定数量的页面。它会将被释放的页面重新添加到空闲内存块链表中，并尝试合并相邻的空闲内存块。
  传入参数为物理页基地址`base`和物理页个数`n`，先判断`n`是否大于0，然后遍历从`base`开始的`n`个页，通过这些页的两个标志位判断这些页是否存在保留的页。如果不是保留页，就将标志位置零，并将被引用计数清零（`ref=0`）。遍历结束后，认为这`n`个页无被引用被保留，可以被清空。设置基地址的`property`为n，即有n个页空闲，将基地址的标记为设置为1，表示可以空闲可以使用。重新计算总的空闲页数。然后判断空闲链表是否为空，如果为空，将此次清空的页数放入链表，如果不为空，则将此次空闲块按照内存地址插入空闲链表内。然后查看链表中`base`的前后空闲块，如果前一个和`base`连在一起，则将两个空闲块合并，将列表中指向`base`的指针删除。如果后一个和`base`相连，同理进行合并。在合并后，需要计算新的基地址的可用空闲页并赋值。

+ `default_nr_free_pages`函数返回当前空闲内存块的数量。

### 程序进行内存分配的过程

首先，需要准备好内存管理所需的数据结构。在这里，使用了一个名为`free_area`的全局变量，其中包含了一个空闲内存块的链表`free_list`和空闲内存块的总数`nr_free`。
`default_init`函数用于初始化内存管理的数据结构。它将空闲内存块链表`free_list`初始化为空，并将空闲内存块的总数`nr_free`设置为0。
`default_init_memmap`函数用于初始化一个空闲内存块。它接受一个指向内存块数组的指针`base`和内存块的数量`n`作为参数。在函数中，首先确保`n`大于0。然后，将空闲内存块的总数`nr_free`增加`n`。接下来，如果空闲内存块链表`free_list`为空，将当前内存块添加到链表中。否则，遍历链表找到合适的位置将当前内存块插入链表中。
`default_alloc_pages`函数用于分配指定数量的内存页。它接受一个参数`n`，表示需要分配的内存页数量。在函数中，首先确保`n`大于0。然后，检查空闲内存块的总数`nr_free`是否足够分配所需的内存页数量。如果不足，则返回`NULL`表示内存不足以进行分配。如果足够，则遍历空闲内存块链表`free_list`，找到第一个大小大于等于`n`的空闲块，并将其分配出去。分配后，更新空闲内存块的总数`nr_free`和相应的标志位。
`default_free_pages`函数用于释放已分配的内存页。它接受一个参数`base`，表示要释放的内存页的起始地址。在函数中，首先检查要释放的内存页是否合法。然后，将要释放的内存页重新链接到空闲内存块链表`free_list`中，并根据需要合并相邻的空闲块。最后，更新空闲内存块的总数`nr_free`和相应的标志位。
`default_nr_free_pages`函数用于返回当前系统中的空闲内存页数量。它返回全局变量`nr_free`的值。

### 算法不足和改进

算法还有改进空间，比如当前使用的是线性搜索的方式来寻找合适的空闲块进行分配。这种方式的时间复杂度为O(n)，其中n为空闲块的数量。可以使用更高效的数据结构，如二叉搜索树或哈希表，来加速寻址过程。

## 练习2：实现 Best-Fit 连续物理内存分配算法

### 设计实现过程

Best-Fit 算法的主要思想是为了分配n字节，使用最小的可用空闲块，以致块的尺寸比n大（为了避免分割大空闲块，为了最小化外部碎片产生的尺寸）。基于这个思想，主要对 first-fit 算法的以下部分进行了修改：

```C
if (n > nr_free) {
    return NULL;
}
struct Page *page = NULL;
list_entry_t *le = &free_list;
while ((le = list_next(le)) != &free_list) {
    struct Page *p = le2page(le, page_link);
    if (p->property >= n) {
        page = p;
        break;
    }
}
```

根据上面的代码，基于Best-Fit 算法的主要思想，编写下面的Best-Fit 算法代码：

```
struct Page *page = NULL;
    list_entry_t *le = &free_list;
    size_t min_size = nr_free + 1;
     /*LAB2 EXERCISE 2: YOUR CODE: 2113414、2112117、2113021*/ 
    // 下面的代码是first-fit的部分代码，请修改下面的代码改为best-fit
    // 遍历空闲链表，查找满足需求的空闲页框
    // 如果找到满足需求的页面，记录该页面以及当前找到的最小连续空闲页框数量
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n && min_size > p->property) {
            page = p;
            min_size = p->property;
        }
    }
```

首先创建一个min_size变量，大小是nr_free + 1，方便进行之后的更新。接下来，主要针对满足条件的可用空闲块进行查找。通过while循环，每次查找如果找到的可用空闲块大于需要的，同时min_size也大于这次找到的可用空闲块，则更新这两个量，如果有一个条件不满足，就进行下一轮循环，直到循环结束。最后得到的指针和min_size指向最小的可用空闲块，以致块的尺寸比n大。

上述就是主要的物理内存分配思想。物理内存释放的思想和 first-fit 算法基本一致。

具体代码见编程部分。

### 算法不足和改进

Best-Fit 的实现可能需要搜索整个可用分区列表以找到最佳适应项。这可能导致较高的时间复杂度。一些改进可以通过使用更复杂的数据结构，如二叉树或其他高效的查找结构，来提高搜索速度。还可以考虑在系统启动时或运行时进行一些预分配，以减少后续的内存碎片问题。