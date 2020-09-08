# lab2 - Exercise

#### 【2020-04-01】 

建立分支，准备环境，

#### 【2020-04-07】

学习`pmap.c`、`memlayout.h`、`pmap.h`，~~根据自己的理解做了详细注释~~

#### 【2020-04-10】

 [Intel 80386 Reference Manual](https://pdos.csail.mit.edu/6.828/2017/readings/i386/toc.htm) : 5-2 、6-4

### 练习前梳理

引用一张图，图像化展示内存布局(复习一下lab1)：

![](./img/mem_after_boot.png)

对比lab1给的布局(做了一下补充)：

```shell

+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB) // kern code
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |  <- 0x10000 (ELF header)
|                  |
+------------------+  <- 0x00000000


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
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.  JOS user programs map pages temporarily at UTEMP.
 */
```

在进行实验之前，我们通过`boot.S`的`call bootmain`转移到了`bootmain(boot/main.c)`的`((void (*)(void)) (ELFHDR->e_entry))();`然后进入内核`entry.S`的`call i386_init`来到了目前的练习。

在`entry.S`之前，首先，内核使分页能够使用虚拟内存并解决位置依赖性。 用`kern/entrypgdir.c`中的实现映射，静态初始化的页面目录和页面表。 仅映射物理内存的前4M。这样，`entry.S`可以在高地址操作。即：

```c
// Map VA's [0, 4MB) to PA's [0, 4MB)
// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
```

- virtual addresses `0xf0000000` through `0xf0400000` to physical addresses` 0x00000000` through `0x00400000`
- virtual addresses `0x00000000` through `0x00400000` to physical addresses `0x00000000` through `0x00400000`

`Init.c`的预操作：

```c
// kern/init.c
void
i386_init(void)
{
	extern char edata[], end[];

	// Before doing anything else, complete the ELF loading process.
	// Clear the uninitialized global data (BSS) section of our program.
	// This ensures that all static/global variables start out zero.
	memset(edata, 0, end - edata);

	// Initialize the console.
	// Can't call cprintf until after we do this!
	cons_init();

	cprintf("6828 decimal is %o octal!\n", 6828);

	// Lab 2 memory management initialization functions
	mem_init();

	// Drop into the kernel monitor.
	while (1)
		monitor(NULL);
}
```

这里初始化了`bss`段，众所周知不初始化的全局变量在`bss`里声明，并置为`0`。

**到目前为止，我们仅使用`entry_pgdir`和`entry_pgtable`手动映射了`KERNBASE`开始的4M内存。** 

### Ecercise 1

调用`mem_init();`来到需要我们完善的地方`pmap.c`

这里只设置内核部分，即`addresses >= UTOP`。

首先确定有多少可用内存，更新`npages`，`npages_basemem`。然后使用一个页面来作为页面目录`page directory`，来到`boot_alloc()`。

```c
// nextfree是空闲内存下一个字节的虚拟地址，被初始化到.bss的end。
[...]
if(n > 0) {
  result = nextfree;
  nextfree = ROUNDUP((char*)(nextfree + n), PGSIZE);
  if((uint32_t)nextfree - KERNBASE > (npages * PGSIZE))	//检测是否超出分配范围。
    panic("Out Of Memory!\n");
  return result;
}
else if(n == 0)
  return nextfree;
return NULL;
[···]
```

然后（先跳过在虚拟地址UVPT处形成虚拟页表部分），分配`npages`个`struct PageInfo`元素组成数组，并将其存储在`pages`中。内核使用此数组来跟踪物理页面。

```c
struct PageInfo {
	// Next page on the free list.
	struct PageInfo *pp_link;

	// pp_ref is the count of pointers (usually in page table entries)
	// to this page, for pages allocated using page_alloc.
	// Pages allocated at boot time using pmap.c's
	// boot_alloc do not have valid reference count fields.

	uint16_t pp_ref;
};	//memlayout.h
pages = (struct PageInfo *) boot_alloc(npages * sizeof(struct PageInfo));
memset(pages, 0, npages * sizeof(struct PageInfo));
```

随后需要进入并完成`page_init()`函数：

根据要求把这几项提示完成即可：

```c
	size_t i;
	page_free_list = NULL;

	// 已分配使用部分(extmem)
	int num_alloc = ((uint32_t)boot_alloc(0) - KERNBASE) / PGSIZE;
	//IO部分 384K
	int num_IOhole = (EXTPHYSMEM - IOPHYSMEM) / PGSIZE;

	// page 0
	pages[0].pp_ref = 1;
	// last base memory 
	for(i = 1; i < npages_basemem; i++)
	{
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}
	// alloc / IO hole 
	for(i = npages_basemem; i < npages_basemem + num_IOhole + num_alloc; i++)
  	pages[i].pp_ref = 1;
	// last
	for(; i < npages; i++)
	{
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}
```

此时，`page_free_list`链表指向地址最高的`free page`

最后进入`check_page_free_list()`函数来测试。~~参数`1`指定了包含`entry_pgdir`的`Page Dictionary`。~~

然后是`page_alloc()`、`page_free()`

```c
struct PageInfo *
page_alloc(int alloc_flags)
{
	// Fill this function in
	if(!page_free_list)
		return NULL;
	struct PageInfo *pp = page_free_list;
	if(alloc_flags & ALLOC_ZERO) {
		memset(page2kva(pp), 0, PGSIZE);
	}
	page_free_list = pp->pp_link;
	pp->pp_link = NULL;
	return pp;	
}

void
page_free(struct PageInfo *pp)
{
	// Fill this function in
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
	if(pp->pp_ref != 0)
		panic("pp->pp_ref is nonzero\n");
	if(pp->pp_link)
		panic("pp->pp_link is not NULL\n");

	pp->pp_link = page_free_list;
	page_free_list = pp;
}
```

### Exercise 2

详见[sources](./sources/)

在x86术语中，**虚拟地址**由 段选择器 `segment selector` 和段内的偏移量 `offset` 组成。 **线性地址**是您在分段翻译 `segment translation` 之后, 页面翻译 ` page translation` 之前获得的。 **物理地址**是在段和页面翻译之后最终得到的，最终在硬件总线上抵达RAM的内容。

```shell

           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical

```

虚拟地址 -> 线性地址 -> 物理地址:

![虚拟地址 -> 线性地址](./img/fig5-12.gif)

C指针是虚拟地址的`offset`部分。 **在`boot / boot.S`中，我们设置了全局描述符表（`GDT`），该表通过将所有段基址设置为`0`并将限制设置为`0xffffffff`，有效地禁用了段转换。 因此，`selector`无效，线性地址始终等于虚拟地址的偏移量。** 在lab3中，我们将需要与分段进行更多交互才能设置特权级别，但是对于内存转换，我们可以在整个JOS实验中忽略分段，而只专注于页面转换。

回想一下，在lab1 Part 3中，我们设置了一个简单的`page table`，以便内核可以实际上以其链接地址`0xf0100000`运行，即使该内核实际上已加载到ROM BIOS上方的物理内存中，即`0x00100000`。 该`page table`仅映射了`4MB`的内存。 在本lab中，您将在虚拟地址空间布局中为JOS进行设置，**我们将对其进行扩展以映射从虚拟地址`0xf0000000`开始的前256MB物理内存，并映射虚拟地址空间的许多其他区域。**

从在CPU上执行的代码开始，一旦进入保护模式（我们在`boot / boot.S`中输入了第一件事），就无法直接使用线性或物理地址。 **所有内存引用都被解释为虚拟地址，并由`MMU`转换，这意味着C中的所有指针都是虚拟地址。**

操作系统和MMU是这样配合的：

1. 操作系统在初始化或分配、释放内存时会执行一些指令在物理内存中填写页表，然后用指令设置MMU，告诉MMU页表在物理内存中的什么位置。
2. 设置好之后，CPU每次执行访问内存的指令都会自动引发MMU做查表和地址转换操作，地址转换操作由硬件自动完成，不需要用指令控制MMU去做。

JOS内核通常需要将地址作为不透明值或整数进行操作，而无需在例如物理内存分配器中对其进行反引用。 有时这些是虚拟地址，有时是物理地址。

 为了帮助记录代码，JOS源代码区分了两种情况：`uintptr_t`类型代表不透明的虚拟地址，而`physaddr_t`类型代表物理地址。 这两种类型实际上只是32位整数（`uint32_t`）的同义词，因此编译器不会阻止您将一种类型分配给另一种类型！ 由于它们是整数类型（不是指针），因此，如果您尝试取消引用它们，则编译器将complain。

JOS内核可以通过首先将`uintptr_t`强制转换为指针类型来取消引用。 相反，内核无法明智地取消对物理地址的引用，因为`MMU`会转换所有内存引用。 如果将`physaddr_t`强制转换为指针并取消引用，则可以加载并存储到结果地址（硬件会将其解释为虚拟地址），但可能无法获得所需的内存位置。

| C type       | Address type |
| ------------ | ------------ |
| `T*`         | Virtual      |
| `uintptr_t`  | Virtual      |
| `physaddr_t` | Physical     |

为了将物理地址转换为内核可以实际读写的虚拟地址，内核必须在物理地址上添加`0xf0000000`才能在重映射区域中找到其对应的虚拟地址。 应该使用`KADDR(pa)`进行添加。

**引用计数**

在之后的lab中，通常会同时在多个虚拟地址（或多个环境的地址空间）中映射相同的物理页面。 您将在与物理页面对应的`struct PageInfo`的`pp_ref`字段中保留对每个物理页面的引用数量的计数。 当物理页面的此计数变为`0`时，可以释放该页面，因为它不再使用。 一般来说，这个计数应该等于物理页面在所有页面表中出现在`UTOP`下面的次数（`UTOP`上面的映射主要是在内核启动时设置的，不应该被释放，因此不需要引用他们）。 我们还将使用它来跟踪我们保留到页面目录页面的指针数量，进而跟踪页面目录对页面表页面的引用数量。

使用`page_alloc`时要注意，它返回的页面的引用计数始终为`0`，因此只要您对返回的页面执行某些操作（例如将其插入页面表），`pp_ref`就应该递增。 有时这是由其他函数处理的（例如，`page_insert`），有时调用`page_alloc`的函数必须直接执行。

### Exercise 4

现在，您将编写一组例程来管理页表：插入和删除线性到物理的映射，以及在需要时创建页表页面。

 In the file `kern/pmap.c`, you must implement code for the following functions.

在文件`kern/pmap.c`中，您必须实现以下功能的代码

```c
        pgdir_walk()
        boot_map_region()
        page_lookup()
        page_remove()
        page_insert()
```

`check_page()`, called from `mem_init()`, tests your page table management routines. You should make sure it reports success before proceeding.

##### `pgdir_walk()`

该函数的作用是获得指向线性地址页表项的指针

```c
// Given 'pgdir', a pointer to a page directory, pgdir_walk returns
// a pointer to the page table entry (PTE) for linear address 'va'.
// This requires walking the two-level page table structure.
//
// The relevant page table page might not exist yet.
// If this is true, and create == false, then pgdir_walk returns NULL.
// Otherwise, pgdir_walk allocates a new page table page with page_alloc.
//    - If the allocation fails, pgdir_walk returns NULL.
//    - Otherwise, the new page's reference count is incremented,
//	the page is cleared,
//	and pgdir_walk returns a pointer into the new page table page.
//
// Hint 1: you can turn a PageInfo * into the physical address of the
// page it refers to with page2pa() from kern/pmap.h.
//
// Hint 2: the x86 MMU checks permission bits in both the page directory
// and the page table, so it's safe to leave permissions in the page
// directory more permissive than strictly necessary.
//
// Hint 3: look at inc/mmu.h for useful macros that manipulate page
// table and page directory entries.
//
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	// Fill this function in
	pde_t *pt = pgdir + PDX(va);
	pde_t *pt_addr_v;

	if (*pt & PTE_P)
	{
		pt_addr_v = (pte_t *)KADDR(PTE_ADDR(*pt));
		return pt_addr_v + PTX(va);
	}
	else
	{
		struct PageInfo *pg;
		if (create == 1 && (pg = page_alloc(ALLOC_ZERO)) != 0)
		{
			memset(page2kva(pg), 0, PGSIZE);
			pg->pp_ref++;
			*pt = PADDR(page2kva(pg)) | PTE_U | PTE_W | PTE_P;
			pt_addr_v = (pte_t *)KADDR(PTE_ADDR(*pt));
			return pt_addr_v + PTX(va);
		}
	}
	return NULL;
}
```

##### `boot_map_region()`

将虚拟地址 `[va, va+size)` 映射到物理地址 `[pa, pa+size)`

注释中提到可以使用上面写的 `pgdir_walk` ，获取页表地址，接着将物理地址的值与上权限位赋给页表地址

需要注意这里是 静态映射，不改变 `reference counting`。

关键点是理解 `*page table entry` 结果是对应的物理地址。

```c
// Map [va, va+size) of virtual address space to physical [pa, pa+size)
// in the page table rooted at pgdir.  Size is a multiple of PGSIZE, and
// va and pa are both page-aligned.
// Use permission bits perm|PTE_P for the entries.
//
// This function is only intended to set up the ``static'' mappings
// above UTOP. As such, it should *not* change the pp_ref field on the
// mapped pages.
//
// Hint: the TA solution uses pgdir_walk
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	// Fill this function in
	int offset;
	pte_t *pt;
	for (offset = 0; offset < size; offset += PGSIZE)
	{
		pt = pgdir_walk(pgdir, (void *)va, 1);
		*pt = pa | perm | PTE_P;
		pa += PGSIZE;
		va += PGSIZE;
	}
}
```

##### `page_lookup()`

根据提示，`pgdir_walk` 可以获取对应的页表条目指针；`pa2page` 可以将 页表地址转换为页表。

```c
// Return the page mapped at virtual address 'va'.
// If pte_store is not zero, then we store in it the address
// of the pte for this page.  This is used by page_remove and
// can be used to verify page permissions for syscall arguments,
// but should not be used by most callers.
//
// Return NULL if there is no page mapped at va.
//
// Hint: the TA solution uses pgdir_walk and pa2page.
//
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	// Fill this function in
	pte_t *pte = pgdir_walk(pgdir, va, 0);

	if (pte_store)
		*pte_store = pte;

	if (pte && (*pte & PTE_P))
		return pa2page(PTE_ADDR(*pte));

	return NULL;
}
```

##### `page_remove()`

取消虚拟地址 `va` 的映射，包含以下操作：

- `reference count` 减少
- 如果`reference count` 减至0，需要释放物理页面
- 对应 `va` 的页表条目应被置为0（如果存在）
- 如果从页表中移除了条目，则 `TLB` 应失效

```c
// Unmaps the physical page at virtual address 'va'.
// If there is no physical page at that address, silently does nothing.
//
// Details:
//   - The ref count on the physical page should decrement.
//   - The physical page should be freed if the refcount reaches 0.
//   - The pg table entry corresponding to 'va' should be set to 0.
//     (if such a PTE exists)
//   - The TLB must be invalidated if you remove an entry from
//     the page table.
//
// Hint: The TA solution is implemented using page_lookup,
// 	tlb_invalidate, and page_decref.
//
void page_remove(pde_t *pgdir, void *va)
{
	// Fill this function in
	pte_t *pte;
	struct PageInfo *page = page_lookup(pgdir, va, &pte);
	if (page)
	{
		page_decref(page);
		*pte = 0;
		tlb_invalidate(pgdir, va);
	}
}
```

##### `page_insert()`

完成物理页面`pp` 和虚拟地址 `va` 之间的映射，将页表条目的低12bit设置为 `perm|PTEE_P`

```c
// Map the physical page 'pp' at virtual address 'va'.
// The permissions (the low 12 bits) of the page table entry
// should be set to 'perm|PTE_P'.
//
// Requirements
//   - If there is already a page mapped at 'va', it should be page_remove()d.
//   - If necessary, on demand, a page table should be allocated and inserted
//     into 'pgdir'.
//   - pp->pp_ref should be incremented if the insertion succeeds.
//   - The TLB must be invalidated if a page was formerly present at 'va'.
//
// Corner-case hint: Make sure to consider what happens when the same
// pp is re-inserted at the same virtual address in the same pgdir.
// However, try not to distinguish this case in your code, as this
// frequently leads to subtle bugs; there's an elegant way to handle
// everything in one code path.
//
// RETURNS:
//   0 on success
//   -E_NO_MEM, if page table couldn't be allocated
//
// Hint: The TA solution is implemented using pgdir_walk, page_remove,
// and page2pa.
//
int page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
	// Fill this function in
	pte_t *pte = pgdir_walk(pgdir, va, 1);

	if (!pte)
		return -E_NO_MEM;

	if (*pte & PTE_P)
	{
		if (PTE_ADDR(*pte) == page2pa(pp))
		{
			tlb_invalidate(pgdir, va);
			pp->pp_ref--;
		}
		else
			page_remove(pgdir, va);
	}

	*pte = page2pa(pp) | perm | PTE_P;
	pp->pp_ref++;
	pgdir[PDX(va)] |= perm;
	return 0;
}
```

### Exercise 5

JOS将处理器的32位线性地址空间分为两部分。 我们将在实验3中开始加载和运行的用户环境（进程）将控制下部的布局和内容，而内核始终保持对上部的完全控制。 分界线由inc / memlayout.h中的ULIM符号任意定义，为内核保留了大约256MB的虚拟地址空间。 这就解释了为什么我们需要在实验1中为内核提供如此高的链接地址：否则，内核的虚拟地址空间中就没有足够的空间可以同时在其下面的用户环境中进行映射。

#### 权限和故障隔离

Permissions and Fault Isolation

由于内核和用户内存都存在于每个环境的地址空间中，因此我们必须在x86页表中使用权限位，以允许用户仅访问地址空间的用户部分。 否则用户代码中的bug可能会覆盖内核数据，从而导致崩溃或更微妙的故障; 或者也可能窃取其他环境的私有数据。 请注意，可写权限位`PTE_W`会对用户和内核代码均有效！

`ULIM`以上内存用户没有任何权限，而内核能够拥有读写权限。 对于地址范围`[UTOP，ULIM)`，内核和用户环境都具有相同的权限：它们可以读取但不能写入此地址范围。 此范围的地址用于将某些内核数据结构以只读方式暴露给用户环境。 最后，`UTOP`下面的地址空间供用户环境使用; 用户环境将设置访问此内存的权限。

#### 初始化内核地址空间

Initializing the Kernel Address Space

现在你将设置`UTOP`上方的地址空间——地址空间的内核部分。 `inc/memlayout.h`显示了你应该使用的布局。 您将使用刚刚编写的函数来设置适当的线性到物理的映射。

**Exercise 5.** Fill in the missing code in `mem_init()` after the call to `check_page()`.

Your code should now pass the `check_kern_pgdir()` and `check_page_installed_pgdir()` checks.



- 从 `UPAGES` 开始，往上 `PTSIZE` 范围的地址空间，可以看到其对用户和内核均为只读

  ```c
  boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages), PTE_U);
  ```

- `bootstack`: 由注释可知，该部分的空间范围为`[KSTACKTOP-KSTKSIZE, KSTACKTOP)`，即从 `KSTACKTOP-KSTKSIZE` 开始往上的 `KSTKSIZE` 大小的空间，可以看到内核具有读写权限，用户没有访问权限

  ```c
  boot_map_region(kern_pgdir, KSTACKTOP-KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
  ```

- 所有的物理内存，即从 `KERNBASE` 开始的 `32bit` 能表示的最大地址 232−1232−1, 因此其尺寸恰好为 负数以2的补码表示下的 `-KERNBASE` ； 可以看到对于这部分空间，内核具有读写权限，用户没有权限。

  ```c
  boot_map_region(kern_pgdir, KERNBASE, -KERNBASE, 0, PTE_W);
  ```



Revisit the page table setup in `kern/entry.S` and `kern/entrypgdir.c`. Immediately after we turn on paging, EIP is still a low number (a little over 1MB). At what point do we transition to running at an EIP above KERNBASE? What makes it possible for us to continue executing at a low EIP between when we enable paging and when we begin running at an EIP above KERNBASE? Why is this transition necessary?

从 `kern/entry.S` 中的 `jmp *%eax` 语句开始，系统便跳转到高地址运行。因为在 `entry.S` 中我们的`CR3` 加载的是 `entry_pgdir`，它将物理地址 `[0, 4M)`同时映射到了虚拟地址的 `[0, 4M)` 和`[KERNBASE, KERNBASE+4M)`，所以能保证正常运行。而新的`kern_pgdir`加载后，并没有映射低位的虚拟地址 `[0, 4M)`，所以这一步跳转是必要的。

> *Challenge!* Extend the JOS kernel monitor with commands to:
>
> - Display in a useful and easy-to-read format all of the physical page mappings (or lack thereof) that apply to a particular range of virtual/linear addresses in the currently active address space. For example, you might enter `'showmappings 0x3000 0x5000'` to display the physical page mappings and corresponding permission bits that apply to the pages at virtual addresses 0x3000, 0x4000, and 0x5000.
>
> - Explicitly set, clear, or change the permissions of any mapping in the current address space.
>
> - Dump the contents of a range of memory given either a virtual or physical address range. Be sure the dump code behaves correctly when the range extends across page boundaries!
>
> - Do anything else that you think might be useful later for debugging the kernel. (There's a good chance it will be!)
>
>   [这个摘抄自大佬的\^_\^](https://github.com/Clann24/jos/tree/master/lab2)

```shell
➜  lab git:(lab2) ✗ make grade
make clean
make[1]: Entering directory '/home/mech0n/JOS/lab'
rm -rf obj .gdbinit jos.in qemu.log
make[1]: Leaving directory '/home/mech0n/JOS/lab'
./grade-lab2
make[1]: Entering directory '/home/mech0n/JOS/lab'
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/pmap.c
+ cc kern/kclock.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
ld: warning: section `.bss' type changed to PROGBITS
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 390 bytes (max 510)
+ mk obj/kern/kernel.img
make[1]: Leaving directory '/home/mech0n/JOS/lab'
running JOS: (1.2s)
  Physical page allocator: OK
  Page management: OK
  Kernel page directory: OK
  Page management 2: OK
Score: 70/70
```

### Reference 

[MMU (一）](https://www.cnblogs.com/ikaka/p/3602536.html)

[MIT6.828 | Lab 2: Memory Management](https://www.zhangjc.site/archives-579/)