## What is zone

In previous article, we learned what a NUMA node is. Node is separation by its physical characteristics. Sometimes, we need to separate a node into several zones for special purposes. For example, in x86_64 UMA architecture, there is a single node in system and it may have ZONE_DMA, ZONE_DMA32, ZONE_NORMAL. ZONE_NORMAL is a zone for usual allocations and others are reserved for device's buffer allocation.

## Types of zone

Note that a node can have 1+ zones. In author's arm64 odroid board for example, it has only ZONE_DMA for 4GB RAM and author's x86_64 laptop has ZONE_DMA, ZONE_DMA32, ZONE_NORMAL for 16GB RAM.

### ZONE_NORMAL

This is a normal zone that's used for usual memory allocation.

### ZONE_DMA

Some old devices (ISA devices for example) can only access memory that has 24-bit address. If those devices are used, kernel should be built with CONFIG_ZONE_DMA=y. With CONFIG_ZONE_DMA=y, kernel manages pages in lower 16MB of memory within ZONE_DMA.

Allocating from ZONE_DMA is avoided as possible; See struct pglist_data's node_zonelists for detail.

### ZONE_DMA32

Similar to ZONE_DMA, ZONE_DMA32 is to manage 32-bit addressable pages in a separate zone, called ZONE_DMA32. It is used when CONFIG_ZONE_DMA32=y.

### ZONE_HIGHMEM

It is only used for 32 bit machines. 32 bit address space became too small many years ago. As an example x86's virtual address is separated into lower 3GB (for user space) and upper 1GB (for kernel space). So the kernel could use only 1GB of physical RAM.  

But we cannot map all of 1GB RAM into ZONE_NORMAL. If we do, the kernel cannot access memory outside of ZONE_NORMAL because there's no virtual address spaces left for mapping. So some of kernel address space should be reserved for arbitrary mapping.

In x86, 0 ~ 896MB is used for ZONE_NORMAL and 896MB ~ 1GB is used for ZONE_HIGHMEM. pages in ZONE_NORMAL are mapped in boot process, and pages in ZONE_HIGHMEM are mapped by kmap() and friends when needed.

[For someone who are curious why kernel and user space should share virtual address space, that's because of TLB. Let's say kernel and user space has separate address space, and a virtual address 0xDEADBEEF points physical address 0xF0000000 in user space and 0x10000000 in kernel space. then kernel should flush TLB entries for every system calls. The cost is too high.]  

### ZONE_DEVICE

ZONE_DEVICE is for high speed and high volume device memory.

### ZONE_MOVABLE

I'll cover in later chapters what a migrate type is. To be brief, the page allocator separately maintains pages by its mobility; this separation by mobility decreases memory fragmentation. but when memory is low, it becomes harder to serve high order allocation because unmovable page allocation can take pages from movable pages.

ZONE_MOVABLE is a zone for only allocations that specified __GFP_HIGHUSER & __GFP_MOVABLE (citation needed). This gives better availabilty for higher order allocation.

## struct zone

```c
struct zone {
	/* Read-mostly fields */

	/* zone watermarks, access with *_wmark_pages(zone) macros */
	unsigned long _watermark[NR_WMARK];
	unsigned long watermark_boost;

	unsigned long nr_reserved_highatomic;

	/*
	 * We don't know if the memory that we're going to allocate will be
	 * freeable or/and it will be released eventually, so to avoid totally
	 * wasting several GB of ram we must reserve some of the lower zone
	 * memory (otherwise we risk to run OOM on the lower zones despite
	 * there being tons of freeable ram on the higher zones).  This array is
	 * recalculated at runtime if the sysctl_lowmem_reserve_ratio sysctl
	 * changes.
	 */
	long lowmem_reserve[MAX_NR_ZONES];
```

```c
#ifdef CONFIG_NUMA
	int node;
#endif
```

```c
	struct pglist_data	*zone_pgdat;
	struct per_cpu_pages	__percpu *per_cpu_pageset;
	struct per_cpu_zonestat	__percpu *per_cpu_zonestats;
	/*
	 * the high and batch values are copied to individual pagesets for
	 * faster access
	 */
	int pageset_high;
	int pageset_batch;
```

**zone_pgdat** is to save descriptor address of a node that this zone resides. a "pageset" is per-cpu caches for fast allocation of pages. a page allocator gets page from pageset if possible. if it fails, it try to allocate from zone's free_area.  

