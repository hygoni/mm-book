# Entrypoint to kernel

The entrypoint of kernel is [\_start ](https://elixir.bootlin.com/linux/v5.17.3/C/ident/\_start)symbol at [arch/x86/boot/header.S](https://elixir.bootlin.com/linux/v5.17.3/source/arch/x86/boot/header.S), which jumps to start\_of\_setup.

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

and \_start symbol is in the middle of kernel header, at address X + 0x200. Note that the \_start symbol is not the beginning of boot sector. That is because it used to be a bootable image as itself in early days of linux. Now we use external boot loader, the boot loader should jump to second sector just after boot sector,.



According to boot protocol, There are some information that boot loader should provide: [the kernel header](https://www.kernel.org/doc/html/latest/x86/boot.html#the-real-mode-kernel-header) header.  that is needed for booting. and then we have .data and .bss section, where initialized and uninitialized global variables reside at. (and there are some more sections).



One notable section is .signature. This stores a magic value, which is required by boot protocol. if the the signature is not set, the kernel stops to boot.





