---
title: MIT 6.828 JOS操作系统实验6-文件系统实现详解
category: Lab Report
tags: [OS, JOS]
excerpt_separator: <!--more-->
---

# Lab6：文件系统实现


JOS操作系统提供了一个简单的文件系统，它可以满足基本功能：创建，读取，写入，删除和以分层目录结构来组织文件。但并未提供文件所有权、用户权限的概念，不支持硬链接、符号链接、时间戳或特殊设备文件等，它提供的保护仅能捕获错误。

<!--more-->
## 磁盘上的文件系统结构

文件储存在磁盘上，磁盘最小的存储单位为扇区，JOS采用的扇区大小为512字节。为了提高数据交换的效率，与磁盘的一次数据交换以块为单位，不同的操作系统定义不同的块大小，具体而言，更大地块管理的开销越小，但是可能出现的内部碎片越多。JOS采用的块大小为4096bytes，与内存的页大小相同。块和扇区的大小定义在`inc/fs.h`和`fs/fs.h`中：

```c
#define BLKSIZE		PGSIZE
#define BLKBITSIZE	(BLKSIZE * 8)

#define SECTSIZE	512			// bytes per disk sector
#define BLKSECTS	(BLKSIZE / SECTSIZE)	// sectors per block
```

大多数UNIX文件系统采用索引结构来组织文件的块，为每一个文件分配一个**索引节点（inode）**，索引节点上保存了文件的重要元数据，包括文件名称，大小，创建者，创建日期，权限，以及文件的数据块存放的物理位置。**inode号码**而非文件名唯一地标识了系统中的一个文件。

一般情况下，文件全名与inode号码是意义对应的关系，但UNIX系统允许硬链接，也就是多个文件名指向同一个inode，可以用不同的文件名访问同一个文件，不同的文件名没有依赖关系。当删除一个文件名时，不会影响到其他文件名对该inode的访问，只有在链接数为0的时候，该inode才会被真正释放。由于JOS不实现硬链接或者符号链接，因此目录之间是没办法共享文件的，所以不需要采用inode结构，直接将文件的元数据存放在目录条目中。

### 超级块

