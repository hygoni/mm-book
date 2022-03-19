# setup_arch()

In this page, we'll take a look how x86_64 initializes its memory in setup_arch().  
Removed some x86_32/64 ifdefs as we are interested in x86_64.  

```c
/*
 * Determine if we were loaded by an EFI loader.  If so, then we have also been
 * passed the efi memmap, systab, etc., so we should use these data structures
 * for initialization.  Note, the efi init code path is determined by the
 * global efi_enabled. This allows the same kernel image to be used on existing
 * systems (with a traditional BIOS) as well as on EFI systems.
 */
/*
 * setup_arch - architecture-specific boot-time initializations
 *
 * Note: On x86_64, fixmaps are ready for use even before this is called.
 */

void __init setup_arch(char **cmdline_p)
{
	printk(KERN_INFO "Command line: %s\n", boot_command_line);
	boot_cpu_data.x86_phys_bits = MAX_PHYSMEM_BITS;

	/*
	 * If we have OLPC OFW, we might end up relocating the fixmap due to
	 * reserve_top(), so do this before touching the ioremap area.
	 */
	olpc_ofw_detect();

	idt_setup_early_traps();
	early_cpu_init();
	jump_label_init();
	static_call_init();
	early_ioremap_init();

	setup_olpc_ofw_pgd();

	ROOT_DEV = old_decode_dev(boot_params.hdr.root_dev);
	screen_info = boot_params.screen_info;
	edid_info = boot_params.edid_info;
#ifdef CONFIG_X86_32
	apm_info.bios = boot_params.apm_bios_info;
	ist_info = boot_params.ist_info;
#endif
	saved_video_mode = boot_params.hdr.vid_mode;
	bootloader_type = boot_params.hdr.type_of_loader;
	if ((bootloader_type >> 4) == 0xe) {
		bootloader_type &= 0xf;
		bootloader_type |= (boot_params.hdr.ext_loader_type+0x10) << 4;
	}
	bootloader_version  = bootloader_type & 0xf;
	bootloader_version |= boot_params.hdr.ext_loader_ver << 4;

#ifdef CONFIG_BLK_DEV_RAM
	rd_image_start = boot_params.hdr.ram_size & RAMDISK_IMAGE_START_MASK;
#endif
#ifdef CONFIG_EFI
	if (!strncmp((char *)&boot_params.efi_info.efi_loader_signature,
		     EFI32_LOADER_SIGNATURE, 4)) {
		set_bit(EFI_BOOT, &efi.flags);
	} else if (!strncmp((char *)&boot_params.efi_info.efi_loader_signature,
		     EFI64_LOADER_SIGNATURE, 4)) {
		set_bit(EFI_BOOT, &efi.flags);
		set_bit(EFI_64BIT, &efi.flags);
	}
#endif

	x86_init.oem.arch_setup();

	/*
	 * Do some memory reservations *before* memory is added to memblock, so
	 * memblock allocations won't overwrite it.
	 *
	 * After this point, everything still needed from the boot loader or
	 * firmware or kernel text should be early reserved or marked not RAM in
	 * e820. All other memory is free game.
	 *
	 * This call needs to happen before e820__memory_setup() which calls the
	 * xen_memory_setup() on Xen dom0 which relies on the fact that those
	 * early reservations have happened already.
	 */
	early_reserve_memory();

	iomem_resource.end = (1ULL << boot_cpu_data.x86_phys_bits) - 1;
	e820__memory_setup();
	parse_setup_data();

	copy_edd();

	if (!boot_params.hdr.root_flags)
		root_mountflags &= ~MS_RDONLY;
	setup_initial_init_mm(_text, _etext, _edata, (void *)_brk_end);

	code_resource.start = __pa_symbol(_text);
	code_resource.end = __pa_symbol(_etext)-1;
	rodata_resource.start = __pa_symbol(__start_rodata);
	rodata_resource.end = __pa_symbol(__end_rodata)-1;
	data_resource.start = __pa_symbol(_sdata);
	data_resource.end = __pa_symbol(_edata)-1;
	bss_resource.start = __pa_symbol(__bss_start);
	bss_resource.end = __pa_symbol(__bss_stop)-1;

#ifdef CONFIG_CMDLINE_BOOL
#ifdef CONFIG_CMDLINE_OVERRIDE
	strlcpy(boot_command_line, builtin_cmdline, COMMAND_LINE_SIZE);
#else
	if (builtin_cmdline[0]) {
		/* append boot loader cmdline to builtin */
		strlcat(builtin_cmdline, " ", COMMAND_LINE_SIZE);
		strlcat(builtin_cmdline, boot_command_line, COMMAND_LINE_SIZE);
		strlcpy(boot_command_line, builtin_cmdline, COMMAND_LINE_SIZE);
	}
#endif
#endif

	strlcpy(command_line, boot_command_line, COMMAND_LINE_SIZE);
	*cmdline_p = command_line;

	/*
	 * x86_configure_nx() is called before parse_early_param() to detect
	 * whether hardware doesn't support NX (so that the early EHCI debug
	 * console setup can safely call set_fixmap()). It may then be called
	 * again from within noexec_setup() during parsing early parameters
	 * to honor the respective command line option.
	 */
	x86_configure_nx();

	parse_early_param();

	if (efi_enabled(EFI_BOOT))
		efi_memblock_x86_reserve_range();

#ifdef CONFIG_MEMORY_HOTPLUG
	/*
	 * Memory used by the kernel cannot be hot-removed because Linux
	 * cannot migrate the kernel pages. When memory hotplug is
	 * enabled, we should prevent memblock from allocating memory
	 * for the kernel.
	 *
	 * ACPI SRAT records all hotpluggable memory ranges. But before
	 * SRAT is parsed, we don't know about it.
	 *
	 * The kernel image is loaded into memory at very early time. We
	 * cannot prevent this anyway. So on NUMA system, we set any
	 * node the kernel resides in as un-hotpluggable.
	 *
	 * Since on modern servers, one node could have double-digit
	 * gigabytes memory, we can assume the memory around the kernel
	 * image is also un-hotpluggable. So before SRAT is parsed, just
	 * allocate memory near the kernel image to try the best to keep
	 * the kernel away from hotpluggable memory.
	 */
	if (movable_node_is_enabled())
		memblock_set_bottom_up(true);
#endif

	x86_report_nx();

	if (acpi_mps_check()) {
#ifdef CONFIG_X86_LOCAL_APIC
		disable_apic = 1;
#endif
		setup_clear_cpu_cap(X86_FEATURE_APIC);
	}

	e820__reserve_setup_data();
	e820__finish_early_params();

	if (efi_enabled(EFI_BOOT))
		efi_init();

	dmi_setup();

	/*
	 * VMware detection requires dmi to be available, so this
	 * needs to be done after dmi_setup(), for the boot CPU.
	 */
	init_hypervisor_platform();

	tsc_early_init();
	x86_init.resources.probe_roms();

	/* after parse_early_param, so could debug it */
	insert_resource(&iomem_resource, &code_resource);
	insert_resource(&iomem_resource, &rodata_resource);
	insert_resource(&iomem_resource, &data_resource);
	insert_resource(&iomem_resource, &bss_resource);

	e820_add_kernel_range();
	trim_bios_range();
  
	early_gart_iommu_check();

	/*
	 * partially used pages are not usable - thus
	 * we are rounding upwards:
	 */
	max_pfn = e820__end_of_ram_pfn();

	/* update e820 for memory not covered by WB MTRRs */
	if (IS_ENABLED(CONFIG_MTRR))
		mtrr_bp_init();
	else
		pat_disable("PAT support disabled because CONFIG_MTRR is disabled in the kernel.");

	if (mtrr_trim_uncached_memory(max_pfn))
		max_pfn = e820__end_of_ram_pfn();

	max_possible_pfn = max_pfn;

	/*
	 * This call is required when the CPU does not support PAT. If
	 * mtrr_bp_init() invoked it already via pat_init() the call has no
	 * effect.
	 */
	init_cache_modes();

	/*
	 * Define random base addresses for memory sections after max_pfn is
	 * defined and before each memory section base is used.
	 */
	kernel_randomize_memory();

	check_x2apic();

	/* How many end-of-memory variables you have, grandma! */
	/* need this before calling reserve_initrd */
	if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
		max_low_pfn = e820__end_of_low_ram_pfn();
	else
		max_low_pfn = max_pfn;

	high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;

	/*
	 * Find and reserve possible boot-time SMP configuration:
	 */
	find_smp_config();

	early_alloc_pgt_buf();

	/*
	 * Need to conclude brk, before e820__memblock_setup()
	 * it could use memblock_find_in_range, could overlap with
	 * brk area.
	 */
	reserve_brk();

	cleanup_highmap();

	memblock_set_current_limit(ISA_END_ADDRESS);
	e820__memblock_setup();

	/*
	 * Needs to run after memblock setup because it needs the physical
	 * memory size.
	 */
	sev_setup_arch();

	efi_fake_memmap();
	efi_find_mirror();
	efi_esrt_init();
	efi_mokvar_table_init();

	/*
	 * The EFI specification says that boot service code won't be
	 * called after ExitBootServices(). This is, in fact, a lie.
	 */
	efi_reserve_boot_services();

	/* preallocate 4k for mptable mpc */
	e820__memblock_alloc_reserved_mpc_new();

#ifdef CONFIG_X86_CHECK_BIOS_CORRUPTION
	setup_bios_corruption_check();
#endif

	/*
	 * Find free memory for the real mode trampoline and place it there. If
	 * there is not enough free memory under 1M, on EFI-enabled systems
	 * there will be additional attempt to reclaim the memory for the real
	 * mode trampoline at efi_free_boot_services().
	 *
	 * Unconditionally reserve the entire first 1M of RAM because BIOSes
	 * are known to corrupt low memory and several hundred kilobytes are not
	 * worth complex detection what memory gets clobbered. Windows does the
	 * same thing for very similar reasons.
	 *
	 * Moreover, on machines with SandyBridge graphics or in setups that use
	 * crashkernel the entire 1M is reserved anyway.
	 */
	reserve_real_mode();

	init_mem_mapping();

	idt_setup_early_pf();

	/*
	 * Update mmu_cr4_features (and, indirectly, trampoline_cr4_features)
	 * with the current CR4 value.  This may not be necessary, but
	 * auditing all the early-boot CR4 manipulation would be needed to
	 * rule it out.
	 *
	 * Mask off features that don't work outside long mode (just
	 * PCIDE for now).
	 */
	mmu_cr4_features = __read_cr4() & ~X86_CR4_PCIDE;

	memblock_set_current_limit(get_max_mapped());

	/*
	 * NOTE: On x86-32, only from this point on, fixmaps are ready for use.
	 */

#ifdef CONFIG_PROVIDE_OHCI1394_DMA_INIT
	if (init_ohci1394_dma_early)
		init_ohci1394_dma_on_all_controllers();
#endif
	/* Allocate bigger log buffer */
	setup_log_buf(1);

	if (efi_enabled(EFI_BOOT)) {
		switch (boot_params.secure_boot) {
		case efi_secureboot_mode_disabled:
			pr_info("Secure boot disabled\n");
			break;
		case efi_secureboot_mode_enabled:
			pr_info("Secure boot enabled\n");
			break;
		default:
			pr_info("Secure boot could not be determined\n");
			break;
		}
	}

	reserve_initrd();

	acpi_table_upgrade();
	/* Look for ACPI tables and reserve memory occupied by them. */
	acpi_boot_table_init();

	vsmp_init();

	io_delay_init();

	early_platform_quirks();

	early_acpi_boot_init();

	initmem_init();
	dma_contiguous_reserve(max_pfn_mapped << PAGE_SHIFT);

	if (boot_cpu_has(X86_FEATURE_GBPAGES))
		hugetlb_cma_reserve(PUD_SHIFT - PAGE_SHIFT);

	/*
	 * Reserve memory for crash kernel after SRAT is parsed so that it
	 * won't consume hotpluggable memory.
	 */
	reserve_crashkernel();

	memblock_find_dma_reserve();

	if (!early_xdbc_setup_hardware())
		early_xdbc_register_console();

	x86_init.paging.pagetable_init();

	kasan_init();

	/*
	 * Sync back kernel address range.
	 *
	 * FIXME: Can the later sync in setup_cpu_entry_areas() replace
	 * this call?
	 */
	sync_initial_page_table();

	tboot_probe();

	map_vsyscall();

	generic_apic_probe();

	early_quirks();

	/*
	 * Read APIC and some other early information from ACPI tables.
	 */
	acpi_boot_init();
	x86_dtb_init();

	/*
	 * get boot-time SMP configuration:
	 */
	get_smp_config();

	/*
	 * Systems w/o ACPI and mptables might not have it mapped the local
	 * APIC yet, but prefill_possible_map() might need to access it.
	 */
	init_apic_mappings();

	prefill_possible_map();

	init_cpu_to_node();
	init_gi_nodes();

	io_apic_init_mappings();

	x86_init.hyper.guest_late_init();

	e820__reserve_resources();
	e820__register_nosave_regions(max_pfn);

	x86_init.resources.reserve_resources();

	e820__setup_pci_gap();

#ifdef CONFIG_VT
#if defined(CONFIG_VGA_CONSOLE)
	if (!efi_enabled(EFI_BOOT) || (efi_mem_type(0xa0000) != EFI_CONVENTIONAL_MEMORY))
		conswitchp = &vga_con;
#endif
#endif
	x86_init.oem.banner();

	x86_init.timers.wallclock_init();

	/*
	 * This needs to run before setup_local_APIC() which soft-disables the
	 * local APIC temporarily and that masks the thermal LVT interrupt,
	 * leading to softlockups on machines which have configured SMI
	 * interrupt delivery.
	 */
	therm_lvt_init();

	mcheck_init();

	register_refined_jiffies(CLOCK_TICK_RATE);

#ifdef CONFIG_EFI
	if (efi_enabled(EFI_BOOT))
		efi_apply_memmap_quirks();
#endif

	unwind_init();
}
```

