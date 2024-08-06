# `maplength`

_(A sub-proposal of the [WebAssembly memory control proposal](Overview.md).)_

`maplength` would give WebAssembly memories a special "mappable" region of memory in which extra features are enabled, such as the ability to map file-like objects from other Web APIs. Specifically, `maplength` would extend the MemoryDescriptor in the JS API to include a configurable mappable range from the low region of memory.

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
