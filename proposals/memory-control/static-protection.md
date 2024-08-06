# Static memory protection

_(A sub-proposal of the [WebAssembly memory control proposal](Overview.md).)_

**Static memory protection** is a new design that would satisfy the most common use cases for memory protection in WebAssembly, while being efficiently implementable even without virtual memory hardware.

With static memory protection, WebAssembly memory would be split into three sections:

1. `no-access` (useful for null pages)
2. `read-only` (useful for constant data)
3. `read-write` (useful for everything else)

The sections are ordered so that `no-access` is always first, followed by `read-only`, followed by `read-write`. This ordering allows memory protection to be achieved easily through bounds checks if virtual memory hardware is not available.


## Details

### Typing

These sections are expressed by modifying the memory type with a descriptor of the sections.

```
(memory
  no-access-size
  read-only-size
  read-write-min
  read-write-max
)
```

As before, all properties are specified in terms of WebAssembly pages. Existing memory types are re-interpreted so that `min` corresponds to `read-write-min`, `max` corresponds to `read-write-max`, and `no-access-size` and `read-only-size` are zero.

The total initial size of memory would be `no-access-size + read-only-size + read-write-min`. The total max size of memory would be `no-access-size + read-only-size + read-write-max`.


### Growth semantics

At runtime, only the `read-write` section of memory may grow. The `no-access` and `read-only` sections always remain fixed at the amount specified in the type.


### Access semantics

For convenience, we will define the following variables:

- `read-only-base` is `no-access-size`
- `read-write-base` is `no-access-size + read-only-size`

Any memory instruction that performs a read must have an effective address of `>= read-only-base`. Any memory instruction that performs a write must be `>= read-write-base`.

These checks could be performed as part of an existing bounds check by subtracting (with wrap-around) the static section’s base from the address and memory length.

For example, a write could check against the length and read-only limit by: `address - read-write-base < memoryLength - read-write-base`. Note that `read-write-base` is completely static, and can be hardcoded into machine code or even optimized away in some cases.

Of course, virtual memory hosts could also implement this protection with hardware memory protection, incurring no new bounds check costs.


### Initializing read-only memory

One tricky piece is how to initialize the read-only area.

One option is to initialize memory in stages, such that the `read-only` section is temporarily read-write during instantiation until after data segments are evaluated. After data segments are evaluated, the `read-only` section becomes truly read-only.

This would only work for active segments, not passive segments. This may be an issue for multi-threading or dynamic linking, but these more exotic use cases would still benefit from the no-access part of static memory protection.

#### Alternative: allowing read-only to be introduced at runtime

We could allow the `read-only` section to grow at runtime, or the static limit to be imposed at runtime through a `memory.readprotect` instruction. This would allow passive segments to be written into memory, and the memory then switched to read-only protection afterward.

However, this would increase the cost of bounds checks because the `read-write-base` can no longer be baked into the executable code, and must be dynamically loaded. This would likely also increase register pressure.

Growing the read-only area would also implicitly cause the read-write area to shrink, violating compiler assumptions in the same way that a `memory.shrink` might.


### JS API

WebAssembly memory is accessible to JS via `ArrayBuffer`/`TypedArray`. These APIs do not have a notion of read-only/no-access memory, so we would need to decide how they behave.

One option would be to have the `Memory.buffer()` property only return a view of the `read-write` memory. Addresses would need to be manually translated to account for this, but the offset would be statically known.

The `read-only` portion of memory could be accessed through exported WebAssembly functions, and these functions could potentially be optimized and inlined. Alternatively, the read-only portion of memory could be exposed separately through another property, e.g. `Memory.readOnlyBuffer()`. This could perhaps use a copy-on-write mapping to cheaply prevent JS code from mutating the memory, similar to WebGPU’s `GPUMapMode.READ`.

There is no reason to expose the `no-access` portion of memory to JS.

We are not aware of any other way to safely expose read-only WebAssembly memory to JS, much less no-access memory. The contents of typed arrays in JS may be mutated arbitrarily by either JS or browser internal code, which makes the idea of trapping on bad access nearly impossible. The only sensible approach, it seems, is to expose each region of memory separately, if at all.

#### Alternative: Work with TC39 to develop first-class read-only collections for JS.

This would be hard, but perhaps possible, and perhaps the best move.


### Tables

All of this could theoretically apply to tables as well. For symmetry we could support the same for them as well, although the use case for this is not clear.

Regardless, we suggest extending the `limits` type and encoding for both memories and tables in order to preserve spec symmetry. Validation can ensure that tables always have a zero `no-access-size` and `read-only-size`.


## Rationale

Adding memory protection features to WebAssembly would be very useful for developers and toolchain authors. However, memory protection is typically implemented through virtual memory hardware, and allowing individual pages to be protected requires signal handling to implement efficiently.

Accepting a feature in core WebAssembly that requires virtual memory could be debatable, but accepting one that requires signal handling support is much more difficult. While support for virtual memory seems pretty widespread and consistent (outside of embedded), support for signal handling is much less common. Furthermore, we have learned that some WebAssembly hosts do not wish to expose virtual memory features to their users, since modifications to memory mappings can cause inter-process interrupts and significant performance problems.

From conversations with developers and compiler authors, the most common use cases for memory protection seem to be:

1. Null pointers not trapping
2. Constant data being writable

Static memory protection handles these use cases easily, at very little cost. It can be efficiently implemented on all WebAssembly hosts, and is fully backwards-compatible with current memories. It should therefore be a useful addition to the core WebAssembly spec.