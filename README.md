# Hello!
This is a tutorial for OS development, meant to help guide you through all the things you will need to know on your journey!

You are expected to know how to implement the things that are explained, but simple code examples will be given to give you an idea of what to do if you do not understand the text.

If you have trouble or any questions, you can join the [OSdev discord](https://discord.gg/RnCtsqD) and we will do our best to help you and provide answers to your questions.

The [tools section](tools/chapter.md) of this tutorial assumes you are on some linux distro, and that you want to target the x86_64 architecture. If you want to use some other OS to develop, you may need some extra steps that we will not include here. If you want to target a different architecture with your OS there are many, many things that will be different that will not be included.

There also may be language specific setup that will need to be done at runtime or for compile time if you don't use C.

This tutorial will not cover how to make your own bootloader, as there are already many good ones out there (such as [qloader2](https://github.com/qloader2/qloader2), which this tutorial will use) and developing the kernel is most of the fun. :)

## Table of conents

### Tools setup
[Setting up your tools](tools/chapter.md)

### Kernel
[Chapter 1: Loading the kernel and getting code execution](1/chapter.md)

[Chapter 2: Loading the GDT](2/chapter.md)

[Chapter 3: Getting debugging output](3/chapter.md)