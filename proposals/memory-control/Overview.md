The linear memory associated with a WebAssembly instance is a contiguous, byte addressable range of memory. In the MVP each module or instance can only have one memory associated with it, this memory at index zero is the default memory. 

The need for finer grained control of memory has been in the cards since the early days of WebAssembly, and some functionality is also described in the [future features document](https://github.com/WebAssembly/design/blob/main/FutureFeatures.md#finer-grained-control-over-memory).

### Motivation
 - The primary motivation for this proposal is to reduce the impact of copying into Wasm memory. In comparison with native applications, ML,and Web Codecs applications in the browser context incur a non-negligible performance overhead due to the number of copies that need to be made across the pipeline (source: [WebCodecs](https://github.com/WICG/reducing-memory-copies/issues/1), [WebGPU](https://github.com/gpuweb/gpuweb/issues/747), Machine learning on the Web [1](https://github.com/w3c/machine-learning-workshop/issues/93), [2](https://github.com/w3c/webtransport/issues/131#issuecomment-685004031)). 
 - Widely used WebAssembly engines currently reserve large chunks of memory upfront to ensure that memory.grow can be efficient, and we can obtain a large linear contiguous chunk of memory that can be grown in place. While this works in the more general cases, it can be a very difficult requirement for systems that are memory constrained. In some cases it is possible that applications don’t use the memory that an engine implicitly reserves, so having some way to explicitly tell the OS that some reserved memory can be released would mean that applications have better control over their memory requirements.This was also previously discussed as an [addition to the MVP](https://github.com/WebAssembly/design/issues/384), and more recently as an option for [better memory management](https://github.com/WebAssembly/design/issues/384).
 - Ensuring that some data can also be read-only is useful for applications that want to provide a read only API to inspect the memory, or for media use cases that want to disallow clients from manipulating buffers that the decoder might still be using. From a security perspective, read only memory will provide increased security guarantees for constant data, or better security guarantees for sensitive data which is currently absent due to all of the memory being write accessible.


### Proposed changes

At a high level, this proposal aims to introduce the functionality of the instructions below: 
 - `memory.map`: Provide the functionality of `mmap(addr, length, PROT_READ|PROT_WRITE, MAP_FIXED, fd)` on POSIX, and `MapViewOfFile` on Windows with access `FILE_MAP_READ/FILE_MAP_WRITE`.
 - `memory.unmap`: Provide the functionality of POSIX `munmap(addr, length)`, and `UnmapViewOfFile(lpBaseAddress)` on Windows.
 - `memory.protect`: Provide the functionality of `mprotect` with `PROT_READ/PROT_WRITE` permissions, and `VirtualProtect` on Windows with memory protection constants `PAGE_READONLY` and `PAGE_READWRITE`.
 - `memory.discard`: Provide the functionality of `madvise(MADV_DONTNEED)` and `VirtualFree(MEM_DECOMMIT);VirtualAlloc(MEM_COMMIT)` on windows. 

Some options for next steps are outlined below, the details of the instruction semantics will depend on the option that introduces the least overhead of mapping external memory into the Wasm memory space. Both the options below below assume that additional memories apart from the default memory will be available. The current proposal will only introduce `memory.discard` to work on the default memory, the other three instructions will only operate on memory not at index zero. 

#### Option 1: Statically declared memories, with bind/unbind APIs (preferred)
Extend the multi-memory proposal with a JS API that enables access to memories other than the default memory

Reasons for preferring this approach: 
 - Having a statically known number of memories ahead of time may be useful for optimizing engine implementations
 - From looking at applications, it looks like applications do not require a large number of additional memories, and having a single digit number of extra memories may be sufficient for most cases. (This is a limited survey of applications, if there are more that would benefit from fully first class memories please respond here, or send them my way.)
 - Incremental addition over existing proposals

#### Option 2: First class WebAssembly memories
 - Explore adding memoryref, to dynamically bind/unbind memories from instances
 - Introduce a memory table, that stores the indices, base address, and properties of the memory (limited to read/write protections in this proposal, and potentially a growable bit to disable growing where necessary).


### Other alternatives

#### Why not just map/unmap to the single linear memory, or memory(0)?
 
 - I'm not sure that this can be done in any way that can still be compatible with the performance guarantees for the current memory. At minimum, I expect that more memory accesses would need to be bounds checked, read/write protections would also add extra overhead. 
 - Generalizing what this would need to look like, we need to store granular page level details for the memory which is complicates the engine implementations, especially because engines currently assume that Wasm owns the default memory, and have tricks in place to make this work in a performant and secure way (the use of guard pages for example).
 - To maintain backwards compatibility.

#### Web API extensions

To support WebAssembly owning the memory and also achieving zero copy data transfer is to extend Web APIs to take typed array views as input parameters into which outputs are written. The advantage here is that the set of APIs that need this can be scaled incrementally with time, and it minimizes the changes to the WebAssembly spec.

The disadvantages are that this would require changes to multiple Web APIs across different standards organizations, it’s not clear that the churn here would result in providing a better data transfer story as some APIs will still need to copy out.

This is summarizing a discussion from the [previous issue](https://github.com/WebAssembly/design/issues/1162) in which this approach was discussed in more detail.

#### Using GC Arrays

Though the proposal is still in phase 1, it is very probable that ArrayBuffers will be passed back and forth between JS/Wasm. Currently this proposal is not making assumptions about functionality that is not already available, and when available will evaluate what overhead it introduces with benchmarks. If at that time the mapping functionality is provided by the GC proposal without much overhead, and it makes sense to introduce a dependency on the GC proposal will be scoped to the other remaining functionality outlined above. 

### JS API story
Interaction of this proposal with JS is somewhat tricky because 

 - WebAssembly memory can be exported as an ArrayBuffer or a SharedArrayBuffer if the memory is shared, but ArrayBuffers do not have the notion of read protections for the ArrayBuffer. There are proposals in flight that explore this, and when this is standardized in JS, WebAssembly memory that is read-only either by using a map-for-read mapping or, protected to read only can be exposed to JS. There are currently proposals in flight that explore these restricted ArrayBuffers. ([1](https://github.com/tc39/proposal-limited-arraybuffer), [2](https://github.com/tc39/proposal-readonly-collections))
 - Multiple ArrayBuffers cannot alias the same backing store unless a SharedArrayBuffer is being used. One option would be for the [BufferObject](https://webassembly.github.io/spec/js-api/index.html#memories) to return the reference to the existing JS ArrayBuffer. Alternatively a restriction that could be imposed to only use SharedArrayBuffers when mapping memory, but this also would has trickle down effects into Web APIs. 
 - The multiple memories proposal adds the facility to Wasm to have declare and use multiple memories, but does not expose this in the JS interface. This proposal would need to extend that to be exposed to JS. The most natural API would be to expose each WebAssembly memory as a separate ArrayBuffer. 

### Open questions

#### Consistent implementation across platforms

The functions provided above only include Windows 8+ details. Chrome still supports Windows 7 for critical security issues, but only until Jan 2023, this proposal for now will only focus on windows system calls available on Windows 8+ for now. Any considerations of older Windows users will depend on usage stats of the interested engines.

#### How would this work in the tools? 

While dynamically adding/removing memories is a key use case, for C/C++/Rust programs operate in a single address space, and library code assumes that it has full access to the single address space, and can access any memory. With multiple memories, we are introducing separate address spaces so it’s not clear what overhead we would be introducing.

Similarly, read-only memory is not easy to differentiate in the current model when all the data is in a single read-write memory. 
