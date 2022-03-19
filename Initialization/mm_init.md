# Initialization

This page describes how memory is initialized in x86_64. This is highly architecture-specific.    
And there are tons of initializations codes like ACPI, SMP, tracing, cgroups, ACPI, ... etc.  
We'll get buried in the code if we analyze all of them. I tried to explain what is important in terms of memory management.  

## start_kernel()

start_kernel() in init/main.c is an entrypoint to kernel. every architecture-specific initialization code jumps to start_kernel().  

Below is simplified overview of start_kernel(). It's self-explanatory.  

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

## setup_arch()


## mm_init()

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
        mem_init();
        mem_init_print_info();
        kmem_cache_init();

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