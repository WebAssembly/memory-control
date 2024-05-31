The linear memory associated with a WebAssembly instance is a contiguous, byte addressable range of memory. In the MVP, this memory is a simple read-write buffer, has no memory protection, which can grow but never shrink.

The need for finer-grained control of memory has been discussed since the early days of WebAssembly, and some functionality is also described in the [future features document](https://github.com/WebAssembly/design/blob/main/FutureFeatures.md#finer-grained-control-over-memory).


## Motivation

The primary motivations for this proposal are:

- **Reducing the impact of copies into WebAssembly memory.** In comparison with native applications, ML and Web Codecs applications in the browser context incur a non-negligible performance overhead due to the number of copies that need to be made across the pipeline. (Examples: [WebCodecs](https://github.com/WICG/reducing-memory-copies/issues/1), [WebGPU](https://github.com/gpuweb/gpuweb/issues/747), [Machine learning on the Web](https://github.com/w3c/machine-learning-workshop/issues/93), [WebTransport](https://github.com/w3c/webtransport/issues/131#issuecomment-685004031))
- **Allowing applications to control their memory footprint.** WebAssembly memory can only grow and never shrink, and there are no APIs for releasing memory to the host, so a WebAssembly application's memory footprint can only increase. This may impact system performance and can on some devices (especially mobile devices) result in the application being terminated. In addition, large WebAssembly memories can count against system memory limits, such as the [commit limit](https://techcommunity.microsoft.com/t5/windows-blog-archive/pushing-the-limits-of-windows-virtual-memory/ba-p/723750#toc-hId--959000264) on Windows, even if most of the memory is never used—forcing applications to try and be extremely conservative with not just use of memory, but address space. (Example and related concerns: [Unity](https://github.com/WebAssembly/design/issues/1397).)
- **Supporting memory protection.** There are many use cases for making memory inaccessible or read-only. For example, languages may wish to trap on null pointer access, toolchains may wish to make constant data read-only to prevent corruption, and media codecs may wish to prevent clients from manipulating buffers that the decoder is actively using.

All of these concerns can be addressed with a set of features based on system virtual memory APIs.


## Proposed changes

### Mappable memory

WebAssembly memories would be given a special "mappable" region of memory in which extra features are enabled. We are currently considering two ways of creating a mappable region of memory:

1. **`maplength`**: From the JS API, specify an amount of memory at the beginning of the wasm address space which will be mappable. In this proposal we have added a new `maplength` parameter to memory descriptors in JS.
2. **`mappable`**: Add a `mappable` mode to WebAssembly memories, e.g. `(memory mappable <min> <max>)`. Pages in a `mappable` memory would have individual state, be unmapped (and therefore trap) by default, and be mappable by host APIs and a few core wasm instructions.

#### `maplength`

The `maplength` approach would extend the memory descriptor to include a configurable mappable range from the low region of memory.

```javascript
dictionary MemoryDescriptor {
  required [EnforceRange] unsigned long initial;
  [EnforceRange] unsigned long maximum;
  boolean shared = false;
[EnforceRange] unsigned long maplength;
};
```
```html
<div algorithm>
    The <dfn constructor for="Memory">Memory(|descriptor|)</dfn> constructor, when invoked, performs the following steps:
    1. Let |initial| be |descriptor|["initial"].
    1. If |descriptor|["maximum"] [=map/exists=], let |maximum| be |descriptor|["maximum"]; otherwise, let |maximum| be empty.
    1. If |maximum| is not empty and |maximum| &lt; |initial|, throw a {{RangeError}} exception.
    1. Let |shared| be |descriptor|["shared"].
    1. If |shared| is true and |maximum| is empty, throw a {{TypeError}} exception.
    1. Let |maplength| be |descriptor|["maplength"].
    1. If |maximum| is empty and |maplength| is not empty, throw a {{TypeError}} exception.
    1. If |maplength| is not empty and |maximum| &lt; |maplength|, throw a {{RangeError}} exception.
    1. Let |memtype| be { min |initial|, max |maximum|, shared |shared|, maplength |maplength| }.
    1. Let |store| be the [=surrounding agent=]'s [=associated store=].
    1. Let (|store|, |memaddr|) be [=mem_alloc=](|store|, |memtype|). If allocation fails, throw a {{RangeError}} exception.
    1. Set the [=surrounding agent=]'s [=associated store=] to |store|.
    1. [=initialize a memory object|Initialize=] **this** from |memaddr|.
</div>
```

This approach allows the JS `WebAssembly.Memory.buffer` API to continue working mostly as it did before. By reserving the mappable space at the beginning of the memory, the `buffer` property can simply start after the mappable region, and other memory operations like `grow` remain unaffected. However, this approach mostly does not address the memory footprint concern, and restricts memory protection only to the beginning of memory.

#### `mappable`

The `mappable` approach would add a `mappable` mode to WebAssembly memories:

```
(memory mappable 1048576 1048576) ;; 20GiB of reserved mappable memory
```

All accesses within a `mappable` memory would trap by default. Pages of memory could then be mapped with either anonymous or file-backed mappings. Because the notion of a "file" is host-dependent, file-mapping functionality is left to host APIs (see the JS API section below).

Like shared memories, `mappable` memories would have a specified maximum size. The `mappable` mode would compose with other parameters like index type (i32/i64) and `shared`.

This approach is the most general and easily encompasses all of the primary motivators, but it is not clear how the WebAssembly memory would be exposed to JS, and it would be a more substantial departure from current semantics, raising questions of backwards compatibility and toolchain implementation effort. (Some of the toolchain effort may be mitigated by having a memory model much closer to typical native platforms.)

### JS API

At a high level, this proposal aims to introduce the functionality of the instructions below:

 - `memory.map`: Provide the functionality of `mmap(addr, length, PROT_READ|PROT_WRITE, MAP_FIXED, fd)` on POSIX, and `MapViewOfFile` on Windows with access `FILE_MAP_READ/FILE_MAP_WRITE`.
 - `memory.unmap`: Provide the functionality of POSIX `munmap(addr, length)`, and `UnmapViewOfFile(lpBaseAddress)` on Windows.
 - `memory.protect`: Provide the functionality of `mprotect` with `PROT_READ/PROT_WRITE` permissions, and `VirtualProtect` on Windows with memory protection constants `PAGE_READONLY` and `PAGE_READWRITE`.
 - `memory.discard`: Provide the functionality of `madvise(MADV_DONTNEED)` and `VirtualFree(MEM_DECOMMIT);VirtualAlloc(MEM_COMMIT)` on windows.

The example below assumes that a buffer exists, and is represented by an ArrayBuffer. memory.map then returns the start address of the mapping of the ArrayBuffer in Wasm memory. The buffer start address would be the beginning of the addresss to be mapped, and then the length of the mapping are used. Throws a runtime error on a failure to map.

The length is currently defined as bytes, but can also be changed to page size.

```javascript
// Example

var memory = new WebAssembly.Memory({initial = 10, maximum = 150, shared = false, maplength = 15})

WebAssembly.instantiateStreaming(fetch('memory.wasm'), {js: { memory }})
  .then(({instance}) => {
     let mapped_addr = memory.map(buffer,65536);
     // Do important stuff
     memory.unmap(buffer);
 });
```

##### TODO: memory.map runtime traps, additional details on the different buffer types.

##### TODO: memory.discard & memory.protect APIs

### New instructions

The following core wasm instructions round out the core spec by providing support for anonymous mappings, which toolchains can use in order to control null pointer behavior, read-only constant data regions, etc. These instructions are only valid within mappable memory regions.

- `memory.map <memidx> <prot>` (type `[addr:idx, size:idx] -> [addr:idx]`)

  Maps anonymous memory with the protection specified by `prot`, which can be `none`, `read`, or `readwrite`. The address and size will be aligned down and up respectively to the page size.

  Returns the actual start address of the mapped memory (after alignment).

  Traps if:
  - `size` is less than or equal to zero.
  - The requested range is out of bounds.
  - Any part of the range is already mapped. (Overlapping mappings are not allowed.)
    - TODO: Experiment on Windows to see if this requirement is necessary. (macOS and Linux both allow you to map over existing mappings at any time.)
  - The host disallows it for any other reason.

- `memory.unmap <memidx>` (type `[addr:idx size:idx] -> []`)

  Unmaps the requested range of pages. The address and size will be aligned down and up respectively to the page size. It is legal to unmap already-unmapped memory, so `memory.unmap` will never trap based on the state of the pages being unmapped. (TODO: Determine if Windows allows you to `VirtualFree(MEM_DECOMMIT)` part of a file view.)

  Traps if:
  - `size` is less than or equal to zero.
  - The requested range is out of bounds.
  - The host disallows it for any other reason.

- `memory.protect <memidx> <prot>` (type `[addr:idx size:idx] -> []`)

  Changes the protection on pages within a mapped memory region. `prot` can be `none`, `read`, or `readwrite`. The address and size will be aligned down and up respectively to the page size. The pages being protected need not be anonymous, but the host may not allow all types of protection on all types of host mappings.

  Traps if:
  - `size` is less than or equal to zero.
  - Any pages in the range are unmapped.
  - The host determines that the memory region cannot be protected in that way (e.g. attempting to make read-only host memory writable).
  - The host disallows it for any other reason.

Data segments can continue to be used for initializing memory with only one modification: when applied to a `mappable` memory, an active data segment will first execute `(memory.map <memidx> read (<data offset*>) (idx.const <data length>))`, then the rest of the initialization sequence as usual. `memory.init` would still work as currently specified.

Additionally, the following instruction would be added for non-mappable memory regions:

- `memory.discard <memidx>` (type `[addr:idx size:idx]`)

  "Frees" a range of WebAssembly linear memory by replacing it with zeroes while hinting to the operating system that any physical resources associated with the pages can be released. This is analogous to `madvise(MADV_DONTNEED)` on Linux. This provides a means for controlling the memory footprint of linear memory without introducing any new memory semantics. The address and size will be aligned down and up respectively to the page size.

  Traps if:
  - The requested range is out of bounds.


## Suggested Implementation

The general expectation is that a mappable memory region will act as "reserved", allocating address space in the process but not assigning any physical resources. On Windows, this would be a `VirtualAlloc(MEM_RESERVE)`; on macOS and Linux this would be `mmap(MAP_FIXED, PROT_NONE)` (which works even on systems without overcommit). Implementations should reserve (but not commit) the entire max memory size when creating a WebAssembly memory.

### Core instructions

The core `memory.map` would act as a "commit", (lazily) assigning physical resources to the pages. On Windows this would be a `VirtualAlloc(MEM_COMMIT, <prot>)`; on macOS and Linux this would be `mmap(MAP_FIXED, <prot>)`.

The core `memory.unmap` would act as a "decommit", releasing physical resources associated with the pages but retaining the address space within the process. On Windows this would be a `VirtualFree(MEM_DECOMMIT)`; on macOS and Linux this would be `mmap(MAP_FIXED, PROT_NONE)`. (Using `munmap` would release the address space to the operating system, possibly allowing it to be allocated by other parts of the host process.) Both macOS and Linux handle memory metrics correctly in this situation and collect the old mappings behind the scenes. [TODO: Find a reference for this.]

The core `memory.protect` does what it says on the tin. On Windows this would be a `VirtualProtect`; on macOS and Linux this would be `mprotect`.

### JS API

TODO: Describe approach to file descriptors, interaction with ArrayBuffers, restrictions and traps around core instructions, etc.


## Rationale

### Design decisions

- **Aligning `addr` and `size` vs. requiring alignment and trapping:** Different operating systems have different rounding and alignment requirements for various memory APIs, so we cannot rely on the operating system to provide this behavior. This means that either the user or the WebAssembly runtime must align these parameters. We have opted to have the runtime align these parameters to avoid unnecessary trap conditions and codegen, and to hopefully make the APIs easier to use.

- **Avoiding overlapping mappings:** TODO

### Alternative approaches

#### Multiple memories

Earlier versions of this proposal exploration included statically declared memories, with bind/unbind APIs, and exploring support for first class memories. However, this version of the proposal is currently designed to work with a single linear memory, and file mapping operations will only be available as JS API functions. The core spec should have some notion of a “special” mappable memory region, but outside of that we will likely not expose any other operations unless there is a consistent way that we can expose them that will be useful to other embeddings. Some reasons for scoping the proposal to use a single linear memory are:

- Ease of portability: Non-trivial addition of work when porting existing applications. A big win for existing applications is that porting to Wasm should in most cases work pretty much out of the box, having separate memories will need additional annotations, and only address a smaller set of application use cases
- Better toolchain integration: While LLVM supports multiple address spaces, most C/C++, Rust programs assume a single linear memory space, and that all pointers are accessible.
- Graceful degradation for existing applications: This is a requirement for some larger applications, especially when running on older devices where updates aren't always possible - that there would be an easy way to polyfill this proposal. While this isn't a gating requirement, using a secondary memory does make this harder.

Challenges of using a single linear memory:

 - Significant API changes to decouple the current 1:1 mapping for a WebAssembly.Memory object to a JS Memory object.
 - Potential performance degradation to default memory (however, the goal would be to minimize any effects to the default memory)
 - Engine implementations are more involved.

#### "Out parameters" in Web APIs

To support WebAssembly owning the memory and also achieving zero copy data transfer is to extend Web APIs to take typed array views as input parameters into which outputs are written. The advantage here is that the set of APIs that need this can be scaled incrementally with time, and it minimizes the changes to the WebAssembly spec.

The disadvantages are that this would require changes to multiple Web APIs across different standards organizations, it’s not clear that the churn here would result in providing a better data transfer story as some APIs will still need to copy out.

This also only addresses the cost of copies, not the other motivators of this proposal.

This is summarizing a discussion from the [previous issue](https://github.com/WebAssembly/design/issues/1162) in which this approach was discussed in more detail.

#### Lazy commit

If the goal is primarily to allow applications to manage their memory footprint, it may be possible to change the memory allocation strategy for linear memory to commit pages lazily. Currently, the general expectation is that engines will reserve + commit any accessible WebAssembly memory; under this approach, however, the engine would reserve memory only when accessed. This is, essentially, implementing page faults in software.

This strategy could be implemented in a signal handler, with a (presumed) slowdown on page faults. To avoid this slowdown, a new instruction could be provided (e.g. `memory.commit`) to pre-commit pages of memory before they are accessed. This instruction would be a complement to `memory.discard`. In effect, `memory.commit` would be a "soft allocate" and `memory.discard` a "soft free"—providing memory footprint control on most platforms but having no effect on WebAssembly hosts without virtual memory.

This approach may also make the implementation of `memory.discard` more portable for shared memories, since it could be implemented simply as a decommit rather than a re-commit (which is difficult / impossible to implement without the possibility of traps on all platforms).

However, this approach has no impact on the copy-reduction or memory-protection motivators for this proposal.

### JS API Considerations

Interaction of this proposal with JS is somewhat tricky because

 - WebAssembly memory can be exported as an ArrayBuffer or a SharedArrayBuffer if the memory is shared, but ArrayBuffers do not have the notion of read protections for the ArrayBuffer. There are proposals in flight that explore this, and when this is standardized in JS, WebAssembly memory that is read-only either by using a map-for-read mapping or, protected to read only can be exposed to JS. There are currently proposals in flight that explore these restricted ArrayBuffers. ([1](https://github.com/tc39/proposal-limited-arraybuffer), [2](https://github.com/tc39/proposal-readonly-collections))
 - Multiple ArrayBuffers cannot alias the same backing store unless a SharedArrayBuffer is being used. One option would be for the [BufferObject](https://webassembly.github.io/spec/js-api/index.html#memories) to return the reference to the existing JS ArrayBuffer. Alternatively a restriction that could be imposed to only use SharedArrayBuffers when mapping memory, but this also would has trickle down effects into Web APIs.
 - Detailed investigation needed into whether growing memory is feasible for memory mapped buffers. What restrictions should be in place when interacting with resizeable/non-resizeable buffers?

## Open questions / TODOs

- Add many more examples
- Define memory ordering (e.g. sequential consistency)
- Integration with streams
