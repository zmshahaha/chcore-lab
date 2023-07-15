## 第一部分：内核启动页表

> 思考题 1：请思考多级页表相比单级页表带来的优势和劣势（如果有的话），并计算在 AArch64 页表中分别以 4KB 粒度和 2MB 粒度映射 0～4GB 地址范围所需的物理内存大小（或页表页数量）。

多级页表允许在整个页表结构中出现空洞，极大减少了页表占用的空间大小，但也导致设计较为复杂、内存利用效率降低（有内部碎片）、访问地址需多次访存等缺点。

以 $4 \text{KB}$ 粒度映射时，低 12 位表示页内偏移量，每一级页表的索引占用 9 位，所以一个页表项包含 $2^9$ 个页表项。总共需要 $4 \text{GB} / 4 \text{KB} = 2^{20}$ 个页条目，因此需要 $2^{20} / 2^9 = 2^{11}$ 个 L3 页表，需要 $2^{11} / 2^9 = 2^2$ 个 L2 页表，需要 1 个 L1页表和 1 个 L0页表，因此共需要 $2^{11} + 2^2 + 1 + 1 = 2054$ 个页表页，每一个页表页占用 $4 \text{KB}$，所以需要占用 $2054  \times 4 \text{KB} = 8 \text{MB}$ 物理内存大小。

以 $2 \text{MB}$ 粒度映射时，2 MB 粒度映射时，低 21 位表示页内偏移量，需要 $2^{11}$ L0页表项。假设页表页大小与页大小一致，只需要一个L0页表，若共一级页表，则需2MB,两级则需4MB。

> 练习题 2：请在 `init_boot_pt` 函数的 `LAB 2 TODO 1` 处配置内核高地址页表（`boot_ttbr1_l0`、`boot_ttbr1_l1` 和 `boot_ttbr1_l2`），以 2MB 粒度映射。

模仿低地址配置过程即可

> 思考题 3：请思考在 `init_boot_pt` 函数中为什么还要为低地址配置页表，并尝试验证自己的解释。

由gdb调试可知内核开始运行时是用低地址，删去这段代码后发现无法翻译地址。

> 思考题 4：请解释 `ttbr0_el1` 与 `ttbr1_el1` 是具体如何被配置的，给出代码位置，并思考页表基地址配置后为何需要ISB指令。

``` tool.S: el1_mmu_activate
	/* Write ttbr with phys addr of the translation table */
	adrp    x8, boot_ttbr0_l0
	msr     ttbr0_el1, x8
	adrp    x8, boot_ttbr1_l0
	msr     ttbr1_el1, x8
	isb
```
isb指令是“指令同步屏障”的缩写，在此上下文中用于确保指令的正确排序和同步。在提供的代码片段中，isb指令跟随对转换表基本寄存器（ttbr0_el1和ttbr1_el1）的写入。

修改转换表基寄存器时，必须确保更改在执行任何后续指令之前生效。isb指令充当同步点，确保isb之前的所有指令都完成了对系统状态的影响。

isb指令提供了一个内存屏障，确保在后续指令开始执行之前完成任何先前的内存访问。它确保对转换表基本寄存器的任何未决更改对以下指令可见，防止任何可能影响后续代码正确性的潜在指令重新排序或猜测。

在这种特殊情况下，isb指令有助于确保在执行到可能依赖于更新的转换表的后续指令之前，对转换表基寄存器（ttbr0_el1和ttbr1_el1）所做的更改已经生效。

请注意，isb指令通常用于需要指令或内存访问显式同步的场景，例如修改控制寄存器或在多处理器系统中管理内存一致性时。

## 第二部分：物理内存管理

> 练习题 5：完成 `kernel/mm/buddy.c` 中的 `split_page`、`buddy_get_pages`、`merge_page` 和 `buddy_free_pages` 函数中的 `LAB 2 TODO 2` 部分，其中 `buddy_get_pages` 用于分配指定阶大小的连续物理页，`buddy_free_pages` 用于释放已分配的连续物理页。

引入chunk概念，chunk代表一个order的第一个page，该page代表一个内存块，仅有该page的metadata在freelist里，其余freelist均为空指针

对某些static函数约束其输入

事实上只需保证chunk page的元数据（是否分配及order）正确性即可，但此处也保证了其他page元数据正确性

page metadata中非chunk首page的node为null(是否必要？？)

语义约束：

```static struct page *get_buddy_chunk(struct phys_mem_pool *pool, struct page *chunk)```中，chunk为每个chunk的第一个page，但后面有通过其修改chunk分布的函数，所以不能assert该chunk的node是否连结

```static struct page *split_page(struct phys_mem_pool *pool, u64 order, page *page)```中，order是要alloc的order，当前chunk的order从page中找，总是分配page代表chunk的前```1<<order```个page

```void buddy_free_pages(struct phys_mem_pool *pool, struct page *page)```中page必须是获取page所返回的

实现方式均以结果为导向，先思考分配回收后元数据状态，再进行设置,注意有些函数会返回NULL需要检查

部分联系紧密操作结合为函数，减少出错，如代码中对pool的改动

## 第三部分：页表管理

> 练习题 6：完成 `kernel/arch/aarch64/mm/page_table.c` 中的 `get_next_ptp`、 `query_in_pgtbl`、`map_range_in_pgtbl`、`unmap_range_in_pgtbl` 函数中的 `LAB 2 TODO 3` 部分，后三个函数分别实现页表查询、映射、取消映射操作，其中映射和取消映射以 4KB 页为粒度。

没啥好说的，一步一步映射就行，注意一些比特的设置。

内核使用的是内核页表，当前设置的是进程页表