# Entrypoint to kernel

The entrypoint of kernel is [\_start ](https://elixir.bootlin.com/linux/v5.17.3/C/ident/\_start)symbol at [arch/x86/boot/header.S](https://elixir.bootlin.com/linux/v5.17.3/source/arch/x86/boot/header.S), which jumps to **start\_of\_setup.**

```
              ~                          ~
      X+10000 +--------------------------+
              |  Stack/heap              |      For use by the kernel real-mode code.
      X+08000 +--------------------------+
              ~                          ~
              |  .bss                    |
              |  .signature              |      magic value (0x5a5aaa55) at the end of kernel setup.
              |  .data                   |
              |  .videocards             |
              |  .rodata                 |
              |  .text32                 |
              |  .text                   |
              |  .initdata               |
              |  .inittext               |
              |  .entrypoint             |      The entrypoint of kernel setup
      X+00268 |  end of kernel header    |
      X+00200 |  _start symbol           |      The second sector. jumps to entrypoint.
      X+001F1 |  .header                 |      start of kernel header.
              |  .bstext, .bsdata        |      The kernel legacy boot sector, not used.
      X       +--------------------------+
              |  Boot loader             |      <- Boot sector entry point 0000:7C00
      001000  +--------------------------+
              |  Reserved for MBR/BIOS   |
              ~                          ~
```

The \_start symbol is in the middle of kernel header, at address X + 0x200. Note that the \_start symbol is not the beginning of boot sector. That is because it used to be a bootable image as itself in early days of linux. Now that we use external boot loader, the boot loader should jump to second sector just after boot sector.

Let's take a brief look at memory layout. at the start address of kernel image (X) There is an ancient boot loader which linux used at the early days. And then the kernel header starts at X+001F1. [The kernel header](https://www.kernel.org/doc/html/latest/x86/boot.html#the-real-mode-kernel-header) is information needed for boot and boot loader should provide it.

And then there are some sections including text, data, bss of kernel setup. One notable section is .signature. This stores a magic value, which is required by boot protocol. if the the signature is not set, the kernel stops to boot.

### start\_of\_setup

This summarizes [start\_of\_setup](https://elixir.bootlin.com/linux/v5.17.3/C/ident/start\_of\_setup), where \_start symbol jumps to. It prepares environment before jumping to C code.

#### Stack Initialization

Before jumping to C code, we need to setup stack and initialize BSS section. It's boot loader's job to initialize stack properly and pass pointer. The boot loader also passes **heap\_end\_ptr** parameter using kernel header so that kernel can setup its heap area. But ancient boot loader may pass invalid stack segment register (%ds != %ss). In that case, we initialize our own stack and check if we can use heap.

After setting up stack, it initialize BSS segment to zero and finally, call [main()](https://elixir.bootlin.com/linux/v5.17.3/source/arch/x86/boot/main.c#L134).

```
# Jump to C code (should not return)
	calll	main
```

