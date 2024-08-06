# `virtual` mode

_(A sub-proposal of the [WebAssembly memory control proposal](Overview.md).)_

This proposal would add a `virtual` mode to WebAssembly memories, enabling per-page control of page mapping and protection. `virtual` would be orthogonal to and composable with other memory options such as `i64` and `shared`.

```
(memory virtual 1048576 1048576) ;; 20GiB of "virtual" memory
```

In a `virtual` memory, all WebAssembly pages would trap on access by default, until mapped by either core WebAssembly instructions or operations on the host. This would naturally enable features like memory protection and file mapping, and would allow WebAssembly applications to use much larger address spaces without impacting system commit limits.

`virtual` mode could be implemented alongside other proposals like [`memory.discard`](discard.md) and [static memory protection](static-protection.md), since those other proposals are still potentially quite useful for non-`virtual` memory.

## Details

### Typing

Memory types would be extended to include a `virtual` mode. This option would compose with other memory properties, such as index type, `shared`, limits (min/max), and custom page sizes. It could also compose with [static memory protection](static-protection.md) if desired, although this would be unnecessary.

### New instructions

These instructions provide support for **anonymous mappings**; that is, normal plain-old-memory mappings not backed by a file. Toolchains can use in order to control null pointer behavior and read-only constant data regions, like in [static memory protection](static-protection.md). They can also be used at runtime to create guard pages or leverage virtual memory for data structures.

These instructions could also possibly be used with [`maplength`](maplength.md). They would be illegal inside a non-`virtual` memory.

#### `memory.map`

```
memory.map $memidx $prot : [addr:idx, size:idx] -> [addr:idx]
```

Maps anonymous memory with the protection specified by `$prot`, which can be `none`, `read`, or `readwrite`. The address and size will be aligned down and up respectively to the page size.

Returns the actual start address of the mapped memory (after alignment).

Traps if:
- `size` is less than or equal to zero.
- The requested range is out of bounds.
- Any part of the range is already mapped. (Overlapping mappings are not allowed.)
  - TODO: Experiment on Windows to see if this requirement is necessary. (macOS and Linux both allow you to map over existing mappings at any time.)
- The host disallows it for any other reason.

#### `memory.unmap`

```
memory.unmap $memidx : [addr:idx size:idx] -> []
```

Unmaps the requested range of pages. The address and size will be aligned down and up respectively to the page size. It is legal to unmap already-unmapped memory, so `memory.unmap` will never trap based on the state of the pages being unmapped. (TODO: Determine if Windows allows you to `VirtualFree(MEM_DECOMMIT)` part of a file view.)

Traps if:
- `size` is less than or equal to zero.
- The requested range is out of bounds.
- The host disallows it for any other reason.

#### `memory.protect`

```
memory.protect $memidx $prot : [addr:idx size:idx] -> []
```

Changes the protection on pages within a mapped memory region. `$prot` can be `none`, `read`, or `readwrite`. The address and size will be aligned down and up respectively to the page size. The pages being protected need not be anonymous, but the host may not allow all types of protection on all types of host mappings.

Traps if:
- `size` is less than or equal to zero.
- Any pages in the range are unmapped.
- The host determines that the memory region cannot be protected in that way (e.g. attempting to make read-only host memory writable).
- The host disallows it for any other reason.

### Instantiation

Data segments can continue to be used for initializing memory with only one modification: when applied to a `virtual` memory, an active data segment will first execute `(memory.map $memidx read (<data offset*>) (idx.const <data length>))`, then the rest of the initialization sequence as usual. `memory.init` would still work as currently specified.

### Implementation

The general expectation is that a virtual memory will reserve, but not commit, the entire memory's worth of address space on instantiation. Operations like `memory.map` would then commit pages as necessary.

- **Windows:** `VirtualAlloc(MEM_RESERVE)` on instantiation, `VirtualAlloc(MEM_COMMIT)` on `memory.map`, `VirtualFree(MEM_DECOMMIT)` on `memory.unmap`.
- **macOS and Linux:** `mmap(MAP_FIXED, PROT_NONE)` on instantiation, `mmap(MAP_FIXED, <prot>)` on `memory.map`, `munmap()` on `memory.unmap`.


## Rationale

Serious applications often need serious control of memory. Even with the potential ability to free memory using [`memory.discard`](discard.md), and to protect memory using [static memory protection](static-protection.md), more advanced use cases may still be limited by WebAssembly's memory model, which is not designed to take advantage of virtual memory on modern CPUs.

In native development, virtual memory hardware is used not just for protection and process isolation, but also to efficiently implement many interesting and useful techniques. For example, the encryption library `libsodium` can create [guarded heap allocations](https://doc.libsodium.org/memory_management#guarded-heap-allocations), where the allocated region is placed immediately before a no-access guard page, causing out-of-bounds accesses to immediately crash. Virtual memory can be used to [protect old pages](https://btmc.substack.com/i/142248177/safe-arenas-and-pools0) in memory allocators to catch a majority of use-after-free errors. Virtual memory can be used with [arena allocators](https://pkg.odin-lang.org/core/mem/virtual/#Arena) to create extremely large growable data structures with minimal memory footprint. It can even be used to implement [magic ring buffers](https://fgiesen.wordpress.com/2012/07/21/the-magic-ring-buffer/), wherein the same physical pages are mapped to _multiple_ adjacent ranges of physical memory, transparently enabling wrap-around reads and writes.

Perhaps most importantly, though, virtual memory allows applications to consume large amounts of _address space_ without consuming large amounts of _memory_. When developing natively for 64-bit systems, you can easily reserve gigabytes of address space and commit pages as necessary, yielding low fragmentation and enabling a wide variety of useful techniques as described above.

However, to confidently consume large amounts of address space, you must be confident that your memory reservations don't count against a system commit limit. All Windows machines and many Linux machines will refuse to commit more memory than the system guarantees it can back physically. For example, a system with 8GB of memory may refuse to commit more than 24GB.

Unless you can guarantee that the address space is only _reserved_, it is therefore unwise to use large amounts of address space. Unfortunately, all WebAssembly memory is generally committed from the start, since the entire memory region is immediately read-write. [`memory.discard`](discard.html) may help control memory footprint, but it does not increase the maximum amount of address space a WebAssembly module can safely use.

`virtual` mode solves this issue by making an explicit distinction between reserved and committed memory. By using `virtual` mode, it should be possible to confidently ship a cross-platform app with a memory size in the tens or potentially even hundreds of gigabytes, handling memory limits via `memory.map` the same way they would in native development.

## Design decisions

- **Aligning `addr` and `size` vs. requiring alignment and trapping:** Different operating systems have different rounding and alignment requirements for various memory APIs, so we cannot rely on the operating system to provide this behavior. This means that either the user or the WebAssembly runtime must align these parameters. We have opted to have the runtime align these parameters to avoid unnecessary trap conditions and codegen, and to hopefully make the APIs easier to use.

- **Avoiding overlapping mappings:** TODO
