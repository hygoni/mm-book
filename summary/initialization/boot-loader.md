# Boot Loader

When you turn on x86 computer, the first instruction executed is in [reset vector](https://en.wikipedia.org/wiki/Reset\_vector), which is located at FFFFFFF0h (16 bytes below 4 GB). [\[link\]](https://stackoverflow.com/questions/9210296/software-initialization-code-at-0xfffffff0h) The address of reset vector does not belong to RAM, but  it is hardwired to ROM, or flash memory. When CPU starts executing instructions in reset vector, the firmware called [BIOS ](https://en.wikipedia.org/wiki/BIOS)is running. The job of BIOS is to test and initialize hardwares (including RAM) and to load boot loader.



[Boot loader](https://en.wikipedia.org/wiki/Bootloader), like GRUB or LILO, is a program that is responsible for booting the computer. It loads kernel into memory and passes control to kernel. Also, it gives some information needed for kernel to boot. The specification for boot protocol is documented at [here.](https://www.kernel.org/doc/html/latest/x86/boot.html) It is job of BIOS to find and load the boot loader. the boot loader is located in [Master Boot Record](https://en.wikipedia.org/wiki/Master\_boot\_record#BIOS\_to\_MBR\_interface), which is first sector in hard drives. But as the size of a sector is just 512 bytes, it is not enough; so usually the image in MBR jumps to another image, that contains full boot loader.

### Memory Layout

Below is memory layout for modern bzImage kernel with boot protocol version >= 2.02.

```
              ~                        ~
              |  Protected-mode kernel |
      100000  +------------------------+
              |  I/O memory hole       |
      0A0000  +------------------------+
              |  Reserved for BIOS     |      Leave as much as possible unused
              ~                        ~
              |  Command line          |      (Can also be below the X+10000 mark)
      X+10000 +------------------------+
              |  Stack/heap            |      For use by the kernel real-mode code.
      X+08000 +------------------------+
              |  Kernel setup          |      The kernel real-mode code.
              |  Kernel boot sector    |      The kernel legacy boot sector.
      X       +------------------------+
              |  Boot loader           |      <- Boot sector entry point 0000:7C00
      001000  +------------------------+
              |  Reserved for MBR/BIOS |
      000800  +------------------------+
              |  Typically used by MBR |
      000600  +------------------------+
              |  BIOS use only         |
      000000  +------------------------+

... where the address X is as low as the design of the boot loader permits.
```

According to boot protocol, the protected-mode kernel is always loaded at address 0x100000, and real-mode kernel can be loaded any address between 0 and 0x10000. The exact address is decided by boot loader.

