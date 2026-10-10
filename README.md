# Boxed Externref for Rust

This is a draft proposal for supporting a [boxed `Externref`](externref-proposal.md) type in Rust.

## Background

`externref` is a fundamental WebAssembly type introduced in the Reference Types proposal, and available in all stable browsers and JS engines as part of WebAssembly 2.0. It can be used to represent arbitrary host-defined types such as JavaScript values, while integrating with the host garbage collector.

All WebAssembly interoperating with JS must deal with host-defined JS values, whose representation in WebAssembly is thus `externref`.

For example, consider importing a JS string `"hello world"` and passing it to a `console.log` function:

```wat
(module
  ;; "hello world" is a JS UTF-16 string, imported as an immutable externref global
  (import "strings" "hello world" (global $hello (ref extern)))

  ;; console.log takes the string by value as an externref
  (import "console" "log" (func $log (param externref)))

  (func (export "main")
    global.get $hello
    call $log))
```

The string never exists in linear memory. It flows from the global to the stack to the call as an opaque handle; the module cannot inspect or copy its bytes, only pass it around and hand it to imports.

Yet WebAssembly like the above is not possible to output from the Rust compiler. Instead, all Rust toolchains needing to marshall `externref` values must do so indirectly through toolchain-specific tables with `table.get` and `table.set` instructions on all `externref` parameter and return paths. This suffers from both a performance cost, and a composition cost in toolchains.

Providing first-class `externref` support for the Rust language can solve this problem and alleviate the burden on Wasm toolchains for Rust in having to provide custom solutions for WebAssembly JS FFI.

### Prior Art

Clang supports `__externref_t` for first-class support for `externref` value marshalling in FFI.

Various discussions on Rust `externref` have been brought up over the years by various parties. A good overview of approaches is provided by @TimNN in https://hackmd.io/@TimNN/SJh8MNiN6.

Earlier in 2026, the same team now recommending this proposal drafted [RFC 3987](https://github.com/rust-lang/rfcs/pull/3987).

### Opaque Type Representation

[RFC 3987](https://github.com/rust-lang/rfcs/pull/3987) specifies a direct representation of Clang's `__externref_t` for Rust, as a new opaque lang type. Its unsized property is designated unobservable by banning its usage in all generic positions and thereby restricting usage to FFI marshalling only to support the use case.

Feedback on the proposal was mixed. The central objection is that Rust's operational semantics assume every value can be represented as bytes (with provenance); an opaque reference owned by the host garbage collector cannot, so a by-value `externref` type violates the language's memory model regardless of how its use is restricted.

This proposal is thus a response to that: rather than representing `externref` values, Rust represents only pointers to them.

## Design

This proposal specifies a boxed `Externref` which, instead of referring to the unsized and unrepresentable `__externref_t`, represents a transparent wrapper around `__externref_t*`, which is fully sized and representable.

To support FFI, `extern "C"` signatures may use a `ByValue<_>` wrapper which acts as an ABI signal that the position is marshalled as `__externref_t` rather than `__externref_t*`. Arguments are `ByValue<ExternrefRef<'_>>` and returns are `ByValue<Externref>`. This is then fully compatible with C and JS linking conventions for proper `externref` passing on the FFI boundary.

The `ByValue<_>` wrapper works by having the compiler emit table operations at the boundary: for an argument, the `externref` is loaded from the borrowed slot with `table.get` and passed by value; for a return, a slot is allocated and the returned `externref` is stored into it with `table.set`.

The WAT example above corresponds to the following Rust:

```rs
use core::arch::wasm32::{ByValue, ExternrefRef};

unsafe extern "C" {
    fn log(s: ByValue<ExternrefRef<'_>>);
}

#[unsafe(no_mangle)]
extern "C" fn greet(s: ByValue<ExternrefRef<'_>>) {
    unsafe { log(s) }
}
```

`greet` exports with the Wasm signature `(param externref)` and `log` imports with the same signature; the `externref` is passed through directly.

While this approach does incur an overhead over a pure `externref` representation due to these extra table operations, it is exactly the model existing Wasm toolchains such as wasm-bindgen already implement today: a `JsValue` is an index into a JS-side heap table, with a `table.get`/`table.set` on every boundary crossing. Moving that model into the compiler costs nothing new relative to current Rust-to-JS practice, while gaining a C-compatible ABI and compiler visibility into the table operations.

That visibility is what enables optimization. Because slot allocation and free carry the same LLVM attributes as `malloc` and `free`, the existing optimization pipeline removes paired allocations whose slot does not escape, so inlined chains of table store and table load fold away to direct `externref` passing, achieving parity with `__externref_t` marshalling in C.

[See the complete proposal here](externref-proposal.md).

### Tradeoffs

Unlike `__externref_t` which is widely supported in Clang, this proposal is dependent on Hood Chatham's [LLVM RFC for `__externref_t` pointers](https://discourse.llvm.org/t/rfc-webassembly-externref-t-pointers-in-clang-and-llvm/91914), which has not yet been approved.

Without the LLVM RFC, Rust would need to define its own externref table, but then the important optimization passes are not guaranteed without LLVM being aware of the table and its operator semantics.

## Next Steps

We are currently seeking early feedback on this proposal approach, to determine whether it should be recommended for RFC.

While https://github.com/rust-lang/rfcs/pull/3987 is effectively a simpler proposal, we believe this is the best sound alternative framing to the shortcomings that were raised there.
