# `memory.discard`

_(A sub-proposal of the [WebAssembly memory control proposal](Overview.md).)_

The `memory.discard` instruction would allow WebAssembly applications to "free" pages of memory. Semantically, the instruction would simply fill a page with zeroes. However, on hosts with virtual memory, the instruction could be implemented by replacing the pages with fresh mappings, discarding the pages from the working set. This allows a WebAssembly application to control its memory footprint with a surprising amount of granularity.

## Details

```
memory.discard $memidx : [addr:idx size:idx] -> []
```

`memory.discard` "frees" a range of WebAssembly linear memory by replacing it with zeroes while hinting to the operating system that any physical resources associated with the pages can be released. The address and size are specified in bytes, and will be aligned down and up respectively to the page size.

Traps if:
- The requested range is out of bounds.

### Implementation

- **Windows:** On memory that has already been reserved and committed, `VirtualFree(MEM_DECOMMIT); VirtualAlloc(MEM_COMMIT)`.
- **Mac:** Use `mmap(MAP_FIXED)` to create a new mapping on top of the existing pages. The now-inaccessible physical pages will be collected by the system.
- **Linux:** Use `mmap(MAP_FIXED)` as on Mac, or `madvise(MADV_DONTNEED)`.

### Binary encoding

SpiderMonkey's prototype is currently using the opcode `0xfc 0x12`.

### JS API

```
interface Memory {
  // ...
  undefined discard(
    [EnforceRange] unsigned long addr,
    [EnforceRange] unsigned long size
  );
}
```


## Rationale

Generally speaking, a page of memory on modern operating systems can be in one of four states:

- **Initial:** The memory is inaccessible and the address space is not reserved.
- **Reserved:** The page's address space has been reserved, such that no other future memory allocation will use it.
- **Committed:** The page is ready to access, but not yet backed by physical memory.
- **Resident:** The page has actually been assigned to physical memory, and is tracked by the MMU and page tables.

Windows makes the distinctions between these states explicit. `VirtualAlloc(MEM_RESERVE)` will move pages from Initial to Reserved, allocating only address space, but not making the pages readable or writable. `VirtualAlloc(MEM_COMMIT)` will move pages from Reserved to Committed, making them accessible and setting their protection, but not yet actually assigning them to physical memory. Accessing the memory then triggers a page fault, causing Windows to move the pages from Committed to Resident to satisfy the fault.

On Linux, the states are less clear, as a call to `mmap` typically moves pages immediately to the Committed stage. However, there is still a clear boundary between Committed and Resident memory—memory will only be Resident once it is touched. (In fact, on Linux, the page must actually be written to, not merely read.)

On modern operating systems, Resident memory is by far the most important metric for memory pressure. It hardly matters how much memory is Reserved or Committed; Resident is what actually impacts system performance. Critically, this means that returning Resident pages to the Committed state is **almost as good as freeing them entirely.**

Furthermore, every WebAssembly memory starts in the Committed state, since engines commit the entire region of memory as read/write on instantiation. As pages of WebAssembly memory are accessed, those pages become Resident. An instruction to convert them back to Committed is as good as instantiating an entirely fresh memory.

### Other info and caveats

- Controlling Resident memory footprint is especially important on mobile, where operating systems are quick to kill expensive background processes. Android, for example, uses RSS, PSS, and USS to determine an application's memory footprint for killing, all of which are various ways of counting Resident memory. ([See docs.](https://developer.android.com/topic/performance/memory-management))

- On Windows, Committed memory does have an additional cost. In order to ensure that all allocated memory can actually be backed by physical memory, Windows has a hard commit limit, determined by the size of physical memory + the maximum size of the paging file (swap file). Maxing out the commit limit will cause memory allocations to fail, which is bad for system stability, but it has no impact on actual system performance, since physical memory is not yet actually used. (Maxing out Resident memory, on the other hand, causes the system to grind to a halt.)

- Linux systems may have a similar commit limit, although it may not be made explicit. Linux can be run with overcommit disabled, at which point Linux will similarly refuse to allocate memory it cannot back physically.


## Downsides

`memory.discard` seemingly has few downsides; it is very effective at reducing actual memory footprint on modern operating systems, it can be implemented trivially on all major operating systems, and it has a straightforward fallback on systems without virtual memory.

However, `memory.discard` does nothing to address commit limits. If you allocate 8GB of WebAssembly memory on a system with a 16GB commit limit, you have spent half of your entire system memory allocation even before touching the memory. This makes it impractical to ever use large WebAssembly memories in many contexts, especially browsers—instantiation will just fail on many systems.

Addressing large address spaces will likely require something like the proposed [virtual mode](virtual.md) for WebAssembly memory.