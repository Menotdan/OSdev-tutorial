# Setting up your development enviroment


A development enviroment is an enviroment that allows you to easily test and edit your code in as little time as possible. It also makes building your code convienient, because you can have it managed automatically.

In this tutorial we will use make as our build system, and you can use any IDE/text editor.

The first thing you will need is a cross compiler. This is because the compiler on your machine (for example gcc) compiles with the standard library and other unneeded things that will cause problems trying to run on a machine that doesn't have an OS running on it yet.

You can get a cross compiler using [this script](make_toolchain.sh). You will want to change the `TARGET` variable in the script to x86_64-elf, because you want to have a compiler that compiles to a bare elf file for the x86_64 architecture.

You should make a folder somewhere to run that script from, since it will download binutils and gcc and build them in the directory that it is run from.

Next you will need to add the toolchain binaries to the `PATH` using your `.bashrc` if you want to use the toolchain from anywhere. The binaries should be located in `script_directory/x86_64-elf-cross/bin`, where `script_directory` is where you ran the script from, assuming that the build did not fail for some reason.

Next you should install QEMU from your distro's package manager, so you can run your OS in a virtual machine which is easy to manage and debug. You can likely figure out how to do this on your own, and there is instructions on how to on the [QEMU website](https://www.qemu.org/download/).

Next you should set up a Makefile to build all of the files in your OS using your newly acquired toolchain into one elf file, that can be loaded by the bootloader.

You will want to use these CFLAGS and your new cross compiler:
```
CC = x86_64-elf-gcc
CFLAGS = -g -fno-pic               \
    -mno-sse                       \
    -mno-sse2                      \
    -mno-mmx                       \
    -mno-80387                     \
    -mno-red-zone                  \
    -mcmodel=kernel                \
    -ffreestanding                 \
    -fno-stack-protector           \
    -O2                            \
    -fno-omit-frame-pointer
```

And you can build each C file into an object file like this:
```makefile
%.o: %.c
	${CC} ${CFLAGS} -I src -c $< -o $@
```

And to make an assembly file into an object file you can use this rule:
```makefile
%.o: %.asm
	nasm -f elf64 -o $@ $<
```

Then you can make a list of object files like this
```makefile
OBJ = ${C_SOURCES:.c=.o} ${NASM_SOURCES:.asm=.o}
```

And to gather a list of C sources and assembly sources you can do something like this:
```makefile
C_SOURCES = $(shell find src/ -type f -name '*.c')
NASM_SOURCES = $(shell find src/ -type f -name '*.asm')
```
And the same can be used to find any kind of file in your src directory (asssuming you use one, you can name it something else or just not use one if you wish).

Now you should make a rule to build all that into an elf file, you can use this command to do so:
```makefile
${CC} -Wl,-z,max-page-size=0x1000 -nostdlib -o $@ -T linker.ld ${OBJ}
```

You will also need a linker.ld, which will be covered in the first chapter of the tutorial.

Now for setting up [qloader2](https://github.com/qloader2/qloader2) (the bootloader that will be used in this tutorial), you can just download a copy of the [binary](https://github.com/qloader2/qloader2/blob/master/qloader2.bin) and the [qloader2-install script](https://github.com/qloader2/qloader2/blob/master/qloader2-install). And then you can use the script later to add qloader2 to your disk image file.

To make a disk image file for qloader2, you will first need to format a disk image with one of these filesystems: FAT32, ext2, or [echfs](https://github.com/qword-os/echfs). Make sure the disk image has an MBR so that `qloader2-install` will work properly and not destroy the filesystem.

You can create the MBR like this:
```
parted -s OS_Image.img mklabel msdos
parted -s OS_Image.img mkpart primary 1 100%
```
where OS_Image is the name of your OS image, which you can create like this:
```
dd if=/dev/zero of=OS_Image.img bs=1M count=30
```
Where count is the number of MiB you want the disk image to be.
This simply reads 30 blocks of 1 MiB from `/dev/zero` (which is just a file that returns a bunch of null bytes when you read from it), and then writes them to the image file, which `dd` will create if it needs to.

Then you should create a filesystem, and mount it to a loopback device.
To make a ext2 filesystem on partition 1 (where the kernel and `qloader2.cfg` will live) you can do this:
```
mkdir mountpoint
sudo losetup -Pf --show OS_Image.img > loopback_dev
sudo partprobe `cat loopback_dev`
sudo mkfs.ext2 `cat loopback_dev`p1
sudo mount `cat loopback_dev`p1 mountpoint
```
This just sets up a loopback device and stores its name in the `loopback_dev` file, then sets up an ext2 filesystem, then mounts partition 1 to the `mountpoint` folder.


Then you just need to move the OS elf file to the filesystem (mounted at `mountpoint` so just copy the files in there) and create a `qloader2.cfg` which will tell qloader2 how to load your kernel.

An example `qloader2.cfg` could look like this:
```
TIMEOUT=5

:OS
KERNEL_PARTITION=0
KERNEL_PATH=kernel.elf
KERNEL_PROTO=stivale
KERNEL_CMDLINE=
```
`TIMEOUT=5` tells qloader2 to wait 5 seconds before autobooting.

`:OS` Makes a new menu entry with the name "OS"

`KERNEL_PARTITION=0` Tells it that the kernel is on partition 0.

`KERNEL_PATH` Is the path where the kernel is located on the filesystem.

`KERNEL_PROTO` Is the boot protocol to use, and the most supported one in qloader2 is stivale, and it happens to also be the easiest to use (In my opinion).

`KERNEL_CMDLINE` are the options that get passed to the kernel. (Note this can be omitted)

You can learn more about these options [here](https://github.com/qloader2/qloader2/blob/master/CONFIG.md), and [here](https://github.com/qloader2/qloader2/blob/master/README.md).

Now for cleaning up the mountpoint and loopback device:
```
sync
sudo umount mountpoint/
sudo losetup -d `cat loopback_dev`
rm -rf mountpoint loopback_dev
```
And you should have an image!

(Remember, you should automate all of this in a makefile, create a rule to make your image, and add your kernel, etc)

Then once you have the `qloader2.cfg` and have cleaned up the mountpoint and loopback-dev files, you can simply run the `qloader2-install` script, which should be simple enough to use.

`./qloader2-install <qloader2 binary path> <device / image to install to>`

So just install the `qloader2.bin` to the `OS_Image.img` and you should be ready to boot, though there isn't a kernel yet, so we will have to make oen.

Now to run the kernel with QEMU and KVM you can do something like this:
```
sudo qemu-system-x86_64 -machine q35 -enable-kvm -smp 1 -cpu host -d cpu_reset -no-shutdown -no-reboot -serial stdio -m 2G -boot menu=on -hda OS_Image.img
```

This will start QEMU with KVM (which is why it needs `sudo`) using 1 core, and it gives 2 GiB of memory to the virtual guest.

If you want to start QEMU without KVM (for checking for faults or whatever) you can use this:
```
qemu-system-x86_64 -machine q35 -no-shutdown -no-reboot -serial stdio -m 2G -boot menu=on -hda OS_Image.img
```

And now if you didn't do anything wrong, you should have a working build system, now to write some code for it to build. :)

-[Back to the start](../README.md)-