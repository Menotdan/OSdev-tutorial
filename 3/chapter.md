# Chapter 3
## Getting debugging output (through serial)

So we have a QEMU window, and we can halt the CPU, but what if we want to talk to the outside world, even if it's just through the terminal (won't work on real hardware)?

We can use a COM port that you can control with QEMU's `-serial` option to print things from the kernel. You just have to send a few values to a few ports to do setup of the serial communication, and then you can just write data to the port and it will be redirected to where you specified with `-serial`.

What are ports? They are another kind of address space like memory, that are accessed using the `in` and `out` instructions (similar to how memory is accessed with `mov`). They are used to control hardware like the serial controller that QEMU emulates, or the programmable interrupt timer (we will add support for that later).

We should write some functions to do port io, remember the naming and directory structure is up to you, but I would at least put this in another file, having all of your code in one file would be really hard to read through. 

(Don't forget to include `<stdint.h>` for the standard integer data types like `uint32_t` and friends)

We can use inline assembly to run the `in` and `out` instructions safely from C, without destroying any of the stuff that GCC stores in registers or whatever. Inline assembly is a bit complicated, although the basics are pretty easy to understand.

To read a byte from a port you would use `in` like this:
```c
uint8_t port_inb(uint16_t port) {
    uint8_t ret;
    asm("in %%dx, %%al" : "=a"(ret) : "d"(port));
    return ret;
}
```

The `in` instruction takes two register parameters, `dx` is the port number, and then `al` is a 1 byte register, which tells `in` to read a single byte (because of the register size), then the `"=a"(ret)` sets `ret` to the `al` register after the code finishes executing. The `"d"(port)` tells it to set the `dx` register to the port number.

The `out` instruction is similar to `in`, and you can output a byte to a port like this:
```c
void port_outb(uint16_t port, uint8_t data) {
    asm("out %%al, %%dx" : : "a"(data), "d"(port));
}
```
The `out` instruction takes two register parameters as well, `al` is 1 byte, so it outputs 1 byte to the port in `dx`, and the `"a"(data)` sets the `al` register to `data` and `"d"(port)` sets `dx` to the port number.

Also you might notice the `%%` before the register names in the assembly, this is because GCC inline assembly is based on AT&T syntax (commonly used in GNU AS) which uses `%` as a prefix; however `%` is used for something else in GCC inline asm ([ASM Wildcards](https://wiki.osdev.org/Inline_Assembly#Wildcards:_How_you_can_let_the_compiler_choose)) so they changed it to `%%`.

Knowing that the register size of the first parameter of `out` and the second parameter of `in` controls how much data is read from the port, you can change the sizes of the register and the size of `data` and `ret` to make `port_outw`, `port_inw`, `port_outd`, and `port_ind`. 

You can change register name used to `ax` (it is 16 bits, or a word) for outw and inw, and the `data` and `ret` should be `uint16_t`. For outd and ind, use `eax` (it is 32 bits, or a double word), and `data` and `ret` should be `uint32_t`.

If this confuses you, this is what the `outw` and `inw` should look like, observe the differences between `inw` and `outw` vs `inb` and `outb`.

```c
uint16_t port_inw(uint16_t port) {
    uint16_t ret;
    asm("in %%dx, %%ax" : "=a"(ret) : "d"(port));
    return ret;
}
```
```c
void port_outw(uint16_t port, uint16_t data) {
    asm("out %%ax, %%dx" : : "a"(data), "d"(port));
}
```
Now, if you notice the differences, you should be able to change the registers and data types to match what was said above with `eax` for the register and `uint32_t` for the data type.

Now that we have port io, we should set up the serial controller. (This should probably go in a new file if you want clean structure) The COM port we want is COM1 and its base io address is `0x3f8`. You can define a macro for COM1's base if you want.

This is the code to setup the port with a 38400 baud rate (you should put this in a function):
```c
port_outb(COM1 + 1, 0); // Disable interrupts
port_outb(COM1 + 3, 1<<7); // Set the DLAB bit so offsets 0 and 1 are for baud rate

/* The magic values for 38400 baud rate */
port_outb(COM1 + 0, 3);
port_outb(COM1 + 1, 0);

port_outb(COM1 + 3, 3); // 8 bits at a time, no parity, one stop bit
port_outb(COM1 + 2, 0xC7); // No idea what this does :/
port_outb(COM1 + 4, 0x0B); // No idea what this does either
```
You can read more about what this does [here](https://wiki.osdev.org/Serial_Ports).


To transmit data, all you have to do is wait until you can (repeatedly check if the transmission buffer is empty), and then send the data to the right port.

To wait for the data to be ready you can do something like this:
```c
while (!(port_inb(COM1 + 5) & 0x20)) { asm("pause"); }
```
This will create a pause loop until the port is ready to send data.

Then, to send a byte, you can do this:
```c
port_outb(COM1, data);
```

Now, to send a string over the COM port, you can do something like this:
```c
void wait_for_transmit() {
    while (!(port_inb(COM1 + 5) & 0x20)) { asm("pause"); }
}

void serial_print(char *string) {
    while (*string != '\0') { // Strings are null terminated
        wait_for_transmit();
        port_outb(COM1, *string);
        string++;
    }
}
```

Now, remember to call your serial setup function in kmain, then you can test the print function with something like this:
```c
serial_print("Hi!\n");
```

If you passed `-serial stdio` to QEMU, it should now print Hi! :D

This can be used later for debugging your code and it is very useful to be able to print things.