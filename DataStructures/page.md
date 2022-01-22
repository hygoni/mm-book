## Prequisites

This document gives a brief description of virtual memory and paging. If you find this confusing, you need get familiar with these concepts first.

## What is a page?

![Memory Management Unit, Wikipedia](https://upload.wikimedia.org/wikipedia/commons/thumb/d/dc/MMU_principle_updated.png/800px-MMU_principle_updated.png)

Most of modern processors has a Memory Management Unit, which translates virtual address into physical address. If the processors has MMU, processes (including kernel threads) use virtual address to access its main memory. a memory management technology used here is "Paging".  
in paging, the smallest block of memory is called a "page", which is 4KB or 8KB. if you have 4MB of RAM and the page size is 4KB, your operating system manages 1024 pages.

## The view of a process

![Virtual Address to Physical Address mapping, Wikipedia](https://upload.wikimedia.org/wikipedia/commons/thumb/3/32/Virtual_address_space_and_physical_address_space_relationship.svg/440px-Virtual_address_space_and_physical_address_space_relationship.svg.png)

in the view of a process, the process uses __whole virtual address space on the system.__ Note that "virtual address space" is usally much larger than the amount of actual memory.  this is possible because operating system manages mapping between virtual address and physical address. It is possible two processes to access same virtual address, but the underlying physical address will be different. (If two process do not share a page, like threads do.)

## Page Table

To map virtual address to physical address, there should be some structures that has mapping information. That's called "Page Table". Let's __imagine__ what a page table should look like. Let's assume we have a 32 bit machine with 4GB of virtual address space (with page 4K size so we have 2^20 pages). With a naive implementation, we can build an array that has 2^20 pointers. in this implementation, the page table is 4MB. Because each processes should have its own page table, the cost is too high. simple fork() system call has to copy 4MB of page table. In 64bit machines, it gets worse because it has larger than terabytes of virtual address space.
