The linear memory associated with a WebAssembly instance is a contiguous, byte addressable range of memory. In the MVP each module or instance can only have one memory associated with it, this memory at index zero is the default memory. 

The need for finer grained control of memory has been in the cards since the early days of WebAssembly, and some functionality is also described in the [future features document](https://github.com/WebAssembly/design/blob/main/FutureFeatures.md#finer-grained-control-over-memory).

### Motivation
 - The primary motivation for this proposal is to reduce the impact of copying into Wasm memory. In comparison with native applications, ML,and Web Codecs applications in the browser context incur a non-negligible performance overhead due to the number of copies that need to be made across the pipeline (source: [WebCodecs](https://github.com/WICG/reducing-memory-copies/issues/1), [WebGPU](https://github.com/gpuweb/gpuweb/issues/747), Machine learning on the Web [1](https://github.com/w3c/machine-learning-workshop/issues/93), [2](https://github.com/w3c/webtransport/issues/131#issuecomment-685004031)). 
 - Widely used WebAssembly engines currently reserve large chunks of memory upfront to ensure that memory.grow can be efficient, and we can obtain a large linear contiguous chunk of memory that can be grown in place. While this works in the more general cases, it can be a very difficult requirement for systems that are memory constrained. In some cases it is possible that applications don’t use the memory that an engine implicitly reserves, so having some way to explicitly tell the OS that some reserved memory can be released would mean that applications have better control over their memory requirements.This was also previously discussed as an [addition to the MVP](https://github.com/WebAssembly/design/issues/384), and more recently as an option for [better memory management](https://github.com/WebAssembly/design/issues/384).
 - Ensuring that some data can also be read-only is useful for applications that want to provide a read only API to inspect the memory, or for media use cases that want to disallow clients from manipulating buffers that the decoder might still be using. From a security perspective, read only memory will provide increased security guarantees for constant data, or better security guarantees for sensitive data which is currently absent due to all of the memory being write accessible.


### Proposed changes

#### Proposal scope

The proposal is currently scoped to work with a single linear memory, i.e. all of the proposed operations will operate on a single linear memory, and the operations proposed here will only be available as JS API functions. The core spec should have some notion of a “special” mappable memory, but outside of that we will likely not expose any other operations unless there is a consistent way that we can expose them that will be useful to other embeddings. Earlier versions of this proposal exploration included statically declared memories, with bind/unbind APIs, and exploring support for first class memories. Some reasons for scoping the proposal to use a single linear memory are: 

- Ease of portability: Non-trivial addition of work when porting existing applications. A big win for exisiting applications is that porting to Wasm should in most cases work pretty much out of the box, having separate memories will need additional annotations, and only address a smaller set of application use cases
- Better toolchain integration: While LLVM supports multiple address spaces, most C/C++, Rust programs assume a single linear memory space, and that all pointers are accessible.
- Graceful degradation for existing applications: This is a requirement for some larger applications, especially when running on older devices where updates aren't always possible - that there would be an easy way to polyfill this proposal. While this isn't a gating requirement, using a secondary memory does make this harder.

Challenges of using a single linear memory:

 - Significant API changes to decouple the current 1:1 mapping for a WebAssembly.Memory object to a JS Memory object.
 - Potential performance degradation to default memory (however, the goal would be to minimize any effects to the default memory)
 - Engine implementations are more involved.

#### Proposed API Extensions

At a high level, this proposal aims to introduce the functionality of the instructions below: 
 - `memory.map`: Provide the functionality of `mmap(addr, length, PROT_READ|PROT_WRITE, MAP_FIXED, fd)` on POSIX, and `MapViewOfFile` on Windows with access `FILE_MAP_READ/FILE_MAP_WRITE`.
 - `memory.unmap`: Provide the functionality of POSIX `munmap(addr, length)`, and `UnmapViewOfFile(lpBaseAddress)` on Windows.
 - `memory.protect`: Provide the functionality of `mprotect` with `PROT_READ/PROT_WRITE` permissions, and `VirtualProtect` on Windows with memory protection constants `PAGE_READONLY` and `PAGE_READWRITE`.
 - `memory.discard`: Provide the functionality of `madvise(MADV_DONTNEED)` and `VirtualFree(MEM_DECOMMIT);VirtualAlloc(MEM_COMMIT)` on windows.

Extend the memory descriptor to include a configurable mappable range from the low region of memory. This is the simplest version of the descriptor, extend this to also consider map for read/map for write. 

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

### Alternative approaches

#### Web API extensions

To support WebAssembly owning the memory and also achieving zero copy data transfer is to extend Web APIs to take typed array views as input parameters into which outputs are written. The advantage here is that the set of APIs that need this can be scaled incrementally with time, and it minimizes the changes to the WebAssembly spec.

The disadvantages are that this would require changes to multiple Web APIs across different standards organizations, it’s not clear that the churn here would result in providing a better data transfer story as some APIs will still need to copy out.

This is summarizing a discussion from the [previous issue](https://github.com/WebAssembly/design/issues/1162) in which this approach was discussed in more detail.

#### Using GC Arrays

Though the proposal is still in phase 1, it is very probable that ArrayBuffers will be passed back and forth between JS/Wasm. Currently this proposal is not making assumptions about functionality that is not already available, and when available will evaluate what overhead it introduces with benchmarks. If at that time the mapping functionality is provided by the GC proposal without much overhead, and it makes sense to introduce a dependency on the GC proposal will be scoped to the other remaining functionality outlined above. 

### JS API Considerations
Interaction of this proposal with JS is somewhat tricky because 

 - WebAssembly memory can be exported as an ArrayBuffer or a SharedArrayBuffer if the memory is shared, but ArrayBuffers do not have the notion of read protections for the ArrayBuffer. There are proposals in flight that explore this, and when this is standardized in JS, WebAssembly memory that is read-only either by using a map-for-read mapping or, protected to read only can be exposed to JS. There are currently proposals in flight that explore these restricted ArrayBuffers. ([1](https://github.com/tc39/proposal-limited-arraybuffer), [2](https://github.com/tc39/proposal-readonly-collections))
 - Multiple ArrayBuffers cannot alias the same backing store unless a SharedArrayBuffer is being used. One option would be for the [BufferObject](https://webassembly.github.io/spec/js-api/index.html#memories) to return the reference to the existing JS ArrayBuffer. Alternatively a restriction that could be imposed to only use SharedArrayBuffers when mapping memory, but this also would has trickle down effects into Web APIs. 
 - Detailed investigation needed into whether growing memory is feasible for memory mapped buffers. What restrictions should be in place when interacting with resizeable/non-resizeable buffers?
