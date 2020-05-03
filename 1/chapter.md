# Chapter 1
## Loading the kernel and getting code execution

It's time to test your build system! :D

The first file you will want in your kernel source is an assembly file. This will do the first bit of init for your kernel before you hop into C.

If you use qloader2 with stivale  (bootloader and boot protocol), most of everything will be set up for you. The CPU will already be in 64 bit long mode, paging enabled, several bootstrap pages mapped, and a pointer to the information struct that the bootloader gives you in the `rdi` register. (In the x86_64 SystemV ABI, that register is the first parameter to C functions) Also qloader2 will set up your `rsp` for you, using the information in the stivale header.

The first thing you will want in this assembly file is the stivale header, which goes in it's own section (`.stivalehdr`). An example header looks like this:
```x86asm
section .stivalehdr

stivale_header:
    dq stack.top    ; rsp
    dw 1            ; video mode
    dw 0            ; fb_width
    dw 0            ; fb_height
    dw 0            ; bpp
```
All of these parameters are documented [here](https://github.com/qloader2/qloader2/blob/master/STIVALE.md).

Your stack goes in `.bss`, which is where uninitialized data is put.
```x86asm
section .bss

stack:
    resb 4096
  .top:
```
Remember that the stack grows downwards in x86, so `rsp` points to the top of the stack.

You will also need a `linker.ld`. If you followed the [tools setup tutorial](../tools/chapter.md), it should be in the same directory that the Makefile is in.

An example `linker.ld` looks like this:
```
ENTRY(exec_start)

OUTPUT_FORMAT(elf64-x86-64)

KRNL_VMA_START = 0xFFFFFFFF80000000;

SECTIONS
{
    . = KRNL_VMA_START + 0x100000;

    .stivalehdr : ALIGN(4K) {
        KEEP(*(.stivalehdr))
    }

    .text : ALIGN(4K) {
        *(.text*)
    }

    .rodata : ALIGN(4K) {
        *(.rodata*)
    }

    .data : ALIGN(4K) {
        *(.data*)
    }

    .bss : ALIGN(4K) {
        *(.bss*)
        *(COMMON)
    }
}
```
This linker script links the elf to be loaded at `0xFFFFFFFF80100000` in virtual memory, which qloader2 will do for you when you use stivale.

The `ENTRY` tells qloader2 where to start executing in your kernel. For example, with the example `linker.ld`, that will start executing at the symbol `exec_start`.

Then you can go ahead and start writing some code.
```x86asm
section .text

extern kmain
exec_start:
    call kmain

    ; Halt if the kernel exits for some reason
    cli
    hlt
```
This will just call the kmain function (not defined yet). Later we will want to load a GDT from here, but not yet. (The GDT in long mode is needed later for permissions, but you probably just want to see your kernel do something for now, right?)

For now you can go ahead and create a C file where all your code execution will start.
(Since you are in charge of your project, all the code naming and structure will be up to you, and you can seperate things however you like.)

```c
// Later we will take a pointer to the stivale structure as a parameter
void kmain() {
    while (1) { asm("pause"); } // Do nothing, don't return to the execution starting point
}
```

Now, if your build system works, and you did everything properly, you should be able to build this, add the kernel to your filesystem and add the config file (this is explained in the [tools setup](../tools/chapter.md)), and then run it with QEMU. 

You should get a QEMU window  with qloader2 running in it, and if you boot your OS, it should hang, not very exciting, but it at least shows that your code did something. If you used `-no-shutdown` or `-no-reboot`, check if the window title says that QEMU was paused, if it was, something might have faulted and you should use `-d int` to figure out what happened, then you can ask in the [OSDev discord](https://discord.gg/RnCtsqD) for help.