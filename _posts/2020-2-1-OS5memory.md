---
title: MIT 6.828 JOS 内存管理详解 操作系统实验5
category: Lab Report
excerpt_separator: <!--more-->
tags: [OS, JOS]
---


本文详解JOS操作系统虚拟内存的结构，以及内存管理单元的结构与实现。
在这次实验中，将探索以下问题：

- 计算机启动时如何开启虚拟内存
- 程序虚拟地址空间的结构
- 如何将内核物理地址与虚拟地址映射
- 访问一个虚拟地址的时候，会发生什么
- 如何管理内存空闲空间
- 如何保护特定的代码和数据
- ……

<!--more-->

# 操作系统lab5： 内存管理

在lab1中，我们看到JOS 物理内存的结构：

 ![address space](/assets/images/report/OS5-2.png) 

最初物理内存只有1MB， 之后扩展到了4GB，这时物理内存的640KB 到1MB之间就成为IO hole，是不可用的，用来分配给外部IO设备，如上图，640KB 到1MB之间被分配给了VGA Display、BIOS ROM以及其他的外部设备，称为IO hole。在JOS中，从0x0到640KB这部分称为 basemem，是可用的， 1MB以上的空间称为extented memory，也是可用的。



为了更有效地管理和使用内存空间，JOS使用了虚拟内存，虚拟内存通过对程序存储地址与真实内存物理地址的解耦，有效解决了内存大小相对于大量用户程序所需空间不足的问题。引入虚拟内存之后，需要解决如何将多个程序分配到物理内存上，以及程序的虚拟地址如何与物理地址映射的问题。JOS通过分页的方式来管理内存和虚拟地址空间 ，将程序地址空间分为固定大小的页，将内存分为同样大小的页框，以页为单位将程序分配到内存物理空间上。页表记录了一个虚拟页对应的物理页框，以及这些页的相关信息，当程序执行中访问一个虚拟地址时，首先要访问它的页表，然后从页表中找到对应的真实地址，再访问真实的物理地址。

在操作系统中，页表的管理、从虚拟地址到物理地址的转换、页面的分配回收以及缓存的管理等等，都是由内存管理单元(MMU)来完成的。内存管理与虚拟内存对用户是不可见的。

在这次实验中，将探索以下问题：

- 计算机启动时如何开启虚拟内存
- 程序虚拟地址空间的结构
- 如何将内核物理地址与虚拟地址映射
- 访问一个虚拟地址的时候，会发生什么
- 如何管理内存空闲空间
- 如何保护特定的代码和数据
- ……




### JOS的虚拟地址空间布局

