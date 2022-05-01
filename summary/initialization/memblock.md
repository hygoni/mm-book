# memblock

Before initialization of buddy allocator, there are few way to allocate memory, like heap. But kernel needs not a small amount of memory in boot process. So there is **memblock:** it's a memory allocator in boot process before initialization of buddy allocator.

### struct memblock\_region

```c
/**
 * struct memblock_region - represents a memory region
 * @base: base address of the region
 * @size: size of the region
 * @flags: memory region attributes
 * @nid: NUMA node id
 */
struct memblock_region {
	phys_addr_t base;
	phys_addr_t size;
	enum memblock_flags flags;
#ifdef CONFIG_NUMA
	int nid;
#endif
};
```

memblock's view of system memory is collection of _memory regions_. each region has base address, size, and flags (and maybe NUMA node if CONFIG\_NUMA=y). This is looks very similar to **e820 memory map.**

### **struct memblock\_type**

```c
/**
 * struct memblock_type - collection of memory regions of certain type
 * @cnt: number of regions
 * @max: size of the allocated array
 * @total_size: size of all regions
 * @regions: array of regions
 * @name: the memory type symbolic name
 */
struct memblock_type {
	unsigned long cnt;
	unsigned long max;
	phys_addr_t total_size;
	struct memblock_region *regions;
	char *name;
};
```

As system memory is not a big, single and continuous region, kernel needs array of struct memblock\_region. struct memblock\_type is for this purpose; it's a _collection of memory regions of certain type_. Note that the size of array is not static; it is doubled when the array is full.

### struct memblock

```c
/**
 * struct memblock - memblock allocator metadata
 * @bottom_up: is bottom up direction?
 * @current_limit: physical address of the current allocation limit
 * @memory: usable memory regions
 * @reserved: reserved memory regions
 */
struct memblock {
	bool bottom_up;  /* is bottom up direction? */
	phys_addr_t current_limit;
	struct memblock_type memory;
	struct memblock_type reserved;
};
```

We saw struct memblock\_type is an array of regions. struct memblock represents the whole memory. we have two types of array of regions for the system**:** **reserved** or **memory** (available).

Note that the memblock\_type _grows_ as boot process progresses. The whole memory may not be available at very early stage of boot process. the direction of growing maybe top-down or bottom-up. _(bool bottom\_up)_

_current\_limit_ is the maximum (or minimum) limit of current memblock.

![memblock's view of memory](../../.gitbook/assets/image.png)

struct memblock is statically initialized at [mm/memblock.c](https://elixir.bootlin.com/linux/v5.18-rc4/source/mm/memblock.c#L111):

```c
struct memblock memblock __initdata_memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		= 1,	/* empty dummy entry */
	.memory.max		= INIT_MEMBLOCK_REGIONS,
	.memory.name		= "memory",

	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,	/* empty dummy entry */
	.reserved.max		= INIT_MEMBLOCK_RESERVED_REGIONS,
	.reserved.name		= "reserved",

	.bottom_up		= false,
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
```

### APIs

There are various functions memblock provides. two basic primitives are: memblock\_add\_range() and memblock\_remove\_range().

#### memblock\_add\_range()

```c
/**
 * memblock_add_range - add new memblock region
 * @type: memblock type to add new region into
 * @base: base address of the new region
 * @size: size of the new region
 * @nid: nid of the new region
 * @flags: flags of the new region
 *
 * Add new memblock region [@base, @base + @size) into @type.  The new region
 * is allowed to overlap with existing ones - overlaps don't affect already
 * existing regions.  @type is guaranteed to be minimal (all neighbouring
 * compatible regions are merged) after the addition.
 *
 * Return:
 * 0 on success, -errno on failure.
 */

```

This function inserts new region and it allows new region to overlap with existing region.

```c
static int __init_memblock memblock_add_range(struct memblock_type *type,
				phys_addr_t base, phys_addr_t size,
				int nid, enum memblock_flags flags)
{
	bool insert = false;
	phys_addr_t obase = base;
	phys_addr_t end = base + memblock_cap_size(base, &size);
	int idx, nr_new;
	struct memblock_region *rgn;

	if (!size)
		return 0;

	/* special case for empty array */
	if (type->regions[0].size == 0) {
		WARN_ON(type->cnt != 1 || type->total_size);
		type->regions[0].base = base;
		type->regions[0].size = size;
		type->regions[0].flags = flags;
		memblock_set_region_node(&type->regions[0], nid);
		type->total_size = size;
		return 0;
	}
```

When the array is empty there is noting complex: just initialize type->regions\[0].

```c
repeat:
	/*
	 * The following is executed twice.  Once with %false @insert and
	 * then with %true.  The first counts the number of regions needed
	 * to accommodate the new area.  The second actually inserts them.
	 */
```

According to the comment, this is executed twice. First round counts number of regions and second round actually inserts.

```c
	base = obase;
	nr_new = 0;

	for_each_memblock_type(idx, type, rgn) {
		phys_addr_t rbase = rgn->base;
		phys_addr_t rend = rbase + rgn->siz
```

for\_each\_memblock\_type iterates every region in memblock\_type. rbase, rend is base and end address of a region.

```c

		if (rbase >= end)
			break;
```

if rbase >= end, \[base, end) is new region without overlap. in that case just break the loop and insert.

```c
		if (rend <= base)
			continue;
```

if rend <= base, this region is way before new region. continue and check next region.

```c
		/*
		 * @rgn overlaps.  If it separates the lower part of new
		 * area, insert that portion.
		 */
		if (rbase > base) {
#ifdef CONFIG_NUMA
			WARN_ON(nid != memblock_get_region_node(rgn));
#endif
			WARN_ON(flags != rgn->flags);
```

if execution gets here, the two regions may look like:

![](<../../.gitbook/assets/image (1).png>)

```c
			nr_new++;
			if (insert)
				memblock_insert_region(type, idx++, base,
						       rbase - base, nid,
						       flags);
		}
```

Then it inserts (in second round) blue-colored region in image below into memblock\_type. Note that memblock did not touch existing region. if two regions overlap, it inserts  lower parts of new region that does not belong to existing region.

![](<../../.gitbook/assets/image (7).png>)

```c
		/* area below @rend is dealt with, forget about it */
		base = min(rend, end);
	}
```

Then base is updated to:

![](<../../.gitbook/assets/image (6).png>)

```c

	/* insert the remaining portion */
	if (base < end) {
		nr_new++;
		if (insert)
			memblock_insert_region(type, idx, base, end - base,
					       nid, flags);
	}
```

In the example case memblock did not insert region \[updated base, end). insert it.

```c
	if (!nr_new)
		return 0;
```

If nr\_new is zero, that means new regions was just subset of existing region. Then just return.

```c
	/*
	 * If this was the first round, resize array and repeat for actual
	 * insertions; otherwise, merge and return.
	 */
	if (!insert) {
		while (type->cnt + nr_new > type->max)
			if (memblock_double_array(type, obase, size) < 0)
				return -ENOMEM;
		insert = true;
		goto repeat;
```

If it was first round, resize array (if needed) and go to next round. Note that nr\_new can be bigger than 2 when new region looks like:

![](<../../.gitbook/assets/image (5).png>)

```c
	} else {
		memblock_merge_regions(type);
		return 0;
	}
}
```

If it was second round, merge neighbor regions and then return. Note that there is no two regions that are neighboring after memblock\_add\_range(). they are merged before returning the function.

#### memblock\_isolate\_range()





### References

{% embed url="https://lwn.net/Articles/761215" %}
