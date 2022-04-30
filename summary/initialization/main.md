# main()

## main()

In previous page, we took a look at header.S. It prepares environment for C code (stack, bss) and then calls [main()](https://elixir.bootlin.com/linux/v5.17.3/source/arch/x86/boot/main.c#L134). We're not going to dig into every details of main(). We'll focus on what is important for memory management.

```c
void main(void)
{
	/* First, copy the boot header into the "zeropage" */
	copy_boot_params();

	/* Initialize the early-boot console */
	console_init();
	if (cmdline_find_option_bool("debug"))
		puts("early console in setup code\n");

	/* End of heap check */
	init_heap();
	
	[...]

	/* Detect memory layout */
	detect_memory();

	[...]

	/* Do the last things and invoke protected mode */
	go_to_protected_mode();
}
```

### Heap Initialization

In header.S, we did not initialize heap yet. We only have stack. in init\_heap()_, we use heap when boot loader provided heap\_end\_ptr._

```c
static void init_heap(void)
{
	char *stack_end;

	if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
		asm("leal %P1(%%esp),%0"
		    : "=r" (stack_end) : "i" (-STACK_SIZE));

		heap_end = (char *)
			((size_t)boot_params.hdr.heap_end_ptr + 0x200);
		if (heap_end > stack_end)
			heap_end = stack_end;
	} else {
		/* Boot protocol 2.00 only, no heap available */
		puts("WARNING: Ancient bootloader, some functionality "
		     "may be limited!\n");
	}
}
```

After init\_heap(), the memory layout will look like below:

```
              ~                        ~
      X+10000 +------------------------+
              |                        |
              |  stack start           |
              |                        |
              |                        |
              |  stack end             |      stack start - STACK_SIZE (1024)
              ~                        ~
              |  heap end              |      
              |                        |
              |                        |
              |                        |
              |  heap start (_end)     |      end of kernel setup, also start of heap
      X+08000 +------------------------+
              |  Kernel setup          |      The kernel real-mode code.
              |  Kernel boot sector    |      The kernel legacy boot sector.
      X       +------------------------+
              ~                        ~

```

Note that start of heap is not initialized yet. Kernel can use heap after RESET\_HEAP(). helpers for heap are defined at [arch/x86/boot/boot.h](https://elixir.bootlin.com/linux/v5.17.3/source/arch/x86/boot/boot.h#L191)

```c
/* Heap -- available for dynamic lists. */
extern char _end[];
extern char *HEAP;
extern char *heap_end;
#define RESET_HEAP() ((void *)( HEAP = _end ))
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))

static inline bool heap_free(size_t n)
{
	return (int)(heap_end-HEAP) >= (int)n;
}
```

### detect\_memory()

The way to detect available memory highly depends on architectures. In x86, kernel asks BIOS for range of available memory. Linux uses E820, E801, and 88 to detect available memory.

```c
void detect_memory(void)
{
	detect_memory_e820();

	detect_memory_e801();

	detect_memory_88();
}
```

### detect\_memory\_e820()

When using functions provided by BIOS, we invoke interrupt using [INT instruction](https://en.wikipedia.org/wiki/BIOS\_interrupt\_call#Interrupt\_table). BIOS serves various functions and we should choose and interrupt number and value of AX register to call a function of BIOS. the name _E820_ comes from the value of AX for the function.

E820 is used to detect ranges of available memory of  the system. it can detect memory that can be addressed using **64bits** (So it can detect memory above 4G). All of modern x86 computers (since 2002) implements this function.

The specification can be found at [INT 15h, AX=E820h - Query System Address Map](http://www.uruk.org/orig-grub/mem64mb.html).

#### struct boot\_e820\_entry

In x86, the system memory is not a big and single continuous area. it is composed of small and big chunks of memory. each chunk is represented by **struct boot\_e820\_entry**

```c
/*
 * The E820 memory region entry of the boot protocol ABI:
 */
struct boot_e820_entry {
	__u64 addr;
	__u64 size;
	__u32 type;
} __attribute__((packed));
```

#### detect\_memory\_e820()

```c
#define SMAP	0x534d4150	/* ASCII "SMAP" */

static void detect_memory_e820(void)
{
	int count = 0;
	struct biosregs ireg, oreg;
	struct boot_e820_entry *desc = boot_params.e820_table;
	static struct boot_e820_entry buf; /* static so it is zeroed */
```

The memory range described by e820 is saved in boot\_params.e820\_table. Kernel initializes its direct map and passes memory to memblock (early memory allocator) according to this information.

```c
	initregs(&ireg);
	ireg.ax  = 0xe820;
	ireg.cx  = sizeof(buf);
	ireg.edx = SMAP;
	ireg.di  = (size_t)&buf;
```

It first sets registers (See [the specifiction](http://www.uruk.org/orig-grub/mem64mb.html)):

* AX is the function code E820
* DI is pointer to buffer.
* CX is size of buffer.
* DX is signature 'SMAP', which means the kernel wants information about memory to be returned in the buffer.
* EBX is zero, which means it's first iteration. after first iteration, it is set to the value previously returned by this function.

```c
	/*
	 * Note: at least one BIOS is known which assumes that the
	 * buffer pointed to by one e820 call is the same one as
	 * the previous call, and only changes modified fields.  Therefore,
	 * we use a temporary buffer and copy the results entry by entry.
	 *
	 * This routine deliberately does not try to account for
	 * ACPI 3+ extended attributes.  This is because there are
	 * BIOSes in the field which report zero for the valid bit for
	 * all ranges, and we don't currently make any use of the
	 * other attribute bits.  Revisit this if we see the extended
	 * attribute bits deployed in a meaningful way in the future.
	 */

	do {
		intcall(0x15, &ireg, &oreg);
		ireg.ebx = oreg.ebx; /* for next iteration... */
```

After the call, BIOS returns:

* EAX is 'SMAP'
* DI is the buffer we passed
* CX is number of bytes consumed by BIOS
* EBX the 'continuation value' that kernel should pass in next iteration.

```c
		/* BIOSes which terminate the chain with CF = 1 as opposed
		   to %ebx = 0 don't always report the SMAP signature on
		   the final, failing, probe. */
		if (oreg.eflags & X86_EFLAGS_CF)
			break;

		/* Some BIOSes stop returning SMAP in the middle of
		   the search loop.  We don't know exactly how the BIOS
		   screwed up the map at that point, we might have a
		   partial map, the full map, or complete garbage, so
		   just return failure. */
		if (oreg.eax != SMAP) {
			count = 0;
			break;
		}
```

If BIOS sets carry flag or do not return SMAP signature, kernel gives up.

```c

		*desc++ = buf;
		count++;
	} while (ireg.ebx && count < ARRAY_SIZE(boot_params.e820_table));

	boot_params.e820_entries = count;
}
```

#### detect\_memory\_e801(), detect\_memory\_88()

When are these used? Is e820 insufficient?

### go\_to\_protected\_mode()

```c
/*
 * Actual invocation sequence
 */
void go_to_protected_mode(void)
{
	/* Hook before leaving real mode, also disables interrupts */
	realmode_switch_hook();

	/* Enable the A20 gate */
	if (enable_a20()) {
		puts("A20 gate not responding, unable to boot...\n");
		die();
	}

	/* Reset coprocessor (IGNNE#) */
	reset_coprocessor();

	/* Mask all interrupts in the PIC */
	mask_all_interrupts();

	/* Actual transition to protected mode... */
	setup_idt();
	setup_gdt();
	protected_mode_jump(boot_params.hdr.code32_start,
			    (u32)&boot_params + (ds() << 4));

```