在lab1中，我们追踪了开机时bootloader加载内核的过程，加载完成后，物理内存的布局为：（图片来自[https://blog-1253119293.cos.ap-beijing.myqcloud.com/](https://blog-1253119293.cos.ap-beijing.myqcloud.com/）)

 <img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/6.828/lab2/lab2_1_physical_memory.png" alt="物理内存分布" style="zoom: 67%;">图片来自https://blog-1253119293.cos.ap-beijing.myqcloud.com/</img>

JOS用了手写的内存映射，将物理地址0x00000000-0x00400000之间4MB的空间映射到了虚拟地址0xf0000000-0xf0400000处。0xf0000000即为在虚拟地址空间中内核部分的起始。

真正开启虚拟内存之后，对于内核和用户程序来说，虚拟地址的布局在`memlayout.h`中定义：

```c
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 *
```

[KERNBASE, 4Gig ] :  这部分映射到物理内存上中断向量表、引导扇区代码、IOhole以及内核代码、数据。在这部分，一个虚拟地址 - KERNBASE就是它的物理地址。内核部分会被同样地映射到每个进程的高地址空间，用户是没有权限访问的。

KERNBASE往下是进程的地址空间，如之前报告中所述，进程地址空间的高处是内核栈，这部分地址是用户模式下不可访问的。

[MMIOBASE, MMIOLIM ] : 这部分空间属于内存映射的IO设备，与IO设备通信要陷入内核完成，因此用户模式也不可访问。

[UVPT, ULIM ] : 从ULIM往下直到UTOP是用户模式下只读的地址。UVPT到ULIM的这部分是当前的页目录，用户可以读取页表知道一个虚拟地址所在的物理页面，但不可操纵页表。

[UPAGES, UVPT] ：这部分对应着`pages`数组在物理内存中存放的位置，用户也可以通过`uvpt[n].pp_ref`来知道某个物理页框是否已经被占用，但也不可操纵。

[0, UTOP] : 这部分才真正是用户模式下可以读写的地址空间，它包括了用户程序的代码段、数据段、堆栈等。


### JOS中的三种地址

JOS中有三种地址： 逻辑地址(virtual address)， 线性地址(linear address),  物理地址(physical address). 逻辑地址是程序编译链接之后变量的符号，实际上，逻辑地址是变量的段内偏移。 线性地址是逻辑地址经过保护模式的段地址变换之后的虚拟地址，线性地址=段首地址+逻辑地址。物理地址则是内存存储单元的编址，它会被直接送到内存的地址线上进行读写。

逻辑地址到线性地址的变换在保护模式下自动完成。如果没有开启页式地址转换（Paging），那么线性地址就是物理地址，如同我们在lab 1中，`mov %eax, cr3`之前看到的一样。如果开启分页，线性地址就会按查询页表的方式转换成物理地址。**后面的实验内容中，我们直接将线性地址称为虚拟地址。**



### JOS的页表结构

页表记录了从虚拟地址到真实物理地址之间的映射，JOS的页表结构、虚拟地址组成定义在`mmu.h`中，它使用的是一个两级页表：

```c
 A linear address 'la' has a three-part structure as follows:

// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |      Index     |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \---------- PGNUM(la) ----------/
```

地址最高10位表示页表目录，中间10位表示页表索引，最后12位表示在一个页面内的偏移，因此，页面总数为$2^{10}$ *$2^{10}$=1024x1024，页面大小为2^12=4096字节。

JOS使用两级页表，将全部的地址空间分为了一个页表目录和1024个页表，由于页表有1024个条目，每个条目的长度是4字节，则**每个页表刚好就占一个页面**，因此页表的地址只需要20位来区分。所以，在页面目录中，我们只需要20位来存放索引对应页表所在的物理地址，剩下的12位用来存放各种标志。页面目录中也含有1024条目，所以**页面目录也只占一个页面**。所以在用户的虚拟地址中，只需要存放一份页面目录的镜像，就可以让用户程序访问到页表，而不需要将所有1024个页表都映射到用户的虚拟地址空间。

一个页表目录条目（或者页表条目，一样的）的结构为：

 ![pgdir entry](http://plafer.github.io/img/x86_pgdir.png) 

一个目录条目的前20位记录了一个页表的物理地址。访问一个虚拟地址时，首先根据前10位目录索引从page directory上找到相应的条目，取出前20位作为页表的物理地址，然后访问该页表，根据10位的页表索引找到页表上对应的物理地址（也是前20位，与PGSIZE对齐），这个20位的物理地址加上offset就得到了物理地址。如图：

 ![x86 addr format](http://plafer.github.io/img/x86-addr-format.gif) 

 ![x86 addr translation](http://plafer.github.io/img/x86-addr-translation.gif) 

条目剩下的低12位用来存放各种标志，来表示一个页表/页面的状态，所有的状态在`mmu.h`中定义：

```c
// Page table/directory entry flags.
#define PTE_P		0x001	// Present
#define PTE_W		0x002	// Writeable
#define PTE_U		0x004	// User
#define PTE_PWT		0x008	// Write-Through
#define PTE_PCD		0x010	// Cache-Disable
#define PTE_A		0x020	// Accessed
#define PTE_D		0x040	// Dirty
#define PTE_PS		0x080	// Page Size
#define PTE_G		0x100	// Global
```

其中，Present位是用来判断对应的页表或者条目是否存在物理内存中，如果存在则为1. 在后面的代码中，我们判断一个虚拟页是否与一个物理页框映射，即是否驻留在内存时，就可以通过 `entry & PTE_P`来判断。



## TODO 1: Physical Page Management 代码阅读 

### mem_init()

`mem_init()`在内核刚启动时调用，它的任务是在开机之后，设置好分页系统，并完成内核部分虚拟地址与物理地址的映射。目前只完成了一部分，它需要初始化的变量如下：

```c
// These variables are set in mem_init()
pde_t *kern_pgdir;		// Kernel's initial page directory
struct PageInfo *pages;		// Physical page state array
static struct PageInfo *page_free_list;	// Free list of physical pages

```

`kern_pgdir`是页表目录。

`PageInfo`是一个用来描述物理页框的结构体，它定义在`memlayout.h`中, 由一个指向下一个节点的指针，和引用位构成。每一个物理页框都对应着一个PageInfo结构，引用位表示该页框是否已经被占用。`pages`数组记录了所有物理页框（总共`npages`个）的信息，而为了分配页面时更快地找到一个空的页框，JOS还维护了`page_free_list`链表，动态地保存所有空闲的页框。当需要分配页面时，从`page_free_list`的头部指针获取第一个空闲页框，然后将头部指针后移；当有新的空闲页面时，将这个新页面的指针添加到`page_free_list`中。

```c
struct PageInfo {
	// Next page on the free list.
	struct PageInfo *pp_link;

	// pp_ref is the count of pointers (usually in page table entries)
	// to this page, for pages allocated using page_alloc.
	// Pages allocated at boot time using pmap.c's
	// boot_alloc do not have valid reference count fields.

	uint16_t pp_ref;
};
```



`mem_init`具体实现如下：

```c
// Set up a two-level page table:
//    kern_pgdir is its linear (virtual) address of the root
//
// This function only sets up the kernel part of the address space
// (ie. addresses >= UTOP).  The user part of the address space
// will be set up later.
//
// From UTOP to ULIM, the user is allowed to read but not write.
// Above ULIM the user cannot read or write.
void
mem_init(void)
{
	uint32_t cr0;
	size_t n;

	// Find out how much memory the machine has (npages & npages_basemem).
	i386_detect_memory();     //检测机器有多少物理内存。

	// Remove this line when you're ready to test this function.
	panic("mem_init: This function is not finished\n");

	//////////////////////////////////////////////////////////////////////
	// create initial page directory.
	kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
	memset(kern_pgdir, 0, PGSIZE);

	//////////////////////////////////////////////////////////////////////
	// Recursively insert PD in itself as a page table, to form
	// a virtual page table at virtual address UVPT.
	// (For now, you don't have understand the greater purpose of the
	// following line.)

	// Permissions: kernel R, user R
	kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;

	//////////////////////////////////////////////////////////////////////
	// Allocate an array of npages 'struct PageInfo's and store it in 'pages'.
	// The kernel uses this array to keep track of physical pages: for
	// each physical page, there is a corresponding struct PageInfo in this
	// array.  'npages' is the number of physical pages in memory.  Use memset
	// to initialize all fields of each struct PageInfo to 0.
	pages = (struct PageInfo *) boot_alloc(npages * sizeof(struct PageInfo));
	memset(pages, 0, npages * sizeof(struct PageInfo));


	//////////////////////////////////////////////////////////////////////
	// Now that we've allocated the initial kernel data structures, we set
	// up the list of free physical pages. Once we've done so, all further
	// memory management will go through the page_* functions. In
	// particular, we can now map memory using boot_map_region
	// or page_insert
	page_init();

	check_page_free_list(1);
	check_page_alloc();
	check_page();
```

在JOS开机的时候，我们会看到一句输出：

![image-20191127111815464](/assets/images/report/OS5-1.png)

给出了物理内存的可用空间，`base`是底部的basemem的大小（640K），`extended`是extended memory的大小，是1MB以上的可用空间。检测是在函数`i386_detect_memory()`中完成的：

```c
static void
i386_detect_memory(void)
{
	size_t basemem, extmem, ext16mem, totalmem;

	// Use CMOS calls to measure available base & extended memory.
	// (CMOS calls return results in kilobytes.)
	basemem = nvram_read(NVRAM_BASELO);
	extmem = nvram_read(NVRAM_EXTLO);
	ext16mem = nvram_read(NVRAM_EXT16LO) * 64;

	// Calculate the number of physical pages available in both base
	// and extended memory.
	if (ext16mem)
		totalmem = 16 * 1024 + ext16mem;
	else if (extmem)
		totalmem = 1 * 1024 + extmem;
	else
		totalmem = basemem;

	npages = totalmem / (PGSIZE / 1024);
	npages_basemem = basemem / (PGSIZE / 1024);

	cprintf("Physical memory: %uK available, base = %uK, extended = %uK\n",
		totalmem, basemem, totalmem - basemem);
}
```

注意到读取basemem、extmem和ext16mem的大小使用了函数`nvram_read`。`nvram_read`实际上又调用了`mc146818_read`函数，这个函数通过IO端口0x70与0x71从实时时钟RTC中读取数据。RTC使用芯片mc146818，在系统电源关闭时，RTC仍保持工作，维护系统的日期和时间，当系统启动时，就从RTC中读取日期时间的基准值。时钟和这里的物理内存其实没有关系，但mc146818芯片中带有一个非易失性的RAM，也就是non-volatile-ram（nvram），系统的物理内存basemem和extmem的大小，都存放在这个芯片上，这样可以保证系统电源关闭时，这些信息不会被擦除。

`i386_detect_memory`通过`nvram_read`从mc146818芯片中读取出basemem和extmem大小（以KB为单位），然后根据它们计算出内存总的可用空间以及总的页面数`npages`,`npages_basemem`。PGSIZE定义在`mmu.h`中，为4096字节。



检测出可用内存大小之后，`mem_init`开始设置内核的页表。首先调用`boot_alloc`在物理内存中分配内核的页表。

### boot_alloc()

boot_alloc()只会在JOS初始化虚拟内存之前被调用一次，之后分配页面的时候都只会使用`page_allocator()`. 之所以要写一个单独的`boot_alloc`是因为： 在启动时需要将内核的物理地址映射到虚拟地址，这种映射需要通过访问内核的页表来实现，创建页表涉及到分配页表所在的页面，可是分配页面又是在虚拟内存设置好才可以做到。所以，JOS使用了一个单独的`boot_alloc`，将需要分配的页面映射到一些固定的虚拟地址，并返回所分配的内容的起始虚拟地址。

```c
static void *
boot_alloc(uint32_t n)   //n表示需要分配的字节数
{
	static char *nextfree;	// virtual address of next byte of free memory
	char *result;    //result 用来保存分配的一片虚拟地址的起始地址

	// Initialize nextfree if this is the first time.
	// 'end' is a magic symbol automatically generated by the linker,
	// which points to the end of the kernel's bss segment:
	// the first virtual address that the linker did *not* assign
	// to any kernel code or global variables.
	if (!nextfree) {                     
		extern char end[];
		nextfree = ROUNDUP((char *) end, PGSIZE);
	}

	// Allocate a chunk large enough to hold 'n' bytes, then update
	// nextfree.  Make sure nextfree is kept aligned
	// to a multiple of PGSIZE.
	result = nextfree;
	nextfree = ROUNDUP(nextfree+n, PGSIZE);
	if((uint32_t)nextfree - KERNBASE > (npages*PGSIZE))
		panic("Out of memory!\n");
	return result;
}
```

`nextfree`表示下一个未用的虚拟地址, 是一个静态变量。当`mem_init`调用`kern_pgdir = (pde_t *) boot_alloc(PGSIZE)`时，`nextfree`还未初始化，它会被初始化在内核.bss段的结束，并与页面大小4096B 对齐。

这里使用了函数`ROUNDUP(char* a, uint32_t n)`，它同`ROUNDDOWN`一起在`type.h`中定义，分别是求 a/n的向上和向下取整，因此，ROUNDUP可以用来将地址`a`与`n`对齐。

用`result`保存`nextfree`作为起始地址后，将`nextfree`向后移动`n`个字节（也要和PGSIZE对齐），作为下次分配的起始地址。

在分配时，还要检查是否是一个合法的虚拟地址。 从上面JOS虚拟地址空间布局，我们知道`nextfree-KERNBASE`实际上就是`nextfree`的物理地址，这个物理地址不可超过内存可用的物理空间大小(页框总数`npages`*页面大小`PGSIZE`)，否则抛出错误。



回到`mem_init`中，

```c
	kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
	memset(kern_pgdir, 0, PGSIZE);
```

将内核页表分配在虚拟地址空间中内核.bss段的后面，然后用`memset`将页表初始化为全0.

```c
kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
```

在虚拟地址布局中，我们看到从[UVPT, ULIM]（大小为一个页表大小）这一段是用户和内核都可读的页表目录的复制，那么就要将虚拟地址`UVPT`映射到`kern_pgdir`的真实物理地址上去， 而要完成这种映射，就是要在`kern_pgdir`页表目录中，对应虚拟地址`UVPT`的条目中，将页表地址改为`kern_pgdir`的物理地址。 这样，当用户或内核访问UVPT与ULIM之间的虚拟地址时，就要首先访问`kern_pgdir`，查找`uvpt`对应的物理地址，然后发现该物理地址就是`kern_pgdir`所在的物理地址。

`PDX(la)`在`mmu.h`中定义，计算la对应的页表目录索引。`PADDR`是将传入的虚拟地址减去`KERNBASE`，得到物理地址。 PTE_U表示用户有权限（则内核也有权限），PTE_P表示物理地址存在。这个语句将页表目录中`UVPT`起始的页面对应的条目置为页表目录的物理页面地址， 并设置用户可读。



接下来，初始化`pages`数组，调用`boot_alloc`将它分配在内核页表目录`kern_pgdir`之后，并用`memset`初始化为全0.

```
	pages = (struct PageInfo *) boot_alloc(npages * sizeof(struct PageInfo));
	memset(pages, 0, npages * sizeof(struct PageInfo));
```

`pages`数组用来一对一地记录每一个物理页框是否被占用，可以通过`pages[i].pp_ref`来判断。

之后，`mem_init`调用`page_init()`.



### page_init()

我们已经初始化了页表目录和`pages`数组，则`page_init()`的任务就是初始化`page_free_list`，记录哪一些物理页框是空闲的，并设置`pages`中每一个页框的结构。

`page_free_list`实际上只是一个PageInfo结构体，此结构体中包含了指向下一个的指针，也就是下一个空闲的页框。所以，我们可以遍历`pages`数组，将那些已经分配出去的页框`pp_ref`置为1，将空闲的页框`pp_ref`置为0，并让`page_free_list`指向这个页框，从而将它插入空闲页框链表。

根据注释提示，第一个物理页框已经分配给中断向量表和其他的BIOS结构，basemem中剩下的部分([PGSIZE, npages_basemem*PGSIZE])还是空闲的。

extmem中，我们刚才已经分配了一部分给内核，要知道分配了多少，我们可以调用`boot_alloc(0)`来获取分配完`pages`之后，下一个可用的虚拟地址，将它减去KERNBASE得到物理地址，再除以PGSIZE就得到分配出去的页框数目。

IOhole部分，也就是从640KB到1MB之间的96个页面，都分配给了外部IO设备。IOhole和extmem是连续的，因此page_init的实现如下：

```c
void
page_init(void)
{
	//  1) Mark physical page 0 as in use.
	//     This way we preserve the real-mode IDT and BIOS structures
	//     in case we ever need them.  (Currently we don't, but...)
	//  2) The rest of base memory, [PGSIZE, npages_basemem * PGSIZE)
	//     is free.
	//  3) Then comes the IO hole [IOPHYSMEM, EXTPHYSMEM), which must
	//     never be allocated.
	//  4) Then extended memory [EXTPHYSMEM, ...).
	//     Some of it is in use, some is free. Where is the kernel
	//     in physical memory?  Which pages are already in use for
	//     page tables and other data structures?
	//
	size_t i;
	page_free_list = NULL;

	//num_alloc：在extmem区域已经被占用的页的个数
	int num_alloc = ((uint32_t)boot_alloc(0) - KERNBASE) / PGSIZE;
	//num_iohole：在io hole区域占用的页数
	int num_iohole = 96;

	for(i=0; i<npages; i++)
	{
	    if(i==0)
	    {
	        pages[i].pp_ref = 1;    //第一个页面已经被占用
	    }    
	    else if(i >= npages_basemem && i < npages_basemem + num_iohole + num_alloc)
	    {   //从IOhole到extmem中已分配的部分
	        pages[i].pp_ref = 1;
	    }
	    else
	    {  //页面空闲，将pp_ref=0，将pp_link置为下一个空闲页面的指针，这样，当该页面被分配出去的时候，我们可以让page_free_list指向pp_link来将它从链表中移出。
	        pages[i].pp_ref = 0;
	        pages[i].pp_link = page_free_list;
	        page_free_list = &pages[i];   //将pages[i]插入到page_free_list的头部。
	    }
	}
}
```

### page_alloc()

在`pages`设置好之后，分配页面就不可以再调用`boot_alloc`了，必须调用`page_alloc`通过在`page_free_link`查找空页框的方式来分配页面。`page_alloc`分配一个页面，返回该页面的PageInfo指针。它同时接收一个参数`alloc_flags`, 如果它的值为1（ALLOC_ZERO), 就将分配到的物理页面设置为全0。如果没有可用页框，则返回空指针NULL。所以该函数的步骤为：

1. 从page_free_list中取出一个空闲页框的PageInfo结构体。
2. 将这个页框从page_free_list中移去，并将链表头指针指向下一个空闲页框。
3. 修改取出的PageInfo相关信息，如果有ALLOC_ZERO, 修改该内存页。

```c
struct PageInfo *
page_alloc(int alloc_flags)
{
    struct PageInfo *result;   
    if (page_free_list == NULL)     //没有空闲页框，返回NULL。
        return NULL;

      result= page_free_list;       //将第一个空闲页框分配出去
      page_free_list = result->pp_link;    //让page_free_list指向该页框的pp_link,也就是链表上的下一个空闲页框，从而将队首pop出去。
      result->pp_link = NULL;       

    if (alloc_flags & ALLOC_ZERO)
        memset(page2kva(result), 0, PGSIZE);  //用memset将该页面设置为全0.

      return result;
}
```

其中，`page2kva()`函数是将传进去的`result`加上KERNBASE, 得到result的物理地址。 `memset`将`result`对应的物理地址开始，一个页面大小的物理内存设置为0.



### page_free()

这个函数将一个被分配的页框归还，只有当该页框的引用位pp_ref为0时，才可以调用：

```c
void
page_free(struct PageInfo *pp)
{
    // Fill this function in
    // Hint: You may want to panic if pp->pp_ref is nonzero or
    // pp->pp_link is not NULL.
      assert(pp->pp_ref == 0);
      assert(pp->pp_link == NULL);

      pp->pp_link = page_free_list;
      page_free_list = pp;
}
```

`assert`是断言，用来判断条件是否满足，否则发出panic错误。当页框在`page_alloc`中被分配出去时，会将pp_link设置为NULL，而页框不再被使用时，pp_ref会置回0，只有这两个条件满足才可以调用page_free.

`page_free`只要简单地将页框插入到`page_free_list`的表头即可，为此，将pp_link指向现在的链表头部：page_free_list, 然后将头部指针指向该页框（pp).



## TODO 2:  Virtual Memory  

参考`pgdir_walk`, `boot_map_region` 和`page_insert`函数，实现`page_lookup`和`page_remove`。首先阅读这三个函数的代码，为了方便解释代码，先看一下`mmu.h`中提供的一些宏，以及`pmap.h`中提供的功能函数。

### 宏与功能函数

在`types.h`中定义了与内存管理相关的类型：

```c
typedef uint32_t uintptr_t;    //表示虚拟地址
typedef uint32_t physaddr_t;    //表示物理地址

// Page numbers are 32 bits long.
typedef uint32_t ppn_t;   //表示页面编号的类型
```

```c
typedef uint32_t pte_t;   //表示一个页表条目的类型
typedef uint32_t pde_t;   //表示一个页目录中的条目的类型
```

对于一个虚拟地址`va`，如果它在`KERNBASE`以上，说明它是一个内核的虚拟地址，而内核部分是始终驻留在内存中的，我们可以使用`pmap.h`中定义的`PADDR(va)`直接将其减去`KERNBASE`得到物理地址。如果它不在`KERNBASE`上，那么就要通过MMU访问页表来将它转换成物理地址。相应的，如果是一个内核的物理地址`pa`, 才可以使用`KADDR`将它加上`KERNBASE`得到虚拟地址。

在`mmu.h`中，定义了一些宏，方便从一个虚拟地址获得对应的页目录、页表条目信息以及物理地址：

- `PGNUM(la)`: 表示一个虚拟地址的页编号，因为每个页是4096字节，又编号是从0开始顺序编号的，只要将`la`右移12位。
- `PDX(la)`：对应页目录索引
- `PTX(la)`: 对应页表索引
- `PGOFF(la)`: 在页面内的偏移
- `PGADDR(d,t,o)`: 从已知的页目录索引、页表索引和页内偏移还原一个虚拟地址。
- `PTE_ADDR(pte)`: 从一个页目录条目或者一个页表条目中取出它的高20位物理地址部分。 

在`pmap.h`中，函数`page2pa()`实现了给出一个页面，获取这个页面开始处的物理地址； 函数`pa2page()`实现了给出一个与页大小对齐的物理地址，返回它所在的页面的PageInfo。



### pgdir_walk()

这个函数的功能是，给出一个虚拟地址`va`， 访问两级页表，找到它对应的页表条目，返回页表条目的指针。但是，由于页表不是一直都整个驻留在内存中的，所以`va`对应的条目所在的页表页可能还不在内存中，这时，如果`create`为`False`，就返回空指针，否则就要使用`page_alloc()`函数，分配这个页表页。

这个过程是：

- 从虚拟地址`va`得到它的页目录索引
- 在页目录上根据索引找到对应条目
- 判断该条目的`present`位是否为1， 如果置位，说明对应的页表在内存中，否则不在
  - 如果create置位，要在内存中为这个页表分配一个页框，使用`page_alloc()`.
    - 如果分配不成功，只能返回NULL。
    - 分配成功，要将这个页表页的引用数pp_ref 加上1， 因为我们现在正要从页表上查`va`对应的条目。并且，还要在页目录中，记录这个页表的物理地址，设置`present`为1，并设置权限为用户可读写。
  - `create`为0，返回NULL。
- 找到了页表后，计算`va`对应的页表索引
- 获取页表上该条目，返回条目的地址。（**这里所说的地址是该条目的虚拟地址**）

```c
pte_t * pgdir_walk(pde_t *pgdir, const void * va, int create)
{
      unsigned int page_off;    //页偏移
      pte_t * page_base = NULL;    //页表页的基址（虚拟地址）
      struct PageInfo* new_page = NULL;    //可能需要分配页表页
      
      unsigned int dic_off = PDX(va);    //用PDX获取va的页目录索引
      pde_t * dic_entry_ptr = pgdir + dic_off;    //页目录中对应的条目的虚拟地址，等于页目录的起始地址+条目在页目录中的索引号

      if(!(*dic_entry_ptr & PTE_P))    //*dic_entry_ptr获取目录条目中的内容，与PTE_P 判断present是否置位，也就是该页表是否在内存中，如果不在：
      {
            if(create)   //在内存中分配该页表页
            {
                   new_page = page_alloc(1);   //分配一个页表页，参数为1，表示该片内存被初始化为全0（之后会将该页表从磁盘读取到这片内存，这不是MMU的工作）
                   if(new_page == NULL) return NULL;  //分配失败
                   new_page->pp_ref++;      //分配成功，则该页面的引用数要增加1.
                   *dic_entry_ptr = (page2pa(new_page) | PTE_P | PTE_W | PTE_U);
// 使用page2pa(newpage)获得该页的起始物理地址，将它存放在对应的页目录条目中，并且置present为1， 更改权限为用户可以读写。
            }
           else
               return NULL;      
      }  
     //用PTX(va)宏，获取va对应的页表索引，这就是要返回的条目在页表中的偏移。
      page_off = PTX(va);
    //page_base用来表示该页表页所在的虚拟地址。要获得此虚拟地址，首先要从页目录表上获得该页表页的物理地址，再用KADDR转换成虚拟地址。
      page_base = KADDR(PTE_ADDR(*dic_entry_ptr));
      return &page_base[page_off];
}
```

最后三行代码最为关键，经过上面的判断和分配页表页，现在该页表页的物理地址已经存放在页目录对应的条目中了， 用`PTE_ADDR(*dic_entry_ptr)`就可以从条目中取出该页表页的物理地址。用`KADDR`可以将物理地址转换为页目录的虚拟地址，这时，其实将`page_base+page_off`就可以得到该条目的虚拟地址了。因此，最后`return &page_base[page_off]`也可以替换为`return page_base+page_off`。

### boot_map_region()

`boot_map_region`的功能是，将虚拟地址[va, va+size]映射到物理地址[pa, pa+size]上，意思就是在页表中[va, va+size]对应的条目中设置物理地址为[pa, pa+size]。 这里va, pa,size都是保证与页面大小对齐的，size的单位是页。`perm`参数给出了这块内存空间的权限。

这个函数是用来“静态“地映射`UTOP`以上的用户只读空间的。过程是：

- 遍历从[va, va+size]的每一个虚拟页，使用`pgdir_walk`找到它在页表中的对应条目
- 将该条目的内容设置为 [pa , pa+size ] | `perm` |PTE_P. 表示`present`置位，这些页面存在于内存中，并设置了权限。

```c
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
    int nadd;
    pte_t *entry = NULL;
    for(nadd = 0; nadd < size; nadd += PGSIZE)
    {
        entry = pgdir_walk(pgdir,(void *)va, 1);    //Get the table entry of this page.
        *entry = (pa | perm | PTE_P);    
        
        
        pa += PGSIZE;
        va += PGSIZE;
        
    }
}
```

### page_insert()

这个函数是将一个物理页框`pp`映射到虚拟地址`va`上。要考虑两种情况，一是该va已经映射到了其他的物理页框上，这时就要接触这个映射关系； 另一种是该va本来就映射到`pp`上了。过程是：

- 调用`pgdir_walk`得到`va`在页表上的条目。
- 如果找不到该条目，说明内存不足，返回错误码 `-E_NO_MEM`。
- 要先让`pp->pp_ref++`，之后解释原因。
- 如果该条目已经存在，说明`va`本来已经映射到一个物理页框上，
  - `tlb_invalidate`使该`va`对应的页表条目失效，这样，才不会使进程在这个过程中访问到不正确的物理地址。**这是因为进程会缓存它用到的页表，快速访问页表时，它先访问缓存中是否有该页表，如果没有，才从页目录去找**。
  - `page_remove`解除`va`和原来物理页框的映射。
- 无论之前该条目是否存在，现在`va`已经不与任何物理页框绑定，将条目内容设置为`pp`的物理地址，设置`present`为1，并设置权限为`perm`

```c
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
  
    pte_t *entry = NULL;
    entry =  pgdir_walk(pgdir, va, 1);    //Get the mapping page of this address va.
    if(entry == NULL) return -E_NO_MEM;

    pp->pp_ref++;
    if((*entry) & PTE_P)             //If this virtual address is already mapped.
    {
        tlb_invalidate(pgdir, va);
        page_remove(pgdir, va);
    }
    *entry = (page2pa(pp) | perm | PTE_P);
    pgdir[PDX(va)] |= perm;                  //Remember this step!
        
    return 0;
}
```

在这个函数中，只用`(*entry) & PTE_P`判断页表条目是否存在，来判断`va`是否已经有映射关系，但没有区分`va`是否与`pp`映射。如果我们将`pp->pp_ref++`移到if块后面，那么当`va`已经与`pp`映射时，`pp`原来的引用数有可能为0，在`if`中，我们会不加判断地调用`page_remove`，然后因为引用数为0，直接调用`page_free`将它释放掉。 既然释放了，它就会处在空闲链表`page_free_list`中，`pp_ref`应该保持为0. 我们之后再用`pp->pp_ref++`时，就会让空闲链表管理出错，下一次分配页框时，可能在空闲链表中找到这个`pp`，但它是不可用的。

****

接下来我们实现`page_lookup`和`page_remove`函数。

### **page_lookup()**

这个函数的功能是给出虚拟地址`va`，找到它映射到的物理页框。如果传入的`pte_store`不为NULL的话，就将该虚拟地址对应的页表条目指针存放到`pte_store`中。实现过程是：

- 调用`pgdir_walk`找到va对应的条目，这里`create`应该设置为0，即若va所在的页表页不在内存，我们也不分配它。
- 如果返回的是NULL，表示va所在的页表页不在内存，即`va`现在没有映射到物理页框，返回NULL。
- 如果`va`对应的页表页在内存中，但是条目的`present`位为0，说明`va`没有映射到物理页框:
  - 判断`pte_store`，如果不为NULL，将条目存放在`pte_store`中
  - 返回NULL
- 用`PTE_ADDR`从条目上获得`va`对应的物理地址
- 用`pa2page`获取物理地址对应页框的PageInfo
- 如果`pte_store`非空，将`pte`保存，最后返回PageInfo

```c
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	pte_t *pte = NULL;
	struct PageInfo *page;

    pte = pgdir_walk(pgdir, va, 0);     //不存在，不分配
    if(pte==NULL)            //页表页不存在
        return NULL;
    
    if(pte_store)      
        *pte_store = pte;
    if(!(*pte & PTE_P)) //页表页存在，但是va并不与一个物理页框映射
        return NULL;
    
	page = pa2page(PTE_ADDR(*pte));
	return page;
}

```



### page_remove()

这个函数的功能是解除`va`与它对应的物理页框之间的映射关系。这不一定说明该物理页框已经空闲，可以回到空闲链表中被分配。因为该页框的引用数并不一定为0，例如在共享内存时，有可能不同进程会共享一部分物理内存，不同的虚拟地址会映射到同一个物理页框上。因此，这个函数的实现过程应该是：

- 调用`page_lookup`查找`va`对应的物理页，并保存其页表条目的地址在`&pte`中。
- 如果该`va`有映射到某个物理页框，解除映射：
  - 调用`page_decref`，这里面做的是，将pp_ref减去1，如果等于0，可以调用`page_free`释放空间，将页框归还。
  - 因为使用快速页表访问时，进程可能缓存了最近使用过的页表条目，所以要调用`tlb_invalidate`让这条`va`的缓存条目无效，否则进程会优先访问缓存中的条目，进而访问到非法的物理地址。
  - 将`va`对应的条目内容清0.

```c
void
page_remove(pde_t *pgdir, void *va)
{
    pte_t *pte;
	struct PageInfo *page = page_lookup(pgdir, va, &pte);
	
	if(page){
		page_decref(page);
		tlb_invalidate(pgdir, va);   //使TLB中可能缓存的这条页表条目无效
		*pte=0;
	}
}
```



### 检查

`pmap.c`中实现了几个函数对代码进行了检查。这些检查函数在`mem_init`中被调用，其中`check_page()`是检查分页的基础功能是否已经实现好，包括`page_alloc()`, `page_insert()`, `page_remove()`，`page_lookup()`, `pgdir_walk()` 以及`page_free()`。

重新编译并启动`qemu`，看到控制台输出：


" check_page() succeeded!" 表明实现是正确的。



***

## TODO 3: Kernel Address Space

###完成mem_init()

上面的函数已经完成了分页机制，页表也已经创建好。现在，我们就可以通过修改页表上的条目，完成`UTOP`以上用户不可操作的空间与物理地址的映射。

首先，将[UPAGES, UVPT]这部分虚拟地址映射到`pages`数组上，权限设置为内核与用户只读，相当于为用户保留了物理页框信息的拷贝。这样，虚拟地址空间上实际有两份`pages`，一部分在KERNBASE以上，用户不可见，内核可读写，这一份就是在`mem_init`刚开始分配的`pages`；另一份是为用户准备的只读拷贝，两者通过页表映射到同一片物理内存上，但这份拷贝设置的权限是只读，所以用户不能对这部分虚拟地址的内容进行操作；又因为用户不可访问KERNBASE上面的虚拟地址（在用户访问虚拟地址时，内核会判断虚拟地址是否超出`ULIM`），所以用户不可读写`pages`。这样就实现了对`pages`数组的保护。

这个映射用

```c
boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages), PTE_U);
```

来实现。



然后要完成内核栈的映射。从[KSTACKTOP - PTSIZE, KSTACKTOP]这部分都属于内核栈， 但被分为两个部分，上面[KSTACKTOP-KSTKSIZE, KSTACKTOP]是真正的内核栈，与某些物理页框映射，而[KSTACKTOP - PTSIZE, KSTACKTOP-KSTKSIZE ]不与物理地址映射，只是用来防止内核栈向下增长的时候发生溢出，然后覆盖了Memory-mapped IO部分，称为保护页。如果内核栈溢出，它就会发现物理地址不存在，抛出错误。

```c
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
```

内核栈是内核可读写，但用户不可见的，这些页面的权限要被设置为`PTE_W`。调用`boot_map_region`, 将`KSTACKTOP-KSTKIZE`开始到`KSTACKTOP`的虚拟地址映射到`bootstack`的物理地址上，大小如上面结构所示，为`KSTKSIZE`，但要注意使用`ROUNDUP`与页面大小对齐：

```c
boot_map_region(kern_pgdir, KSTACKTOP-KSTKSIZE, ROUNDUP(KSTKSIZE, PGSIZE), PADDR(bootstack), PTE_W);
```



将整一个[KERNBASE, 2^32）的整个内核地址空间映射到内存的[0, 2^32-KERNBASE)，权限是内核可修改但用户不可见。我们知道KERNBASE的虚拟地址是0xf0000000, 而整个虚拟空间的大小是2^32也就是4G，所以内核的大小总共是256MB=0X10000000，这已经是与页面大小对齐的。 内存将一直有256MB的空间被内核占用。调用`boot_map_region`实现如下：

```c
boot_map_region(kern_pgdir, KERNBASE, 0x10000000, 0, PTE_W);
```

### 检验

重新编译，启动QEMU，所有的检查都已经通过，说明分页机制与内核的分配已经正确实现：




****

## 总结

### 分页机制建立和开启全过程：

完整地阅读`mem_init`，

```c
void
mem_init(void)
{
	uint32_t cr0;
	size_t n;

	i386_detect_memory();
	kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
	memset(kern_pgdir, 0, PGSIZE);
    
	kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;

	pages = (struct PageInfo *) boot_alloc(npages * sizeof(struct PageInfo));
	memset(pages, 0, npages * sizeof(struct PageInfo));

	page_init();
	boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages), PTE_U);
	boot_map_region(kern_pgdir, KSTACKTOP-KSTKSIZE, ROUNDUP(KSTKSIZE, PGSIZE), PADDR(bootstack), PTE_W);    
	boot_map_region(kern_pgdir, KERNBASE, 0x10000000, 0, PTE_W);

	lcr3(PADDR(kern_pgdir));

	cr0 = rcr0();
	cr0 |= CR0_PE|CR0_PG|CR0_AM|CR0_WP|CR0_NE|CR0_MP;
	cr0 &= ~(CR0_TS|CR0_EM);
	lcr0(cr0);
}
```

内核部分虚拟地址空间的初始化全过程是：

- 检测总共可用的物理内存大小，由`basemem`+`extmem`构成，记录总页数`npages`和低地址页数`npages_basemem`

- 要初始化内核虚拟地址，必须通过页表来映射，但一开始，页表还不存在，页表的虚拟地址也还没有映射。因此，首先要分配页表目录的虚拟地址。这个分配无法通过分页机制来完成，只能通过静态映射：`kern_pgdir = boot_alloc(PGSIZE)`. 页目录分配好后，首先初始化为全0.

- 因为用户进程在访问虚拟地址时，需要访问页表，因此我们需要在ULIM以下为用户准备一份页表的只读拷贝。然而，总共1024份页表的开销较大，其实只要能够访问到页目录，就可以通过页目录访问到页表。 而我们甚至不用真的在内存中存放两份页目录，只需要将拷贝的虚拟地址也指向`kern_pgdir`的物理地址即可。 所以，我们在页目录上让`UVPT`条目指向页目录本身，并设置用户只读：`kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;`, 结构如图：

  -  ![img](https://pdos.csail.mit.edu/6.828/2014/lec/vpt.png) 
  - **这里CR3寄存器存放的是进程页目录的虚拟地址。**操作系统中有**内核页表**和**进程页表**两种页表，进程页表是每个进程独自有一份的，页目录的虚拟地址存放在cr3寄存器中，当进程切换时，会加载页目录虚拟地址到cr3寄存器。进程页表既包含了用户态，也包含了内核态的虚拟地址，内核态的虚拟地址是所有进程都一样的，就是**内核页表的拷贝，它的虚拟地址就在`UVPT`，大小为一个页面**。

- 拷贝好内核页目录后，因为接下来涉及到管理物理内存，需要记录每个页框的信息，这些信息保存在`pages`数组中。同样，需要用`boot_alloc`将pages静态映射到一片虚拟地址。如果打印出`pages`和`kern_pgdir`的虚拟地址，可以看到他们的虚拟地址分别是`f0119000`和`f0118000`， 是在`f0400000`之内的，这两个数据结构已经在物理内存中了。

- 将`pages`初始化为0，然后用`page_init`将已经分配出去的物理页框引用位置为1，并将空闲物理页框添加到`page_free_list`链表中，接下来，内核就可以使用`page_free_list`和`pages`两个数据结构管理物理页框。

- 允许用户读取`pages`来知道内存的占用情况，但为了保护该数据结构，同样要在ULIM以下为它保留一份拷贝，这份拷贝在`UPAGES`到`UVPT`之间，用`boot_map_region`来进行映射。

- boot 的时候， 已经为内核初始化了一个栈，栈顶的界限是`bootstack`，用`boot_map_region`将虚拟地址的KSTACK部分映射到`bootstack`所在的物理内存上，进程进入内核态，并进入公共部分时，实际上运行在这个内核栈上。

- 最后，把整个高地址部分的内核虚拟空间映射到物理内存0地址开始处，实际上是内核部分一直驻留在内存中，且内核的虚拟地址空间被拷贝到每个进程的高地址部分。

- `lcr3(PADDR(kern_pgdir))`是把内核页目录基址放在cr3寄存器中，之后如果开启分页，访问虚拟地址时，就会从cr3加载页目录地址，从而访问页目录。

- 将控制寄存器cr0置位为：`CR0_PE|CR0_PG|CR0_AM|CR0_WP|CR0_NE|CR0_MP ，~(CR0_TS|CR0_EM)`各个字段含义如下：

  - ```c#
    #define CR0_PE		0x00000001	// Protection Enable
    #define CR0_MP		0x00000002	// Monitor coProcessor
    #define CR0_EM		0x00000004	// Emulation
    #define CR0_TS		0x00000008	// Task Switched
    #define CR0_ET		0x00000010	// Extension Type
    #define CR0_NE		0x00000020	// Numeric Errror
    #define CR0_WP		0x00010000	// Write Protect
    #define CR0_AM		0x00040000	// Alignment Mask
    #define CR0_NW		0x20000000	// Not Writethrough
    #define CR0_CD		0x40000000	// Cache Disable
    #define CR0_PG		0x80000000	// Paging
    ```

  - cr0被设置为，开启保护模式，**保护模式开启时只是开启了段级保护，没有开启分页，即逻辑地址转换成线性地址，线性地址直接等于物理地址**, CR0_PG开启分页，CR0_AM 开启地址对齐检查， 开启写保护， 开启协处理器错误， 开启监控协处理器。TS任务已切换标志为0，EM为0，表示有协处理器，会将浮点指令交给协处理器用软件来模拟。

- 完成。



### 如何保护内核数据和代码：

- 通过虚拟地址空间的隔离。检查用户访问的虚拟地址与`ULIM`，可以防止用户访问高地址。
- 为`ULIM`以下的每个页面设置权限，并启用cr0中的非法写保护，可以防止无权限用户修改只读页面。

### 如何管理内存空闲空间，以及管理的开销：

- 通过一对一地维护每个页框的信息，并动态维护一个空闲页框链表（头指针）来实现。
- 每个页框PageInfo的大小是8B，`pages`的大小是PTSIZE=4MB, 总共可存放512K个PageInfo，即可维护512K个物理页，总共512K*PGSIZE=2G物理内存。
- 如果全部2G的物理内存都分配出去，那么维护的开销是，`pages`大小+`kern_pgdir`大小+所有页表页大小=4MB+4K+2MB的额外内存。


****

**推荐阅读： [OS操作系统实验 xv6调度算法实现]({{ site.url }}/lab report/2020/01/31/OS4)**