```c
#ifndef CONFIG_SPARSEMEM
	/*
	 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
	 * In SPARSEMEM, this map is stored in struct mem_section
	 */
	unsigned long		*pageblock_flags;
#endif /* CONFIG_SPARSEMEM */
```

```c
	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn;

	/*
	 * spanned_pages is the total pages spanned by the zone, including
	 * holes, which is calculated as:
	 * 	spanned_pages = zone_end_pfn - zone_start_pfn;
	 *
	 * present_pages is physical pages existing within the zone, which
	 * is calculated as:
	 *	present_pages = spanned_pages - absent_pages(pages in holes);
	 *
	 * present_early_pages is present pages existing within the zone
	 * located on memory available since early boot, excluding hotplugged
	 * memory.
	 *
	 * managed_pages is present pages managed by the buddy system, which
	 * is calculated as (reserved_pages includes pages allocated by the
	 * bootmem allocator):
	 *	managed_pages = present_pages - reserved_pages;
	 *
	 * cma pages is present pages that are assigned for CMA use
	 * (MIGRATE_CMA).
	 *
	 * So present_pages may be used by memory hotplug or memory power
	 * management logic to figure out unmanaged pages by checking
	 * (present_pages - managed_pages). And managed_pages should be used
	 * by page allocator and vm scanner to calculate all kinds of watermarks
	 * and thresholds.
	 *
	 * Locking rules:
	 *
	 * zone_start_pfn and spanned_pages are protected by span_seqlock.
	 * It is a seqlock because it has to be read outside of zone->lock,
	 * and it is done in the main allocator path.  But, it is written
	 * quite infrequently.
	 *
	 * The span_seq lock is declared along with zone->lock because it is
	 * frequently read in proximity to zone->lock.  It's good to
	 * give them a chance of being in the same cacheline.
	 *
	 * Write access to present_pages at runtime should be protected by
	 * mem_hotplug_begin/end(). Any reader who can't tolerant drift of
	 * present_pages should get_online_mems() to get a stable value.
	 */
	atomic_long_t		managed_pages;
	unsigned long		spanned_pages;
	unsigned long		present_pages;
#if defined(CONFIG_MEMORY_HOTPLUG)
	unsigned long		present_early_pages;
#endif
#ifdef CONFIG_CMA
	unsigned long		cma_pages;
#endif
```

```c
	const char		*name;
```

This is a name of zone. DMA, DMA32, ... and so on.


```c
#ifdef CONFIG_MEMORY_ISOLATION
	/*
	 * Number of isolated pageblock. It is used to solve incorrect
	 * freepage counting problem due to racy retrieving migratetype
	 * of pageblock. Protected by zone->lock.
	 */
	unsigned long		nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
	/* see spanned/present_pages for more description */
	seqlock_t		span_seqlock;
#endif

	int initialized;

	/* Write-intensive fields used from the page allocator */
	ZONE_PADDING(_pad1_)

	/* free areas of different sizes */
	struct free_area	free_area[MAX_ORDER];

	/* zone flags, see below */
	unsigned long		flags;

	/* Primarily protects free_area */
	spinlock_t		lock;

	/* Write-intensive fields used by compaction and vmstats. */
	ZONE_PADDING(_pad2_)

	/*
	 * When free pages are below this point, additional steps are taken
	 * when reading the number of free pages to avoid per-cpu counter
	 * drift allowing watermarks to be breached
	 */
	unsigned long percpu_drift_mark;

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* pfn where compaction free scanner should start */
	unsigned long		compact_cached_free_pfn;
	/* pfn where compaction migration scanner should start */
	unsigned long		compact_cached_migrate_pfn[ASYNC_AND_SYNC];
	unsigned long		compact_init_migrate_pfn;
	unsigned long		compact_init_free_pfn;
#endif

#ifdef CONFIG_COMPACTION
	/*
	 * On compaction failure, 1<<compact_defer_shift compactions
	 * are skipped before trying again. The number attempted since
	 * last failure is tracked with compact_considered.
	 * compact_order_failed is the minimum compaction failed order.
	 */
	unsigned int		compact_considered;
	unsigned int		compact_defer_shift;
	int			compact_order_failed;
#endif

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* Set to true when the PG_migrate_skip bits should be cleared */
	bool			compact_blockskip_flush;
#endif

	bool			contiguous;

	ZONE_PADDING(_pad3_)
	/* Zone statistics */
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
	atomic_long_t		vm_numa_event[NR_VM_NUMA_EVENT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```