# Chapter 4
## Getting Stivale info and making a PMM

We now have serial printing and a GDT setup, but now what if we want to allocate memory? For that we should use a PMM (physical memory manager) to allocate pages of memory for the kernel. In this tutorial we will be implementing a bitmap PMM, which uses 1 bit to represent 1 page in memory.

First we will need to get a map of memory. This will tell us how much memory there is, what parts are usable by us and what parts are reserved by the BIOS or by hardware for MMIO (memory mapped IO). Luckily, qloader2 provides us with the memory map. All we need to do to get it is take a pointer to the stivale struct in `kmain`, and to do that we need to define the structures so gcc knows how to access it's members.

A header for the structures can be found [here](stivale.h).

Now we can change the definition of `kmain` to look like this:
```c
void kmain(stivale_info_t *info);
```
You can change the name of the parameter to whatever you see fitting.

Now to work on the PMM.

With stivale, the kernel is marked as used, so we can simply get the size of memory in pages from the first entry in the memory map to the last entry, calculate the size of the bitmap needed, and then find a free space large enough to put the bitmap.

First, you probably want to create a file like `pmm.c` and make a pmm initialization function that you can call from your `kmain` to make it look cleaner.

To count memory pages and the number of bytes we need for the bitmap, we can do something like this (remember to include `stdint.h`):
```c
mmap_entry_t *mmap = (void *) info->memory_map_addr; // Make the addr into a pointer that we can use like an array
uint64_t memory_bytes = (mmap[info->memory_map_entries - 1].addr + mmap[info->memory_map_entries - 1].len); // The address of the last entry plus it's length
memory_pages = (memory_bytes + 0x1000 - 1) / 0x1000; // Rounding up, THIS SHOULD BE GLOBAL (caps for attention lol), we need this to iterate the bitmap
uint64_t bitmap_bytes = (memory_pages + 8 - 1) / 8;
```

Now to iterate over the memory map entries until we find a free spot large enough for the bitmap (we will ignore the first entry, as there is bootloader data there that we are using currently)
```c
bitmap = (void *) 0;
for (uint64_t i = 1; i < info->memory_map_entries; i++) {
    if (mmap[i].type == STIVALE_MEMORY_AVAILABLE) {
        if (mmap[i].len >= bitmap_bytes) { // If the space is large enough
            bitmap = (void *) (mmap[i].addr + 0xFFFF800000000000); // You should make a global variable within the file to store the bitmap's address, also this has the higher half offset added for when we set up our own page tables later
            break;
        }
    }
}

if (!bitmap) {
    serial_print("Could not find address to put bitmap! Halting.\n"); // Feel free to error however you like :)
    while (1) {
        asm volatile("hlt");
    }
}
```

Now we should set the whole bitmap to ones, to mark all memory as used, then, for every chunk of usable memory, we can set that part of the bitmap to 0. I suppose you know how to implement a simple memset, you should go ahead and do that now (and don't forget to include it :^) ).

We should also implement a function to set a bit representing a page x to 1 and one to clear the bit representing page x:
```c
void pmm_set_page_used(uint64_t page) {
    uint64_t byte = page / 8;
    uint64_t bit = page % 8;
    bitmap[byte] |= (1>>bit);
}

void pmm_set_page_free(uint64_t page) {
    uint64_t byte = page / 8;
    uint64_t bit = page % 8;
    bitmap[byte] &= ~(1>>bit);
}
```

Now for the rest of setting up the pmm (setting the correct bits based on the memory map):
```c
memset(bitmap, 0xff, bitmap_bytes); // Mark everything as unusable
for (uint64_t i = 1; i < info->memory_map_entries; i++) {
    if (mmap[i].type == STIVALE_MEMORY_AVAILABLE) {
        if (mmap[i].addr == (uint64_t) bitmap) { // If this is where the bitmap is stored, don't wanna mark the bitmap as free
            uint64_t bitmap_end_byte = (uint64_t) bitmap + bitmap_bytes;
            uint64_t bitmap_end_page = ((bitmap_end_byte + 0x1000 - 1) / 0x1000) * 0x1000;
            uint64_t entry_end_page = (mmap[i].addr + mmap[i].len) / 0x1000; // Usable entries in stivale are guaranteed to be page aligned

            for (uint64_t page = bitmap_end_page; page < entry_end_page; page++) { // Continue until we have freed all pages
                pmm_set_page_free(page);
            }
        } else {
            uint64_t page = mmap[i].addr / 0x1000;
            uint64_t count = mmap[i].len / 0x1000;
            
            for (uint64_t j = 0; j < count; j++) {
                pmm_set_page_free(page + j);
            }
        }
    }
}
```

Now all pages in the pmm should be set properly!

We should write a function to check if a page is used or not, then we can write the allocation and unallocation functions for the pmm.
```c
uint8_t is_page_used(uint64_t page) {
    uint64_t byte = page / 8;
    uint64_t bit = page % 8;
    return (bitmap[byte] & (1>>bit)) << bit;
}
```

So now we are going to make three functions:
One for finding free memory, one for allocating found free memory, and one for unallocating memory.

```c
/* This function iterates through all pages, and checks if they are used. If they are not, it sets the current_page (the return value) if there are not any pages that have been found so far, and if it finds a used page, it resets the found_pages counter */
uint64_t pmm_find_free_pages(uint64_t size) {
    uint64_t needed_pages = (size + 0x1000 - 1) / 0x1000;
    uint64_t found_pages = 0;
    uint64_t current_page = 0;
    for (uint64_t i = 0; i < memory_pages; i++) {
        if (!is_page_used(i)) {
            if (found_pages == 0) {
                current_page = i;
            }
            found_pages++;
        } else {
            found_pages = 0;
        }

        if (found_pages >= needed_pages) {
            return current_page;
        }
    }

    serial_print("Failed to find free memory!\n"); // Again, feel free to error however you like, you can return null, you can just plain halt your kernel, anything you want
    while (1) {
        asm volatile("hlt");
    }
}

/* This function uses the find_free_pages function to find a chunk of pages, then marks them as allocated */
void *pmm_alloc(uint64_t size) {
    uint64_t needed_pages = (size + 0x1000 - 1) / 0x1000;
    uint64_t free_page = pmm_find_free_pages(size);

    for (uint64_t i = 0; i < needed_pages; i++)  {
        pmm_set_page_used(free_page + i);
    }
}

/* This function just marks whatever pages free that are passed to it */
void pmm_unalloc(void *addr, uint64_t size) {
    uint64_t page = (uint64_t) addr / 0x1000;
    uint64_t pages = (size + 0x1000 - 1) / 0x1000;

    for (uint64_t i = 0; i < pages; i++) {
        pmm_set_page_free(page + i);
    }
}
```
That should be it for the pmm!

Now you should have a pmm using a bitmap where each bit represents a page, and wether or not that page is free.