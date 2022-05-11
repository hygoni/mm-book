# page table initialization

## Early Page Table

x86's early page table is initialized in [startup_32()](https://elixir.bootlin.com/linux/v5.17-rc8/source/arch/x86/boot/compressed/head_64.S#L202). it initializes page tables used to map 4G of physical memory (even if physical RAM is less than 4G). And it turns on CR4.PAE, EFER.LME, and CR0.PG and CR0.PE. Then kernel is using 4-level paging. In case when kernel wants to enable 5-level paging, it disables paging and then 5-level paging again. In any case, early page table is identity-mapping from 0 to 4G.  


## init_mem_mapping
