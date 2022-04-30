# start\_kernel()

## Initialization

This page describes how memory is initialized in x86\_64. This is highly architecture-specific.\
And there are tons of initializations codes like ACPI, SMP, tracing, cgroups, ACPI, ... etc.\
We'll get buried in the code if we analyze all of them. I tried to explain what is important in terms of memory management.

In this page, I do not explain all details here. Later we'll take a look at memblock, buddy, slab, vmalloc, ... etc.

### start\_kernel()

start\_kernel() in init/main.c is an entrypoint to kernel. every architecture-specific initialization code jumps to start\_kernel(). Below is **simplified overview** of start\_kernel():

```c
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
        char *command_line;

	/* IRQs disabled in early boot */
	local_irq_disable();
        early_boot_irqs_disabled = true;

	/*
	 * architecture-specific initializations including:
	 * 1) initializing page tables, zones and nodes
	 * 2) initializing memblock
	 * 3) parsing early parameters
	 */
	setup_arch(&command_line);

	/* building zonelists and cpu online hooks */
	build_all_zonelists(NULL);
        page_alloc_init();

        /*
	 * Large allocations must be done before memblock returns
	 * all available memory to buddy allocator (page allocator)
	 * because buddy allocator supports sizes up to PAGE_SIZE << (MAX_ORDER - 1).
         */
        setup_log_buf(0);
        vfs_caches_init_early();
        sort_main_extable();
        trap_init();
        mm_init();

	/* Enable IRQs later in boot process*/
        early_boot_irqs_disabled = false;
        local_irq_enable();
	
	/* late initialization of slab */
	kmem_cache_init_late();

	/* Initialize per-cpu pagesets */
        setup_per_cpu_pageset();

	/*
	 * Use interleaving policy for initialization codes
	 * to distribute data structures to various nodes
	 */
        numa_policy_init();

	/* skipping bunch of initializations... */

	/*
	 * Rest of initialization including:
	 * 1) spawning init process
	 * 2) set numa policy to default policy
	 */
        arch_call_rest_init();

}
```

### mm\_init()

```c
/*
 * Set up kernel memory allocators
 */
static void __init mm_init(void)
{
        /*
         * page_ext requires contiguous pages,
         * bigger than MAX_ORDER unless SPARSEMEM.
         */
        page_ext_init_flatmem();
        init_mem_debugging_and_hardening();
        kfence_alloc_pool();
        report_meminit();
        stack_depot_early_init();
```

Above are initialization codes that needs large pages. Only memblock can serve allocations bigger than MAX\_ORDER.

```c
        mem_init();
        mem_init_print_info();
```

Before mem\_init(), kernel uses memblock to allocate memory in early boot process. after mem\_init(), memblock returns all available memory to buddy allocator.

```c
        kmem_cache_init();
```

slab allocator is initialized just after buddy allocator became available.

```c
	/*
	 * page_owner must be initialized after buddy is ready, and also after
	 * slab is ready so that stack_depot_init() works properly
	 */
	page_ext_init_flatmem_late();
	kmemleak_init();
	pgtable_init();
	debug_objects_mem_init();
	
        vmalloc_init();
	/* Should be run before the first non-init thread is created */
	init_espfix_bsp();
	/* Should be run after espfix64 is set up. */
	pti_init();
}
```

Some subsystems that requires slab are initialized after kmem\_cache\_init(), including vmalloc subsystem.

## Summary

1. start\_kernel() is entrypoint to kernel after architecture-specific initialization code.
2. page tables, memblock are initialized in setup\_arch().
3. Linux uses interleave NUMA policy at initialization and than change to default policy.
4. in mm\_init(), memblock returns all available memory to buddy allocator. And then slab and vmalloc is initialized.
