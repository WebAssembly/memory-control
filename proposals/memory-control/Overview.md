The linear memory associated with a WebAssembly instance is a contiguous, byte-addressable range of memory. This memory is a simple read-write buffer, which has no memory protection, and can grow but never shrink.

The current linear memory model is very simple and portable, and has proven to be effective for a surprisingly wide variety of use cases. However, it also imposes some unfortunate restrictions on serious WebAssembly applications, such as the inability to free memory, and a heavy reliance on copies when interacting with the host.

The need for finer-grained control of memory has been discussed since the early days of WebAssembly, and some early ideas were described in the [future features document](https://github.com/WebAssembly/design/blob/main/FutureFeatures.md#finer-grained-control-over-memory). This proposal attempts to finally make those ideas concrete.


## Motivation

The primary goals for this proposal are:

- **Reducing the impact of copies into WebAssembly memory.** In comparison with native applications, ML and Web Codecs applications in the browser context incur a non-negligible performance overhead due to the number of copies that need to be made across the pipeline. Furthermore, getting data from Web APIs into WebAssembly memory often requires copying through an intermediate buffer, instead of writing data into memory directly. (Examples: [WebCodecs](https://github.com/WICG/reducing-memory-copies/issues/1), [WebGPU](https://github.com/gpuweb/gpuweb/issues/747), [Machine learning on the Web](https://github.com/w3c/machine-learning-workshop/issues/93), [WebTransport](https://github.com/w3c/webtransport/issues/131#issuecomment-685004031))

- **Allowing applications to control their memory footprint.** WebAssembly memory can only grow and never shrink, and there are no APIs for releasing memory to the host, so a WebAssembly application's memory footprint can only increase. This may impact system performance, and on some devices (especially mobile devices) can result in the application being terminated. In addition, large WebAssembly memories can count against system memory limits, such as the [commit limit](https://techcommunity.microsoft.com/t5/windows-blog-archive/pushing-the-limits-of-windows-virtual-memory/ba-p/723750#toc-hId--959000264) on Windows, even if most of the memory is never used—forcing applications to try and be extremely conservative with not just use of memory, but address space. (Example and related concerns: [Unity](https://github.com/WebAssembly/design/issues/1397))

- **Supporting memory protection.** There are many use cases for making memory inaccessible or read-only. For example, languages may wish to trap on null pointer access, toolchains may wish to make constant data read-only to prevent corruption, and media codecs may wish to prevent clients from manipulating buffers that the decoder is actively using. In addition, control over page protection can be used to provide extra guarantees for sensitive code, e.g. [libsodium's guarded heap allocations](https://doc.libsodium.org/memory_management#guarded-heap-allocations).


## Proposed changes

This proposal is in its early stages. As such, we have multiple ideas for features that address one or more of the primary motivators (copies, footprint, protection). For clarity, each of these sub-proposals have been split into a separate document, but summarized here. Sub-proposals are expected to change, merge, and/or disappear before moving to phase 2.

- [**BYOB for WebAssembly.Memory**](byob.md) (copies): Allow WebAssembly memory to be used with the [BYOB API](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamBYOBReader) of `ReadableStream` on the web. This would enable various Web APIs like `fetch`, `Blob`, and OPFS to write directly into WebAssembly memory without intermediate copies. Other similar APIs may be developed for WebGPU and other Web APIs that do not use streams.

- [**`memory.discard`**](discard.md) (footprint): Add an instruction to WebAssembly which "frees" a page by clearing it to zero. On hosts with virtual memory, this can be implemented in a way that actually frees physical memory and reduces memory pressure on the system.

- [**Static memory protection**](static-protection.md) (protection): Add optional no-access and read-only sections to the start of WebAssembly memory, allowing common use cases of memory protection (null pointers and read-only data) to be implemented with either hardware memory protection or bounds checks.

- [**`maplength`**](maplength.md) (copies, protection?): Give WebAssembly memories a special region at the start of the address space that is "mappable", enabling special features like mapped files or memory protection. Restricting the special behavior to the start of memory allows the memory to still grow as it did before, and for most of it to be exposed to JS as usual.

- [**`virtual` mode**](virtual.md) (copies, footprint, protection): Add a new `virtual` mode to WebAssembly memory, which would allow each page of memory to be individually mapped and unmapped. This feature would effectively expose a portable subset of virtual memory APIs on major operating systems (and by extension the actual virtual memory capabilities of the hardware).


## Alternative approaches

### Multiple memories

Earlier versions of this proposal exploration included statically declared memories, with bind/unbind APIs, and exploring support for first class memories. However, this version of the proposal is currently designed to work with a single linear memory, and file mapping operations will only be available as JS API functions. Some reasons for scoping the proposal to use a single linear memory are:

- **Ease of portability:** Non-trivial addition of work when porting existing applications. A big win for existing applications is that porting to Wasm should in most cases work pretty much out of the box. Having multiple memories would need additional annotations, and would only address a smaller set of application use cases.
- **Better toolchain integration:** While LLVM supports multiple address spaces, almost all modern languages assume a single address space. Most languages are therefore not easily able to distinguish which memory a poitner should refer to.
- **Graceful degradation for existing applications:** The ability to easily polyfill this proposal is a requirement for some larger applications, especially when needing to target older devices. While this isn't a gating requirement, using a secondary memory does make this harder.

Challenges of using a single linear memory:

 - Significant API changes to expose more exotic memory features to the JS API.
 - Potential performance degradation to default memory (however, the goal would be to minimize any effects to the default memory).

### Lazy commit

TODO: Move this to its own sub-proposal?

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
