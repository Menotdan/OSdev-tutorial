# Chapter 2
## Loading the GDT

In long mode, the GDT is used by the kernel to tell the CPU about what protection rings to run in (user mode, kernel mode, etc) and also about what mode to run in (long mode or protected mode, etc), and even though qloader2 loads a GDT to get into long mode, we should still load our own so we have more control, plus the state of the GDT from qloader2 is undefined.

So lets load a GDT!

A very simple example GDT that can be used in long mode looks like this:
```x86asm
section .data

global GDT64

; The GDT (Global descriptor table)
GDT64:
    .Null: equ $ - GDT64
    dq 0
    .Code: equ $ - GDT64
    dw 0
    dw 0
    db 0
    db 10011010b
    db 00100000b
    db 0
    .Data: equ $ - GDT64
    dw 0
    dw 0
    db 0
    db 10010010b
    db 00000000b
    db 0
GDT_END:
```
Each GDT descriptor looks like this:

![A GDT entry](GDT_Entry.png)

In long mode, the base and limit parts of each GDT descriptor are ignored, and the `flags` and access byte tell the CPU things about the descriptor and what to do when it is loaded into some segment register. In long mode, the only segment register that the CPU really cares about is the `cs` register.

You can load descriptors into most segment registers by just storing the offset into them, but for the `cs` segment register, you need to do something like far jump or `iretq`. In long mode you cannot far jump, so we will `iretq` to reload `cs`.

When a segment register is reloaded with a new value, it reads from that GDT offset and puts the descriptor stored at the offset into a hidden part of the segment register, along with setting any CPU modes or chaging permission rings.

Now that we have the GDT structure, we need to actually load the GDT.

To load the GDT, you use the `lgdt` instruction. The instruction takes a memory operand, which points to a structure describing the GDT. (How big it is and where it starts in memory)

The structure looks like this:
```x86asm
; Remember to put this in the data section as well, so you can put it in right under the GDT, before changing sections

GDT_PTR:
    dw GDT_END - GDT64 - 1    ; Limit
    dq GDT64                  ; Base
```
Now, about using `iretq`, it is a variant of `ret`, in that it pops from the stack to return somewhere, but it pops 5 things off the stack instead of 1, and it is normally used for returning from interrupts. The registers it pops into are these, in the order it pops: `rip`, `cs`, `rflags`, `rsp`, and `ss`.

So we need to push the values we want those registers to be onto the stack.

We can set `rip` to point to another label where we want to continue executing, we can set `cs` to be `0x8`, since that is the offset of the kernel's code segment (we will add user descriptors later) into the GDT, and not `0x0`, because of the null descriptor, for `rflags` we can use `pushf`, and for `rsp` we can just push rsp (but store it before pushing anything else), and we can set `ss` to the offset of the data descriptor, or `0x10` (`0x8` more than `0x8`).

We should finally set all the other segment registers to the data descriptor offset, even though the CPU won't use them in long mode.

So, with the knowledge above, you can change your `exec_start` to something like this (remember to keep the external reference to kmain and the `global exec_start`):
```x86asm
exec_start:
    lgdt [GDT_PTR]

    ; Push the values for iretq
    mov rax, rsp ; Save before we push
    push 0x10       ; ss
    push rax        ; rsp
    pushf           ; rflags
    push 0x8        ; cs
    push run_kernel ; rip
    iretq ; "Return" to the run_kernel
    
run_kernel:
    mov ax, 0x10 ; We can't write to the segment registers directly

    mov ds, ax
    mov es, ax
    mov ss, ax
    mov gs, ax
    mov fs, ax

    ; Run the kernel
    call kmain

    ; Halt if kmain returns
    cli
    hlt
```

And now the GDT should be loaded! :)

-[Back to the start](../README.md)-