在一个UNIX系统中，一般会存在许多种文件系统，例如ext4，NFS, tmpfs等等，他们有着自己的组织方法，权限设置，文件数据块大小定义等，因此文件的创建、打开、删除等具体操作都是不同的。为了让用户透明地处理文件，操作系统引入了虚拟文件系统，封装不同文件系统的文件操作，为用户提供统一的文件操作接口。用户访问文件系统的过程：

 ![How userspace accesses various types of filesystems](https://user-gold-cdn.xitu.io/2019/5/27/16af9c99190c78db?imageView2/0/w/1280/h/960/format/webp/ignore-error/1) 

这样，不同的文件系统必须提供描述该系统的元数据，例如块大小，总磁盘大小，根目录，块设备驱动程序描述符指针，上次安装文件系统的时间，上次检查文件系统错误的时间等等，这些元数据存放在**超级块** 中。超级块存放在磁盘的最开始或者结尾，并且为了可靠性会在不同区域中备份。

JOS的超级块数据结构定义在`inc\fs.h`中：

```c
struct Super {
	uint32_t s_magic;		// Magic number: FS_MAGIC
	uint32_t s_nblocks;		// Total number of blocks on disk
	struct File s_root;		// Root directory node
};
```

包括文件系统魔术字、总共有多少块和根目录节点。

 ![磁盘布局](https://pdos.csail.mit.edu/6.828/2018/labs/lab5/disk.png) 

### 文件元数据

`struct File`描述了文件的元数据：

```c
struct File {
	char f_name[MAXNAMELEN];	// filename
	off_t f_size;			// file size in bytes
	uint32_t f_type;		// file type

	// Block pointers.
	// A block is allocated iff its value is != 0.
	uint32_t f_direct[NDIRECT];	// direct blocks
	uint32_t f_indirect;		// indirect block

	// Pad out to 256 bytes; must do arithmetic in case we're compiling
	// fsformat on a 64-bit machine.
	uint8_t f_pad[256 - MAXNAMELEN - 8 - 4*NDIRECT - 4];
} __attribute__((packed));	// required only on some 64-bit machines
```

- f_name: 文件名，`MAXNAMELEN`定义为128字节。
- f_size: 以字节为单位的文件大小
- f_type： 文件类型（用来区分是普通文件还是目录）
- f_direct:  直接索引块，NDIRECT被定义为10。
- f_indirect: 间接索引块，只有1块。
- f_pad:  这是元数据结构的padding，填充使得`File`结构的大小为256字节。

元数据存储在磁盘上的目录条目中。文件数据块使用索引来组织，每一条索引存放着对应文件数据块的物理地址（块号）。直接索引块有10个，意味着大小不超过10 \*4096=40KB的文件可以直接映射。间接索引块只有一个，可以存放4096/4 = 1024个额外的直接索引块，因此可以提供1024\*4096 =4MB的空间。

**文件系统允许的文件大小为4MB+40KB，1034块**。

 ![档案结构](https://pdos.csail.mit.edu/6.828/2018/labs/lab5/file.png) 



### 目录文件

目录实际上也是一个文件，只是其中存放的是每个文件的元数据（`File`结构体），描述其中的文件和子目录。

**根目录的元数据保存在超级块中**。根目录文件存放了一系列子目录和文件的`File`元数据，虚拟文件系统可以根据超级块找到根目录的元数据，从而访问到根目录文件的每一块，进而再通过元数据找到具体文件。

### 磁盘空间管理

如上面的结构图所示，JOS用bitmap来管理磁盘的空闲块。在创建文件或文件内容需要扩展时，JOS根据bitmap找到空闲的磁盘块并分配给它；当文件空间释放时，文件系统将bitmap中对应的块置为1，表示该块已经空闲。Bitmap的管理相对于空闲块链表来说更加高效，但是也会占据较大空间。因为文件系统最多只能管理3GB的磁盘，则最多有786，432块，需要98，304字节的bitmap，即24个bitmap块。

### 访问磁盘

文件系统的实现需要访问磁盘。一般的UNIX系统中，因为有许多异质的IO设备，因此要将磁盘驱动程序安装为内核的一部分，并提供系统调用使得文件系统可以访问磁盘。在JOS中，为了方便，直接将磁盘驱动程序实现在文件系统的用户级环境中，使得文件系统可以直接访问磁盘。

JOS中程序与磁盘数据交换的方式是轮询， 即基于编程IO的磁盘访问。



本次实验中，我们要实现与探究的问题有：

1. 文件系统中，文件逻辑块是如何映射到物理块上的
2. 文件创建或扩展时，如何分配磁盘块
3. 需要访问文件时，如何将文件数据从磁盘上读取到内存缓冲区中
4. 如何把内存中的文件数据写回到磁盘
5. 如何为用户提供读、写、创建、删除的接口

## TODO1:

> i386_init通过将ENV_TYPE_FS类型传递给环境创建函数env_create来标识文件系统环境。在env.c中修改env_create，使其授予文件系统环境I/O权限，但不要将该权限授予其他环境。

JOS中，环境实际上与用户进程类似， 一个环境包括环境的状态（RUNNING \RUNNABLE等），环境的地址空间，环境的父级id，中断帧等，与进程相似。可以通过系统调用fork来创建用户环境，当发生中断时，保存用户环境的运行状态到中断帧中，然后进入内核处理中断。环境描述还包括环境的类型，只有两种——ENV_TYPE_USER和ENV_TYPE_FS，前者为普通的用户进程，后者为文件系统环境。

系统开启时，在`init.c`的`i386_init()`中会创建文件系统进程：

```c
	...
	boot_aps();		//将初始化代码拷贝到MPENTRY_PADDR处，然后依次启动所有AP

	// Start fs.
	ENV_CREATE(fs_fs, ENV_TYPE_FS);

```

 x86处理器使用EFLAGS寄存器中的IOPL位来确定是否允许保护模式代码执行特殊的设备I / O指令，例如IN和OUT指令。JOS的磁盘寄存器位于x86的IO空间中，因此我们需要赋予文件系统环境IO权限，但不允许其他任何环境，包括内核访问磁盘。

一个进程只能使用`POPF`指令来更改IOPL，但是这条指令的执行也需要内核级特权，文件系统环境并不具有内核级特权。所以，需要在运行于内核模式的`env_create`中，判断环境类型是否为文件系统，然后手动修改该环境下的EFLAGS寄存器。代码为：

```c
void
env_create(uint8_t *binary, enum EnvType type)
{
	// LAB 3: Your code here.
	struct Env *e;
	int r;
	if ((r = env_alloc(&e, 0) != 0)) {
		panic("create env failed\n");
	}

	// If this is the file server (type == ENV_TYPE_FS) give it I/O privileges.
	// LAB 5: Your code here.
	if(type == ENV_TYPE_FS)
		e->env_tf.tf_eflags |= FL_IOPL_MASK;

	load_icode(e, binary);
	e->env_type = type;
}

```

其中，FL_IOPL_MASK在`mmu.h`中定义。eflags寄存器中的第12和13位为IO特权级别位（IOPL），从0-3分别对应4种模式：内核、驱动程序、驱动程序、应用程序。当前的进程只有当权限位<=IOPL时，才可以获得访问IO的权限， 否则就会引发保护异常。

`e->env_tf.tf_flags|=FL_IOPL_MASK`将EFLAGS寄存器置为0x3000，即IOPL=3，也就是该进程的用户模式可访问IO, 于是文件系统便获得IO权限，而对于其他用户环境，eflags中的IOPL位仍为0. **因为每一个进程都有自己独立的标志寄存器，所以用户环境与文件系统环境具有不同的IOPL。**

>Env 数据结构为环境描述符，类似于PCB，在`inc\env.h`中定义：
>
>```c
>struct Env {
>	struct Trapframe env_tf;	// Saved registers
>	struct Env *env_link;		// Next free Env
>	envid_t env_id;			// Unique environment identifier
>	envid_t env_parent_id;		// env_id of this env's parent
>	enum EnvType env_type;		// Indicates special system environments
>	unsigned env_status;		// Status of the environment
>	uint32_t env_runs;		// Number of times environment has run
>	int env_cpunum;			// The CPU that the env is running on
>
>	// Address space
>	pde_t *env_pgdir;		// Kernel virtual address of page dir
>
>	// Exception handling
>	void *env_pgfault_upcall;	// Page fault upcall entry point
>
>	// Lab 4 IPC
>	bool env_ipc_recving;		// Env is blocked receiving
>	void *env_ipc_dstva;		// VA at which to map received page
>	uint32_t env_ipc_value;		// Data value sent to us
>	envid_t env_ipc_from;		// envid of the sender
>	int env_ipc_perm;		// Perm of page mapping received
>};
>```
>
>其中 `env_tf`为中断帧，中断帧数据结构`Trapframe`与之前的实验类似，`tf_eflags`为进程的EFLAGS寄存器。
>
>

###问题：

**在环境切换的时候是否需要其他操作来确保IOPL正确设置和还原？**

不需要。因为在环境切换的时候，操作系统会保存旧的中断帧在内核栈上，其中就包括了EFLAGS寄存器。在环境切换回来的时候，会从内核栈上加载中断帧，恢复原来的EFLAGS寄存器值。

### 检查

运行`./grade-lab5`进行检查：

![image-20191212212137127](C:\Users\CST\AppData\Roaming\Typora\typora-user-images\image-20191212212137127.png)

`fs i/o`结果为OK, 说明文件系统已有IO权限。



-------

## TODO2:

> ​    在fs / bc.c中实现bc_pgfault和flush_block函数。 bc_pgfault是一个页面错误处理程序，bc_pgfault的工作是从磁盘加载页面，flush_block函数应将一个块写出到磁盘。  

为了解决磁盘与CPU处理数据速度的不同与数据大小的不同，通常要在两者之间加入缓冲区。缓冲区实际上是内存上的一块特定区域。JOS的缓冲区只有一个块， 代码在`fs\bc.c`中。

### bc_pgfault()

文件系统在JOS中本质上是一个进程，它如何提供其他进程访问文件的接口呢？

内核将磁盘的物理地址映射到文件系统进程的高虚拟地址空间。在`fs.h`中，磁盘映射的虚拟地址和磁盘大小的定义是：

```c
/* Disk block n, when in memory, is mapped into the file system
 * server's address space at DISKMAP + (n*BLKSIZE). */
#define DISKMAP		0x10000000

/* Maximum disk size we can handle (3GB) */
#define DISKSIZE	0xC0000000
```

意味着该文件系统只能处理最多3GB的磁盘。磁盘地址被映射到文件系统进程的高地址3GB区域，从0x10000000开始。这样，磁盘地址映射到文件系统虚拟地址，而虚拟地址又在被访问时映射到内存的物理地址，当其他进程向文件系统请求读取文件时，文件系统访问虚拟地址空间来读取相应的块。

对于文件系统，磁盘块的读取和普通进程访问代码和数据时按需调页的策略类似，是按需加载的。只有当某个磁盘块被需要时，才要将其加载进内存。当文件系统进程访问某个虚拟地址时，如果对应的磁盘块还不在内存中，就会发生缺页故障，这时应该将虚拟地址对应的磁盘块读入内存，这与普通用户的页错误处理不同，需要自定义页错误处理程序。`bc_pgfault`函数将磁盘块读取到对应的内存，才可以重新执行该访问。

>正常情况下，用户环境运行在JOS分配给用户的正常栈上。但当用户模式下发生页错误时，JOS内核将从正常用户栈切换到用户异常栈，来运行用户级页面错误处理程序。在异常栈上，有`UTrapframe`结构体，为**异常中断帧**，它保存了引起页错误的虚拟地址。
>
>处理用户态页面异常的过程是：
>
>1. 发生异常之前，用户已经向内核注册自定义的页面处理程序在env_pgfault_upcall中。
>2. 用户态发生页错误，陷入内核态，保存正常的中断帧Trapframe，切换到内核栈，进入trap（）
>3. 根据中断号发现是缺页故障，调用`page_fault_handler`进行处理
>4. 检测trapframe上保存的cs寄存器，发现是用户态
>5. 判断是否有用户自定义页面异常处理程序，若没有则销毁进程。
>6. 如果有，压入Utrapframe，并设置tf->tf_eip即下一条指令为用户自定义的缺页处理程序，返回用户态
>7. 用户态会从自定义处理程序开始执行。
>
>**文件系统进程自定义了它的缺页处理程序为bc_pgfault**
>
>

`bc_pgfault`首先对访问的虚拟地址进行了合法性检查：

```c
static void
bc_pgfault(struct UTrapframe *utf)
{
	void *addr = (void *) utf->utf_fault_va;
	uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;
	int r;

	// Check that the fault was within the block cache region
	if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
		panic("page fault in FS: eip %08x, va %08x, err %04x",
		      utf->utf_eip, addr, utf->utf_err);

	// Sanity check the block number.
	if (super && blockno >= super->s_nblocks)
		panic("reading non-existent block %08x\n", blockno);

```

虚拟地址必须在磁盘映射的虚拟地址范围之内，并且由于文件系统所占有的磁盘空间可能比3GB要小，所以要检查对应的磁盘块号是否在文件系统的范围内（`blockno < super->s_nblocks`).

接下来，由于`addr`不一定与页大小对齐，所以要先将它与`PGSIZE`对齐：

```c
addr = ROUNDDOWN(addr, PGSIZE);
```

然后将磁盘块读入内存，则首先要在内存中分配一个页。JOS的`syscall.c`中实现了分配内存页的系统调用`sys_page_alloc`, 它的接口是：
```c
static int
sys_page_alloc(envid_t envid, void *va, int perm)
```

它为id为envid的环境在内存中分配一个页，并将页首地址映射到虚拟地址va, 设置这个页的权限为perm。（与上次实验一样，PTE_U表示用户可读，PTE_W表示可写，PTE_P表示存在内存中）。

我们知道`init.c`中，文件系统环境是第一个被create出来的进程：

```c
	// Starting non-boot CPUs
	boot_aps();		//将初始化代码拷贝到MPENTRY_PADDR处，然后依次启动所有AP

	// Start fs.
	ENV_CREATE(fs_fs, ENV_TYPE_FS);

#if defined(TEST)
	// Don't touch -- used by grading script!
	ENV_CREATE(TEST, ENV_TYPE_USER);

```

它的envid为0.

所以可以用：

```c
sys_page_alloc(0, addr, PTE_W|PTE_U|PTE_P)
```

来分配页。文件系统对这个页的权限当然是可读可写。

如果内存大小不够的话，`sys_page_alloc`会返回E_NO_MEM错误，否则返回0. 这里加以判断，如果内存不足抛出panic。

`ide.c`磁盘驱动程序提供了读取磁盘的接口：

```c
int
ide_read(uint32_t secno, void *dst, size_t nsecs)
{
	int r;

	assert(nsecs <= 256);

	ide_wait_ready(0);

	outb(0x1F2, nsecs);
	outb(0x1F3, secno & 0xFF);
	outb(0x1F4, (secno >> 8) & 0xFF);
	outb(0x1F5, (secno >> 16) & 0xFF);
	outb(0x1F6, 0xE0 | ((diskno&1)<<4) | ((secno>>24)&0x0F));
	outb(0x1F7, 0x20);	// CMD 0x20 means read sector

	for (; nsecs > 0; nsecs--, dst += SECTSIZE) {
		if ((r = ide_wait_ready(1)) < 0)
			return r;
		insl(0x1F0, dst, SECTSIZE/4);
	}

	return 0;
}
```

磁盘驱动器中读取磁盘的单位是一个扇区。提供的接口中，`secno`表示开始扇区号，`dst`表示数据输出的地址，`nsecs`表示读取扇区数目。

`BLKSECTS`定义在`fs.h`中，为`BLKSIZE/SECSIZE`,表示一个块包含的扇区数，则这里`secno`应该为 `blockno * BLKSECTS`, 目的地址为`addr`， 读取扇区数为`BLKSECTS`.

`bc_pgfault()`的实现：

```c
static void
bc_pgfault(struct UTrapframe *utf)
{
	void *addr = (void *) utf->utf_fault_va;
	uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;
	int r;

	// Check that the fault was within the block cache region
	if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
		panic("page fault in FS: eip %08x, va %08x, err %04x",
		      utf->utf_eip, addr, utf->utf_err);

	// Sanity check the block number.
	if (super && blockno >= super->s_nblocks)
		panic("reading non-existent block %08x\n", blockno);

	// Allocate a page in the disk map region, read the contents
	// of the block from the disk into that page.
	// Hint: first round addr to page boundary. fs/ide.c has code to read
	// the disk.
	//
	// LAB 5: you code here:
	addr = ROUNDDOWN(addr, PGSIZE);
    	if((r=sys_page_alloc(0, addr, PTE_W|PTE_U|PTE_P))<0)
		panic("failed to alloc page: %e",r);
    	if ((r = ide_read(blockno*BLKSECTS, addr, BLKSECTS)) < 0){
        	panic("in bc_pgfault,ide_read: %e", r);
    	}


	// Clear the dirty bit for the disk block page since we just read the
	// block from disk
	if ((r = sys_page_map(0, addr, 0, addr, uvpt[PGNUM(addr)] & PTE_SYSCALL)) < 0)
		panic("in bc_pgfault, sys_page_map: %e", r);

	// Check that the block we read was allocated. (exercise for
	// the reader: why do we do this *after* reading the block
	// in?)
	if (bitmap && block_is_free(blockno))
		panic("reading free block %08x\n", blockno);
}

```



### flush_block

flush_block函数必要时将内存中的一个磁盘块的数据写回磁盘, 然后将该块的脏位置为0. 这里的必要，指的是当该块在内存中并且脏位为1的时候。假如数据没有发生改变，那么就没必要写磁盘。形参`addr`是文件系统进程中的一个虚拟地址，用`blockno=((uint32_t)addr - DISKMAP) / BLKSIZE;`将它转为磁盘块号。同样，`addr`需要对齐到页大小。

根据提示，首先用`va_is_mapped`判断该虚拟地址是否绑定到内存物理地址上，如果没有，说明该块不在内存中，不需要写回。用`va_is_dirty`判断该虚拟地址对应页是否被修改过，如果没有也直接返回。

否则用磁盘驱动器的`ide_write(uint32_t secno, const void *src, size_t nsecs)`接口来写回块。同样以扇区为单位操作，src是数据块的源虚拟地址。

写回之后，把内存上该块的脏位（PTE_D）置为0. 可以用`sys_page_map`来实现。sys_page_map的接口是：

```c
static int
sys_page_map(envid_t srcenvid, void *srcva,
	     envid_t dstenvid, void *dstva, int perm)
```

它将进程 srcenvid的虚拟地址srcva映射到进程dstenvid的dstva处，并将权限设置为perm。

可以通过将源和目的虚拟地址都设置为文件系统的虚拟地址`addr`, 并将perm设置为 uvpt[PGNUM(addr)]& PTE_SYSCALL来把PTE_D清空, 并保留addr页面原来的权限设置。

PTE_SYSCALL是只有用户程序进行系统调用时才会设置的，它的值是：`#define PTE_SYSCALL	(PTE_AVAIL | PTE_P | PTE_W | PTE_U)`, PTE_AVAIL表示可为用户使用。

加上错误处理，flush_block()实现便完成：

``` c
// Flush the contents of the block containing VA out to disk if
// necessary, then clear the PTE_D bit using sys_page_map.
// If the block is not in the block cache or is not dirty, does
// nothing.
// Hint: Use va_is_mapped, va_is_dirty, and ide_write.
// Hint: Use the PTE_SYSCALL constant when calling sys_page_map.
// Hint: Don't forget to round addr down.
void
flush_block(void *addr)
{
	uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;

	if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
		panic("flush_block of bad va %08x", addr);

	// LAB 5: Your code here.
    int r;
    addr = ROUNDDOWN(addr, PGSIZE);
    if(!va_is_mapped(addr)|| ! va_is_dirty(addr))
        return;
    if((r = ide_write(blockno * BLKSECTS, addr, BLKSECTS))<0)
        panic("in flush_block, ide_write(): %e",r);
    if((r = sys_page_map(0,addr, 0,addr, uvpt[PGNUM(addr)]&PTE_SYSCALL))<0)
        panic("in flush_block, sys_page_map: %e",r);

}  
```

### 检查

`make grade`后：



`check_bc`,`check_super`和`check_bitmap`OK，表示两个函数实现正确。



-----

## TODO3:

> 使用free_block作为模板在fs / fs.c中实现alloc_block，后者应在bitmap中找到可用的磁盘块，将其标记为已使用，然后返回该块的编号。 分配该块时，应立即使用flush_block将更改后的bitmap块刷新到磁盘，以帮助文件系统保持一致。

alloc_block的过程是：

- 从第一个块号开始，调用`block_is_free`判断它是否空闲，找到第一个空闲的块。`block_is_free`就是从判断`bitmap`中该块对应的bit是否为1（free），如果是则返回真。
- 找到第一个空闲块后，将bitmap中对应的位置为0.
- 使用flush_block()将该块对应的bitmap块写回磁盘，保持同步。因为flush_block()接收的是虚拟地址，所以还要使用`diskaddr`将块号转化为虚拟地址。

`bitmap`数据结构在`fs.h`中定义，是一个类型为`uint32_t`的数组，那么数组的一个元素（32位）就可以表示32个块。要判断一个块是否空闲，应该用`bitmap[blockno/32] &= 1<<blockno%32`是否为1来判断。则要将某一位置为0，可以用`bitmap[blockno/32] &= ~(1<<blockno%32)`。

在JOS中，磁盘的第一个块（blockno=0）是boot sector，第二个块（blockno=1）是超级块，第三个块（blockno=2）开始才是bitmap块。因此给定一个块号blockno，它的bitmap所在的块为： `2+ (blockno/32) / (BLKSIZE /4 )`.  一个bitmap元素的大小是4个字节，一个块大小是`BLKSIZE`，所以一个块总共有`BLKSIZE/4`个bitmap元素， blockno/32为该块对应的bitmap索引号，除以每块能存放的元素数就是相对的块偏移。

`diskaddr`定义在`bc.c`中，它接收一个磁盘块号，返回在文件系统进程中对应的虚拟地址。

alloc_block的实现如下：

```c
int
alloc_block(void)
{
	// The bitmap consists of one or more blocks.  A single bitmap block
	// contains the in-use bits for BLKBITSIZE blocks.  There are
	// super->s_nblocks blocks in the disk altogether.

	// LAB 5: Your code here.
	for(uint32_t blockno=0; blockno< super->s_nblocks; blockno++){
		if(block_is_free(blockno){
			bitmap[blockno/32] &= ~(1<<(blockno%32)); //将bitmap上对应的位置为0，表示占用。
			flush_block(diskaddr(2 + (blockno / 32) / (BLKSIZE/4))); //bitmap的第一块磁盘块号是2, blockno/32是该blockno在bitmap中的索引号。
			return blockno;
		}
	}

	return -E_NO_DISK;
}
```

### 检查

`make grade`:



`alloc_block`OK, 代码正确。

----

## TODO4

> 实现file_block_walk和file_get_block。 file_block_walk从文件中的块偏移量映射到struct File或间接块中的指针，非常类似于pgdir_walk对页表所做的操作。 file_get_block更进一步，并映射到实际的磁盘块，并在必要时分配一个新的磁盘块。

### 阅读fs. c中的代码

fs.c中实现了文件系统的各种操作，包括初始化、从根目录开始遍历文件系统以找到某个绝对路径标识的文件、在目录上分配一个文件的条目、获取一个文件的条目、获取文件的某个块等基本操作，以及对文件进行创建、打开、删除、修改、读取、写回磁盘、扩展、截短等操作。

JOS中，有两个磁盘映像。如果磁盘1可用的话，磁盘0只用来装载内核，磁盘1上才实现了文件系统。`fs_init()`函数检查可用磁盘，并设置超级块`super`为该磁盘的第二块。

fs.c中所有的函数以及对应的功能为：

- bitmap操作：
  - `block_is_free(blockno)`: 接收磁盘块号，根据bitmap判断块是否空闲
  - `free_block(blockno)`: 接收磁盘块号，释放该块，将bitmap中对应位置1.
  - `alloc_block()`: 根据bitmap找到第一个空闲块，将它分配出去，返回块号
- 文件系统结构构建和维护：
  - `fs_init()`： 初始化文件系统。找到可用磁盘，设置磁盘驱动，读取超级块并保存指针到`super`。
  - `file_block_walk(struct File *f, uint32_t filebno, uint32_t **ppdiskbno, bool alloc)`: 查找文件`f`的第`filebno`个块的磁盘地址（通过该文件元数据中的索引来查找），将该索引的地址存放到`ppdiskbno`中；由于直接索引只有10个，则当块号`fileno`大于9时，而间接索引还没有分配的时候，如果`alloc`，就分配一个间接索引页，然后将`ppdiskbno`置为该间接索引页上对应的索引地址。
  -  `file_get_block(struct File *f, uint32_t filebno, char **blk) `: 查找文件`f`的第`filebno`块在文件系统进程中对应的虚拟地址，并将其保存在`blk`中
  - ` dir_lookup(struct File *dir, const char *name, struct File **file) `: 在`DIR`指定的目录下寻找名字为`name`的文件的元数据，并将元数据的地址保存到`file`中。
  - ` dir_alloc_file(struct File *dir, struct File **file) `: 在`dir`目录下寻找一个没有被使用的`File`结构，把它分配给一个新文件使用（调用者负责填充这块元数据），将它的地址存放到`file`中。
  - ` walk_path(const char *path, struct File **pdir, struct File **pf, char *lastelem) `: 解析`path`中的文件路径，如果成功，将文件的元数据地址存放在`pf`中，将文件所在目录的元数据地址存放在`pdir`中。如果不成功，`pdir`还是存放最后匹配的目录元数据地址，而`lastelem`存放最后无法匹配的路径字符串。例如目录/aaa/bbb/下不存在/aaa/bbb/c.c，则pdir指向/aaa/bbb的元数据，`lastelem`='c.c'
- 文件操作：
  - `file_create(const char *path, struct File **pf)`: 创建`path`文件/目录，如果成功，将该文件或目录的元数据指针放在`pf`中。
  - `file_open(const char *path, struct File **pf)`: 打开`path`， 如果成功，将该文件元数据地址赋给`pf`.
  - `file_read(struct File *f, void *buf, size_t count, off_t offset)`: 从文件`f`的`offset`位置（字节为单位）开始， 读取`count`个字节到`buf`中，返回实际读取字节数。
  - `file_write(struct File *f, const void *buf, size_t count, off_t offset)`: 向文件`f`从`offset`位置开始，写入`count`个字节，数据的来源是`buf`。 如果写的时候文件的大小超过了它已分配的块就要扩展文件。
  - `file_free_block(struct File *f, uint32_t filebno)`: 删除文件`f`的第`filebno`个块。
  - `file_truncate_blocks(struct File *f, off_t newsize)`: 将文件`f`截短到`newsize`的大小。具体操作是，计算原来文件具有的块数和新的块数，将超过新块数的部分释放掉。如果新的大小已经小于10块，则不需要间接索引了，释放掉间接索引块
  - `file_set_size(struct File *f, off_t newsize)`: 设置文件`f`的大小为`newsize`,自动截短或扩展。
  - `file_flush(struct File *f)`: 将文件`f`的数据和元数据写回磁盘，遍历文件的所有块，将所有的脏块写回磁盘。

### file_block_walk（）

查找文件`f`的第`filebno`个块的磁盘地址（通过该文件元数据中的索引来查找），将该索引的地址存放到`ppdiskbno`中。如果`alloc`置位，必要时分配间接索引页，否则返回错误信息。函数的过程：

- 判断`filebno`，是否在直接索引支持的范围内（0-9），如果是，就直接将ppdiskbno赋为直接索引上对应的条目的地址，即`& (f->f_direct[filebno])`.
- 如果`filebno`大于9，但是小于1034, 即文件系统可以支持的文件最大块数
  - 如果间接索引块还没有分配，要检查`alloc`，如果alloc为0，返回`-E_NOT_FOUND`表示找不到索引条目。
  - 如果alloc为1：
    - 调用`alloc_block`为间接索引分配一个块， 如果分配错误，返回`-E_NO_DISK`。
    - 分配成功，则将间接索引链接到这个块上，即`f->f_indirect = blockno`，这样查询10号以后的块时，首先查找`f->f_indirect`得到索引块的地址，然后在索引块上找到对应的直接索引条目，根据该条目的值找到物理块号。
    - 分配好之后，还要将索引块初始化为全0，表示这些文件块还没有分配和映射。
  - 无论是新分配的，还是原来就存在的，都要将间接索引块上对应的索引条目地址存放到`ppdiskbno`中。 **如何获得索引块上对应的条目地址呢？** 首先，`f->f_indirect`给出索引块的物理地址，用`diskaddr(f->f_indirect)`就可以知道它的虚拟地址，也就是这个块上第一个索引条目的地址。这个虚拟地址再加上`filebno`在索引块上的偏移量就可以得到索引条目的虚拟地址，代码上有两种实现方式：
    1. `(uintptr_t*) diskaddr(f->f_indirect)`将索引块的虚拟地址转为JOS的指针类型，指针可以用`[i]`操作符来取得第i个数据，这里i应该是`filebno-10`, 因为是索引块上的第`filebno-10`条索引，最后用`&`取址。
    2. 首先用` diskaddr(f->f_indirect)`获得第一个索引条目的虚拟地址，然后加上`(filebno-10)*4`就得到第`filebno-10`的地址，因为每个索引条目大小是4个字节。
- 如果`filebno`超过1034，是无效的，返回`-E_INVAL`，否则返回0，表示成功。

```c
static int
file_block_walk(struct File *f, uint32_t filebno, uint32_t **ppdiskbno, bool alloc)
{
       // LAB 5: Your code here.
	uint32_t blockno;
	if(filebno <=9){
		*ppdiskbno = &(f->f_direct[filebno]);
	}
	else if(filebno<1034){
		if(!f->f_indirect){
			if(!alloc)
				return -E_NOT_FOUND;
			
			else{
				if((blockno=alloc_block())<0)
					return -E_NO_DISK;
				f->f_indirect = blockno;
				memset(diskaddr(blockno), 0, BLKSIZE);
			}
		}
		*ppdiskbno = &((uintptr_t*) diskaddr(f->f_indirect))[filebno-10];
        
       //或者 *ppdiskbno = diskaddr(f->f_indirect)+ (filebno-10)*4;
	}
	else return -E_INVAL;
	return 0;
		
}
```



### file_get_block()

file_get_block在file_block_walk的基础上，更进一步地，获取文件块对应的物理块号，并将物理块在文件系统进程中对应的虚拟地址保存在`blk`中，过程是：

- 调用`file_block_walk`获取`f`的第`filebno`个块的索引条目地址，保存在`slot`中，要把`alloc`设置为1，表示如果需要用到间接索引但索引块还未分配时要自动分配。
- 如果`flie_block_walk`返回值小于0，表示发生了错误，返回对应的错误码。
- 如果`slot`的值为0，即该文件逻辑块还没有分配到具体的物理块，可能是文件被创建或者被扩展等，要为该文件块分配一个物理块，使用`alloc_block()`
  - 如果分配错误应返回`-E_NO_DISK`
  - 分配成功，则令对应的索引条目值为返回的物理块号，这样文件块就被映射到物理块上。
- 无论文件块原来是否已经分配，将物理块对应的虚拟地址保存在`blk`中。`*slot`是对应的物理块号，调用`diskaddr(*slot)`就可以找到虚拟地址，将它保存在`blk`中，成功返回0.

```c
int
file_get_block(struct File *f, uint32_t filebno, char **blk)
{
       // LAB 5: Your code here.
	uint32_t *slot=NULL;    //slot = 索引条目的地址，*slot = 文件块对应的物理块号码
	uint32_t errorcode=file_block_walk(f, filebno, &slot, 1);
	uint32_t blockno;

	if(errorcode<0)
		return errorcode;
	if(!*slot){
		if((blockno = alloc_block())<0)
			return -E_NO_DISK;
		*slot = blockno;
	}
	*blk = (char *)diskaddr(*slot);
	return 0;

}
```

### 检查

`make grade`:


`file_get_block`OK，代码正确。



----

## TODO5:

> •在fs/serv.c中实现serve_read 。
>
> •serve_read的大量操作将由fs / fs.c中已经实现的file_read来完成。 serve_read只需提供RPC接口即可读取文件。 查看serve_set_size中的注释和代码，以大致了解服务器功能的结构。

### 请求文件系统服务的过程

文件系统进程内部实现了对文件的各种操作，但是用户进程无法直接调用这些函数，因为它们是在文件系统进程的内存空间内。这时，需要采用**进程间通信（IPC机制）**来在进程之间交换数据或者方法。JOS中，使用主从式架构来进行文件系统和其他进程之间的通信。通常，使用IPC进行数据交换的两个进程可以被分为服务端和用户端，客户端向服务器发出请求，服务器进行处理之后回应请求。JOS用建立在IPC基础上的远程过程调用（RPC）来进行通信。RPC在两个应用之间建立TCP连接，然后客户端应用将过程调用的参数序列化成二进制数据之后发送给服务端，服务端反序列化之后进行过程调用，返回值再序列化后发送回客户端。

JOS的文件系统调用过程是：

<img src="{{ site.url }}/assets/images/report/OS6.png" alt="image-20191213134403520" style="zoom: 40%;" />

底层实际上是进程间通信。

文件系统的相关数据结构有：

- 文件描述符`Fd`： 文件所在的设备id（因为一个文件系统可以跨多个设备），是否打开, 以及`fd_offset`文件的光标等。
- 设备描述符`Dev`： 设备上各种操作调用的入口，设备名字和设备id。
- 打开文件描述符列表`OpenFile`结构，由内核维护的`opentab`数组，保存了文件元数据的地址，文件打开状态，文件id以及磁盘上文件描述符`Fd`结构的指针
- `File`结构（文件元数据）：维护文件重要信息，完成逻辑块到物理块的映射等。

 <img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/6.828/lab5/lab5_4_%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png" alt="技術分享圖片" style="zoom:50%;" /> 

文件描述符表映射在磁盘的0xD0000000处，文件描述符结构中包含了设备id，当进程调用`fd.c`中的`read`时，将传入文件描述符号，通过文件描述符号找到对应地`Fd`结构体，进而查找设备，设备用`Dev`结构来描述，结构体中包含了`dev_read`, `dev_write`等设备读、写函数的指针。`devfile_read`函数定义在`file.c`中，它将调用`fsipc()`向服务器进程发起请求。

`fsipc()`函数专门负责与文件系统进程进行通信，它建立在`ipc`机制上。

`ipc.c`封装了两个函数——`ipc_send( envid_t to_env, uint32_t val, void *pg, int perm )`和`ipc_recv( envid_t *from_env_store, void *pg, int *perm_store )`，允许与某一个环境进行数据交换，交换的消息包含两个部分：

- 1个32位的值
- 可选的页映射关系

其中，`send`中的`val`是交换的数值，`pg`参数表示发送进程希望与接收进程共享`pg`对应的物理页，并且在接收进程中，对该页的权限是`perm`。

文件系统实现了一个`Fspic`数据结构：

```c
union Fsipc {
	struct Fsreq_open {
		char req_path[MAXPATHLEN];
		int req_omode;
	} open;
	struct Fsreq_set_size {
		int req_fileid;
		off_t req_size;
	} set_size;
	struct Fsreq_read {
		int req_fileid;
		size_t req_n;
	} read;
	struct Fsret_read {
		char ret_buf[PGSIZE];
	} readRet;
	struct Fsreq_write {
		int req_fileid;
		size_t req_n;
		char req_buf[PGSIZE - (sizeof(int) + sizeof(size_t))];
	} write;
	struct Fsreq_stat {
		int req_fileid;
	} stat;
	struct Fsret_stat {
		char ret_name[MAXNAMELEN];
		off_t ret_size;
		int ret_isdir;
	} statRet;
	struct Fsreq_flush {
		int req_fileid;
	} flush;
	struct Fsreq_remove {
		char req_path[MAXPATHLEN];
	} remove;

	// Ensure Fsipc is one page
	char _pad[PGSIZE];
};
```

它是一个联合，其中的各种结构体保存了对应的文件操作需要的参数，例如`fsipc.read`就有要读取的文件的id以及读取的大小`req_n`. **所以，在进程与文件系统通信的时候，两者可以共享一个保存了`Fsipc`结构的页面，从而实现参数的传递。** 在`file.c`中，这个`Fspic`结构的变量名为`fspicbuf`.

`devfile_read`函数接收文件id以及读取大小，设置`fspicbuf`中对应的字段，然后调用`fsipc()`。`fsipc`接收两个参数，一个是文件操作的类型`type`，另一个是虚拟地址，这里应该传入`fsipcbuf`, 然后它会调用`ipc_send()`，其中要交换的32位值就是`type`，对于文件读取，`type`置为`FSREQ_READ`。然后继续调用`ipc_recv`等待文件系统响应。当文件系统响应后，把结果依次返回给用户进程。



在服务端，文件系统中的`serve()`会循环调用`ipc_recv()`监听请求，接收到请求之后，它会解析请求中的`type`参数，然后具体分发到对应的handler。`type`和handler的对应关系是：

```c
fshandler handlers[] = {
	// Open is handled specially because it passes pages
	/* [FSREQ_OPEN] =	(fshandler)serve_open, */
	[FSREQ_READ] =		serve_read,
	[FSREQ_STAT] =		serve_stat,
	[FSREQ_FLUSH] =		(fshandler)serve_flush,
	[FSREQ_WRITE] =		(fshandler)serve_write,
	[FSREQ_SET_SIZE] =	(fshandler)serve_set_size,
	[FSREQ_SYNC] =		serve_sync
};

```

调用`ipc_recv()`时，`serve`会传入一个参数`fsreq`，它表示文件系统接收请求进程的共享页，要将共享页映射到虚拟内存的什么位置，`fsreq`的定义是：

```c
// Virtual address at which to receive page mappings containing client requests.
union Fsipc *fsreq = (union Fsipc *)0x0ffff000;
```

也是一个`Fsipc`类型的指针。通过`Fsipc`，文件系统可以接收来自客户进程的参数。

从对应的handler返回后，`serve()`函数也负责调用`ipc_send()`将结果返回给`fsipc`。其中，交换的32位数据就是各个handler的返回状态，例如0表示成功，-E_NO_DISK表示磁盘空间不足等。**在`handler`中，因为文件系统与客户进程共享`fsreq`数据结构，那么它可以将读取出来的文件数据放在`fsreq.readRet`中，这相当于一个块的缓冲区，让用户去读取。** 另外，如果调用的类型是`FSREQ_OPEN`， 还会将文件描述符`Fd`结构所在的页地址放到`pg`中，与用户进程共享。

### serve_set_size()阅读

这个函数是将`fs.c`中的`file_set_size()`函数封装成一个handler，所有的handler调用入口都是`serve`函数。除了`serve_open`以外，所有的handler都只接受两个参数，一个是服务请求者的进程ID（`envid`或者`whom`），另一个是与请求者共享的`fsreq`页，上面保存了调用的参数。

```c
void
serve(void)
{
	uint32_t req, whom;
	int perm, r;
	void *pg;

	while (1) {
		perm = 0;
		req = ipc_recv((int32_t *) &whom, fsreq, &perm);
		if (debug)
			cprintf("fs req %d from %08x [page %08x: %s]\n",
				req, whom, uvpt[PGNUM(fsreq)], fsreq);

		// All requests must contain an argument page
		if (!(perm & PTE_P)) {
			cprintf("Invalid request from %08x: no argument page\n",
				whom);
			continue; // just leave it hanging...
		}

		pg = NULL;
		if (req == FSREQ_OPEN) {
			r = serve_open(whom, (struct Fsreq_open*)fsreq, &pg, &perm);
		} else if (req < ARRAY_SIZE(handlers) && handlers[req]) {
			r = handlers[req](whom, fsreq);
		} else {
			cprintf("Invalid request code %d from %08x\n", req, whom);
			r = -E_INVAL;
		}
		ipc_send(whom, r, pg, perm);
		sys_page_unmap(0, fsreq);
	}
}
```

`serve_set_size`中，`fsreq`union中只包含了`set_size`结构体，可以通过`req->req_fileid`获取文件id，`req->req_size`获取要设置的文件大小。

首先调用`openfile_lookup`来判断该进程是否打开了该文件。**内核维护了系统中所有被打开的文件在`opentab`数组中，而每一个进程也维护了自己打开文件的描述符**。

如果进程未打开文件，不可以调用`set_file_size()`，返回错误码。否则，`openfile_lookup`会将该文件的描述符保存在`o`中，通过`o->o_file`就可以获取该文件的元数据指针。直接调用`file_set_size`并返回返回值即可。`set_size`不需要为用户进程提供文件数据，只需要交换一个操作成功与否的状态信息。

```c
// Set the size of req->req_fileid to req->req_size bytes, truncating
// or extending the file as necessary.
int
serve_set_size(envid_t envid, struct Fsreq_set_size *req)
{
	struct OpenFile *o;
	int r;

	if (debug)
		cprintf("serve_set_size %08x %08x %08x\n", envid, req->req_fileid, req->req_size);

	// Every file system IPC call has the same general structure.
	// Here's how it goes.

	// First, use openfile_lookup to find the relevant open file.
	// On failure, return the error code to the client with ipc_send.
	if ((r = openfile_lookup(envid, req->req_fileid, &o)) < 0)
		return r;

	// Second, call the relevant file system function (from fs/fs.c).
	// On failure, return the error code to the client.
	return file_set_size(o->o_file, req->req_size);
}
```

### serve_read()

`serve_read`处理读取文件的请求。它基于`fs.c`中实现的`file_read`函数，接收进程的id以及调用参数结构`Fsipc`，从文件的当前光标处读取文件的`req_n`个字节，到缓冲区`readRet.ret_buf`中，并更新文件光标的位置。如果成功，返回实际读取的字节数，否则返回错误号码。

它的实现过程是：

- 首先从与客户端共享的调用参数页`ipc`中，获取调用参数。因为是read操作，所以`fsipc`中保存的是`REQ_READ`结构体：

  - ```c
    struct Fsreq_read {
    		int req_fileid;
    		size_t req_n;
    	} read;
    ```

- 返回的时候，要填充`fspic`中的`readRet`结构体中的`ret_buf`，而因为缓冲区的大小只有一页，所以一次devfile_read调用读取的字节数不可以超过4096字节。readRet结构体的定义：

  - ```c
    struct Fsret_read {
    		char ret_buf[PGSIZE];
    	} readRet;
    ```

- 调用`openfile_lookup`找到该打开文件的`openFile`结构体。如果返回值小于0，说明发生错误，返回错误码。

- `OpenFile`结构体中的o_file保存了文件的元数据指针，o_fd指向磁盘上的文件描述符结构`Fd`，而`Fd`保存了fd_offset,表示文件当前的光标位置。

- 调用`file_read()`，它需要四个参数：文件元数据指针，读取数据的缓冲区，读取数据字节数，读取开始的offset。`read()`调用隐式地从文件光标处读取n个字节，所以`offset`参数可以通过`o->o_fd->fd_offset`获得。缓冲区是`ret->ret_buf`。

- 读取失败的话返回错误码，否则将文件光标移动到读取完成的位置。

- 返回实际读取的字节数。

```c
int
serve_read(envid_t envid, union Fsipc *ipc)
{
	struct Fsreq_read *req = &ipc->read;
	struct Fsret_read *ret = &ipc->readRet;
	struct OpenFile *o;
	int r;

	if (debug)
		cprintf("serve_read %08x %08x %08x\n", envid, req->req_fileid, req->req_n);

	// Lab 5: Your code here:
	if((r = openfile_lookup(envid, req->req_fileid, &o))<0)
		return r;
	
	if((r = file_read(o->o_file, ret->ret_buf, req->req_n, o->o_fd->fd_offset))<0)
		return r;
	o->o_fd->fd_offset+=r;		 
	return r;

}
```

### 检查



`file_read`OK， 实现正确。

---

## TODO6:

> ​     在fs/serv.c中实现serve-write，在lib/file.c中实现devfile-write  

### serve_write()

根据注释，serve_write()的功能应该是接收进程的id`envid`和写请求的参数：

```c
struct Fsreq_write {
		int req_fileid;
		size_t req_n;
		char req_buf[PGSIZE - (sizeof(int) + sizeof(size_t))];
	} write;
```

将`req_buf`中的`req_n`个字节的内容写到对应文件的光标开始处。`req_buf`的大小是`PGSIZE-(sizeof(int) + sizeof(size_t))`， 这是为了保证`fsipc`结构可以与页的大小对齐。`req_n`的大小只能小于或等于`req_buf`的固定大小。也就是一次请求不会超过一个块。

`serve_write`可以通过`file_write()`来实现，`file_write`没有大小的限制，传给`file_write`的count大小只要不要超过文件最大大小的范围即可。`file_write`也已经提供了自动扩展文件大小的功能。根据函数的语义，`serve_write()`的实现为：

```c
int
serve_write(envid_t envid, struct Fsreq_write *req)
{
	int r;
	struct OpenFile *o;

	if (debug)
		cprintf("serve_write %08x %08x %08x\n", envid, req->req_fileid, req->req_n);

	// LAB 5: Your code here.
	if((r = openfile_lookup(envid, req->req_fileid, &o))<0)
		return r;
	if((r = file_write(o->o_file, req->req_buf, req->req_n, o->o_fd->fd_offset))<0)
		return r;
	o->o_fd->fd_offset += r;
	return r; 
}

```

### devfile_write()

`devfile_write()`处理用户程序发出的写请求，`req_n`不能超过`req_buf`大小的条件在这个函数中检查。它将文件的id，请求写字节数和缓冲区保存到`fsipcbuf`的`write`结构中，调用`fsipc()`函数发送ipc请求，并共享`fsipcbuf`页传递参数，把`fsipc`的返回值直接返回给用户进程。

decfile_write的实现过程为：

- 判断请求的字节数`n`是否比缓冲区大小（`PGSIZE - (sizeof(int) + sizeof(size_t)`)大，如果超出，设置为缓冲区大小。
- 将`fsipcbuf.write.req_fileid`保存为用户进程请求的文件id。由于用户进程传给`devfile_write`的是文件的描述符指针，所以通过`fd->fd_file.id`来获得该文件id。
- 将`fsipcbuf.write.req_n`设置为请求字节数n（不超过缓冲区大小）
- 将用户传递进来的`buf`中的前`n`个字节，复制到`fsipcbuf.write.req_buf`中，这个复制使用`memmove`。
- 调用`fsipc()`发起请求并返回。 这个调用的类型为写操作`FSREQ_WRITE`， 操作类型将由文件系统的`serve()`接收，分发到对应的`serve_write`函数。它不需要从文件系统那里共享页（因为不需要读取文件的内容），所以共享页虚拟地址为NULL。（`fsipc`会在`ipc_send`的时候与文件系统共享`fsipcbuf`页）。

```c
static ssize_t
devfile_write(struct Fd *fd, const void *buf, size_t n)
{
	// Make an FSREQ_WRITE request to the file system server.  Be
	// careful: fsipcbuf.write.req_buf is only so large, but
	// remember that write is always allowed to write *fewer*
	// bytes than requested.
	// LAB 5: Your code here
        uint32_t max_n = PGSIZE - (sizeof(int) + sizeof(size_t));
        n = n > max_n ? max_n : n;

    	fsipcbuf.write.req_fileid = fd->fd_file.id;
    	fsipcbuf.write.req_n = n;
    	memmove(fsipcbuf.write.req_buf, buf, n);
    	return fsipc(FSREQ_WRITE, NULL);
}
```

### 检查



本次实验所有实现的函数检查都`OK`.



## 总结

文件系统负责文件的管理，它通过索引、链表、连续分配等方式组织文件，将文件内的逻辑地址映射到磁盘上的物理块地址；它通过bitmap、空闲块链表等数据结构维护磁盘上的空闲块，实现对磁盘空闲空间的管理；它直接与设备驱动程序（JOS）或者间接与设备驱动程序（UNIX系统中，设备驱动程序的操作封装为系统调用）交互，并通过逻辑地址到物理地址的转换，为用户提供了方便简单且统一的接口，以及一个直观的文件系统界面。

JOS的文件系统通过bitmap来管理空闲空间，当文件系统的大小为3GB时，bitmap的大小为24块。通过间接索引结构来组织文件，文件元数据中保存索引，来完成文件逻辑地址到磁盘物理地址的转换。文件系统中的每一个文件都有一个id标识，保存在文件描述符中，文件描述符表位于磁盘上，描述了文件的id、是否打开、设备以及偏移位置等。内核维护打开的文件描述符表，该结构可以通过打开的文件id找到对应的文件描述符，并保存了文件的元数据指针等变量。

文件块的读取是按需读取的，只有当一个块被用户请求时，文件系统引用该块对应的虚拟地址，会发生页错误，然后才将该块从磁盘上读取到内存。文件系统拥有IO权限，所以可以访问设备驱动程序，调用它的读和写函数。 文件的修改用了 延迟写回的策略， 修改暂时修改在内存中，只有在必要时，会将PTE_D标记为脏的内存页面写回到磁盘上。

JOS中的文件系统是一个特殊的用户进程，它将磁盘地址映射到自己的虚拟地址空间上进行方便的管理，并通过自定义页错误处理程序完成磁盘块到内存的数据转移。通过IPC进程间通信机制，使得用户进程可以调用定义在文件系统进程内的过程，从而为用户进程提供系统调用的接口。







