# Boxed Externrefs in Rust

We want to add externref support to Rust. Rust's operational semantics assume
that any Rust value can be serialized to a "bag of bytes with provenance".
However, a true `externref` value is an opaque reference to an object owned by
the host garbage collector. Thus, adding to Rust a type that is lowered to an
externref violates the operational semantics of Rust. Instead, Rust will only
handle pointers to `externref`.

We define two new user-facing types:
* `Externref` which represents an owned externref pointer, and
* `ExternrefRef<'a>` which represents a borrowed externref pointer that is
   guaranteed to be valid for lifetime `'a`.

These both have the ABI of `__externref_t*` when passed to/from `extern "C"`
functions. (Or any other non "rust" ABI functions, I'm going to call them
`extern "C"` functions throughout.)

It is a key requirement for us that it should be possible to call C functions
that take and return `__externref_t` directly. For this reason, we define a
`ByValue<_>` struct that can be used to wrap `Externref` and `ExternrefRef<'a>`.
It marks the parameters and return values of `extern "C"` functions as
representing `__externref_t` instead of `__externref_t*`. If any of the
parameters or the return values have the `ByValue<_>` marker, the compiler emits
appropriate wrappers to adapt between `__externref_t*` and `__externref_t`.

The only place that we materialize a true `externref` is at the boundary of an
`extern "C"` function when a parameter or return type is explicitly marked
`ByValue<_>`.

## Dependency: LLVM RFC to add support for pointer to externref

There is an
[LLVM RFC](https://discourse.llvm.org/t/rfc-webassembly-externref-t-pointers-in-clang-and-llvm/91914)
to add support for pointer to externref, denoted `__externref_t*` in C.
Here I summarize the relevant details of the LLVM proposal for this proposal.

A blessed table is laid out with a bss section, a stack section, and a heap
section. A pointer to externref is represented in LLVM IR as `ptr addrspace(2)`.
Dereferencing a pointer to externref is lowered as a `table.get`. Assigning to
one is lowered as a `table.set`.

A pointer to externref can be allocated on the heap with
`__externref_table_alloc()` and `__externref_table_free()`. They have the
following LLVM attributes:

`__externref_table_alloc`:
    `allockind("alloc,zeroed")` `allocsize(0)` `"alloc-family"="__externref_table"`
    `mustprogress` `nofree` `nounwind` `willreturn` `memory(inaccessiblemem: readwrite)`

`__externref_table_free`:
    `allockind("free")` `allocsize(0)` `"alloc-family"="__externref_table"`
    `mustprogress` `nounwind` `willreturn` `memory(argmem: readwrite, inaccessiblemem: readwrite)`

This are almost the same attributes as are present on `malloc` and `free`. These
attributes allow a paired `__externref_table_alloc` and `__externref_table_free`
to be optimized out.

Because C and Rust objects linked together share the same table, no glue code is
needed to pass externref pointers between them.

## Specification

### The `c_externref_ptr` type

This type is a lang item which lowers to `ptr addrspace(2)`. It is a thin
pointer whose pointee cannot be read or written directly from Rust since the
pointee is not made of bytes. It is `Copy` and pointer-sized. Like raw pointers,
it is `!Send + !Sync`.

It is a pointer with provenance, following the same rules as `*mut T`. Index `0`
is reserved by LLVM as a null niche. Each `__externref_table_alloc` call
produces a distinct allocation, and LLVM performs alias analysis on
`addrspace(2)` pointers just as it does on memory pointers. It is not allowed to
cast `c_externref_ptr` directly to or from an integer, but it is possible to use
`expose_provenance()` and `with_exposed_provenance()` to do so.

It is only available on `wasm32`/`wasm64` targets with the `reference-types`
target feature, and lives in `core::arch::wasm32` alongside the types below.

### Intrinsic Helpers

We need the following helper functions:

```rs
unsafe extern "C" {
    fn __externref_table_alloc(nslots: usize) -> c_externref_ptr;
    fn __externref_table_free(ptr: c_externref_ptr);

    fn __externref_assign(from: c_externref_ptr, to: c_externref_ptr);
    fn __externref_set_null(ptr: c_externref_ptr);
    fn __externref_is_null(ptr: c_externref_ptr) -> bool;
}
```

`__externref_table_alloc` and `__externref_table_free` are implemented in
compiler-rt as part of the LLVM RFC. Rust picks them up through
`compiler-builtins`' `c` feature. A freshly allocated slot contains `ref.null`
and `__externref_table_free` nulls the slot before returning it to the free
list. When emitting LLVM IR declarations for them, Rust should include all the
attributes that LLVM normally emits in order to allow the best possible code
generation.

The remaining helpers have the following C definitions:
```C
void __externref_assign(__externref_t* from, __externref_t* to) {
    *to = *from;
}

void __externref_set_null(__externref_t* ptr) {
    *ptr = __builtin_wasm_ref_null_extern();
}

int __externref_is_null(__externref_t* ptr) {
    return __builtin_wasm_ref_is_null_extern(*ptr);
}
```


### Owned and Borrowed Externref Pointers

We define two types `Externref` and `ExternrefRef<'a>`. The first represents an
owned externref pointer that will be freed on drop. The second represents a
borrowed externref pointer that is valid for lifetime `'a`.

The design is analogous to the `std::os::fd` safety types: `Externref`
corresponds to `OwnedFd` and `ExternrefRef<'a>` corresponds to `BorrowedFd<'a>`.

Both are `repr(transparent)` over `c_externref_ptr`. They are both assumed
non-null so `0` is available as a niche. Because `c_externref_ptr` is
`!Send + !Sync`, so are they.

`Externref` and `ExternrefRef<_>` have interior mutibility. Because Rust does
not represent `__externref_t`, it is not possible for the pointer owned by an
`ExternrefRef` to ever alias with a Rust reference. For this reason, the
interior mutability poses no safety problems.

#### Borrowed Externref Pointers

The type `ExternrefRef<'a>` is defined as follows:

```rs
/// A borrowed externref pointer, valid for `'a`.
#[repr(transparent)]
#[derive(Clone, Copy)]
pub struct ExternrefRef<'a> {
    ptr: c_externref_ptr,
    _marker: PhantomData<&'a ()>,
}

/// Anything that can lend out a slot.
pub trait AsExternref {
    fn as_externref(&self) -> ExternrefRef<'_>;
}
impl AsExternref for Externref { /* ... */ }
impl AsExternref for ExternrefRef<'_> { /* ... */ }
impl<T: AsExternref + ByValueRepr> AsExternref for ByValue<T> { /* ... */ }
impl<T: AsExternref + ?Sized> AsExternref for &T { /* ... */ }

impl ExternrefRef<'_> {
    pub fn is_null(self) -> bool {
        unsafe { __externref_is_null(self.ptr) }
    }

    pub fn set(self, src: impl AsExternref) {
        unsafe { __externref_assign(src.as_externref().ptr, self.ptr) }
    }

    pub fn set_null(self) {
        unsafe { __externref_set_null(self.ptr) }
    }

    pub fn to_owned(self) -> Externref {
        let mut result = Externref::null();
        result.set(self);
        result
    }

    /// `ptr` must be valid for `'a`.
    pub unsafe fn from_raw(ptr: c_externref_ptr) -> Self { /* ... */ }
    pub fn as_raw(self) -> c_externref_ptr { self.ptr }
}
```

#### `Externref`

The type `Externref` is defined as follows:

```rs
/// An owned externref pointer.
#[repr(transparent)]
pub struct Externref { ptr: c_externref_ptr }

impl Externref {
    /// Allocates an externref pointer with value initialized to `ref.null`.
    pub fn null() -> Externref {
        let ptr = unsafe { __externref_table_alloc(1) };
        Externref { ptr }
    }

    pub fn is_null(&self) -> bool { self.as_externref().is_null() }
    pub fn set(&self, src: impl AsExternref) { self.as_externref().set(src) }
    pub fn set_null(&self) { self.as_externref().set_null() }

    /// Takes ownership of an externref pointer obtained from `__externref_table_alloc`.
    pub unsafe fn from_raw(ptr: c_externref_ptr) -> Self { /* ... */ }
    pub fn into_raw(self) -> c_externref_ptr { /* ... */ }
}

impl Drop for Externref {
    fn drop(&mut self) {
        unsafe { __externref_table_free(self.ptr) };
    }
}

impl Clone for Externref {
    fn clone(&self) -> Self { self.as_externref().to_owned() }
}
```

### Passing Raw Externrefs to/from foreign functions

We need a value that has C ABI `__externref_t`. For this purpose, we define the
`ByValue<T>` langitem. When this appears as an argument or a return value to an
`extern "C"` function, the compiler emits special wrapper code to convert
to/from raw `__externref_t`.

In an `extern "C"` function signature, it is only legal for `ByValue<T>` to
appear as a top level type. If it appears as a reference or pointer target, or
as a tuple, struct, or array element, a compiler error is emitted. Furthermore
each position admits exactly one `T`: arguments must be
`ByValue<ExternrefRef<'_>>` and returns must be `ByValue<Externref>`. If a
parameter has type `ByValue<Externref>` or a return value has type
`ByValue<ExternrefRef<'_>>` a compile error is emitted.

```rs
// Trait bound to limit T to one of Externref or ExternrefRef
pub trait ByValueRepr: AsExternref + sealed::Sealed {}
impl ByValueRepr for Externref {}
impl ByValueRepr for ExternrefRef<'_> {}

/// Marker: at a non-`"Rust"` ABI boundary the C side sees a bare
/// `__externref_t`; the Rust side sees `T`.
#[lang = "externref_by_value"]
#[repr(transparent)]
pub struct ByValue<T: ByValueRepr>(T);

impl<T: ByValueRepr> ByValue<T> {
    pub fn new(t: T) -> Self { ByValue(t) }
    pub fn into_inner(self) -> T { self.0 }
}
impl<T: ByValueRepr> From<T> for ByValue<T> { /* free */ }
```

#### Import ABI Wrappers

For a foreign function declared in an `extern` block, calls from Rust resolve
to a compiler-generated wrapper that translates each `ByValue` position:

* `ByValue<Externref>` argument: Compile error. Pass
  `ByValue::new(x.as_externref())` instead.
* `ByValue<ExternrefRef<'_>>` argument: The wrapper loads the externref and
  passes it by value. In C the conversion code would look like:
  ```C
  __externref_t convert_borrowed_arg(__externref_t* arg) {
    return *arg;
  }
  ```

* `ByValue<Externref>` return: The wrapper allocates a fresh externref pointer,
  stores the returned externref into it, and returns the slot as an owned
  `Externref`. In C the conversion code would look like:
  ```C
  __externref_t* convert_return_value(__externref_t retval) {
    __externref_t* result = __externref_table_alloc(1);
    *result = retval;
    return result;
  }
  ```

* `ByValue<ExternrefRef<'_>>` return: Compile error.

Example:

```rs
unsafe extern "C" {
    fn c_import(x: ByValue<ExternrefRef<'_>>, y: ExternrefRef<'_>) -> ByValue<Externref>;
}
```

corresponds to the C declaration

```C
__externref_t c_import(__externref_t x, __externref_t* y);
```

Only the `ByValue` parameter and the return are translated; `y` is already a
pointer.

Calls to `c_import` from Rust resolve to an implicit wrapper that loads the
argument slot, allocates a slot for the return value, and stores into it. In
pseudo code:

```rs
fn c_import(x: ByValue<ExternrefRef<'_>>, y: ExternrefRef<'_>) -> ByValue<Externref> {
    let result = Externref::null();
    unsafe { c_import_do_call(x.0.ptr, y.ptr, result.ptr) };
    ByValue(result)
}
```

with the C helper

```C
void c_import_do_call(__externref_t* x, __externref_t* y, __externref_t* result) {
    *result = orig_c_import(*x, y);
}
```

#### Export ABI Wrappers

For a Rust-defined `extern` function, the exported symbol is a
compiler-generated wrapper with the C by-value signature that calls the real
Rust body with slot pointers:

* `ByValue<Externref>` argument: Compile error. Declare the parameter as
  `ByValue<ExternrefRef<'_>>` and call `.to_owned()` in the body if an owned
  slot is needed.
* `ByValue<ExternrefRef<'_>>` argument: Store the incoming `__externref_t` into
  an alloca and pass the alloca pointer. In C the conversion looks like:
  ```C
  __externref_t* wrapper(..., __externref_t by_value_arg, /* ... other args */) {
    return rust_func(..., &by_value_arg, ...);
  }
  ```
* `ByValue<Externref>` return: The Rust body returns an owned externref
  pointer. The wrapper loads the externref, frees the pointer, and returns the
  externref by value. In C the conversion code would look like:
  ```C
  __externref_t convert_return_value(__externref_t* retval) {
    __externref_t result = *retval;
    __externref_table_free(retval);
    return result;
  }
  ```
* `ByValue<ExternrefRef<'_>>` return: Compile error.

Example:

```rs
#[unsafe(no_mangle)]
extern "C" fn c_export(x: ByValue<ExternrefRef<'_>>) -> ByValue<Externref> {
    // ... do something
}
```

exports the C signature `__externref_t c_export(__externref_t x)` via the
wrapper:

```C
__externref_t c_export(__externref_t x) {
    __externref_t* result_ptr = orig_c_export(&x);
    __externref_t result = *result_ptr;
    __externref_table_free(result_ptr);
    return result;
}
```

Direct calls to `c_export` from Rust resolve to `orig_c_export`, i.e. they
pass table slots and never go through the wrapper. The symbol
`c_export` (or the name given by `#[export_name]`) always refers to the
wrapped export.

#### `ByValue<_>` and Function Pointers

An `extern "C"` function pointer mentioning `ByValue<_>` always has the C
by-value ABI.

For isntance, a function `f` declared like
```rs
extern "C" fn f(ByValue<ExternrefRef<'_>>) -> ByValue<Externref>
```
has C signature:
```C
__externref_t f(__externref_t)
```
Coercing `f` to a pointer type yields the address of the export wrapper. Calling
through such a pointer from Rust goes through an import wrapper. This ensures
that the pointer type has the same ABI whether the pointer came from Rust or
from C.


#### Structs

A `repr(C)` struct containing `Externref` fields corresponds to a C struct
containing `__externref_t*` fields. No translation is applied.

```rs
#[repr(C)]
struct Pair {
    x: Externref,
    y: Externref,
}
```
corresponds to:
```C
struct Pair {
    __externref_t* x;
    __externref_t* y;
};
```

#### Borrowed slots across the boundary

C code that passes the address of a table-stack local, e.g.

```C
void c_fn(__externref_t* p);
__externref_t local = ...;
c_fn(&local);
```

is declared in Rust as `fn c_fn(p: ExternrefRef<'_>)` or, for a Rust export,
`extern "C" fn c_fn(p: ExternrefRef<'_>)`.

#### `with_stack_slot`

```rs
pub fn with_stack_slot<R>(f: impl for<'a> FnOnce(ExternrefRef<'a>) -> R) -> R;
```

A scoped intrinsic lowered to an `alloca` of one `__externref_t` in
`addrspace(2)`: the slot is set to `ref.null` on entry, the closure receives
its address, and it is cleared again on exit so the table stack does not pin
host objects after the scope ends. The HRTB lifetime prevents the slot from
escaping; code that needs the value afterwards calls `to_owned()` and gets an
explicit heap `Externref`.

The use case is scratch slots in small, frequently-called functions, where an
`Externref::null()` cannot be hoisted out of a loop because the function
boundary is the loop body:

```rs
fn swap(a: ExternrefRef<'_>, b: ExternrefRef<'_>) {
    with_stack_slot(|tmp| {
        tmp.set(a);
        a.set(b);
        b.set(tmp);
    });
}
```

The other common case is a C function that returns through an out-pointer,
where the received value is only forwarded:

```C
void lookup(const char* key, __externref_t* out);
void process(__externref_t x);
```

```rs
unsafe extern "C" {
    fn lookup(key: *const c_char, out: ExternrefRef<'_>);
    fn process(x: ByValue<ExternrefRef<'_>>);
}

fn lookup_and_process(key: &CStr) {
    with_stack_slot(|tmp| unsafe {
        lookup(key.as_ptr(), tmp);
        process(ByValue::new(tmp));
    });
}
```

Note this only helps for out-pointer returns. A C function that returns
`__externref_t` by value is declared `-> ByValue<Externref>` and the import
wrapper allocates a heap slot for the result; Rust has no way to express
"return into this slot", so that allocation stays unless the optimizer removes
it.

### Optimizing out allocations

Most `__externref_table_alloc`/`__externref_table_free` pairs introduced by
the wrappers have the shape

```C
__externref_t* p = __externref_table_alloc(1);
*p = x;
use(*p);
__externref_table_free(p);
```
where the slot never escapes as a pointer. The default `-O2` compilation
pipeline will optimize this to the following:
```C
use(x);
```

Only pointers that escape into a callee survive. For those, eliminating the
allocation requires the `AAHeapToStack` pass, which runs only under the
Attributor and is not in the default compilation pipeline. Using
`with_stack_slot` manually ensures that no allocation occurs in these cases. For
example, here the pointer escapes:
```C
__externref_t* p = __externref_table_alloc(1);
*p = x;
use(p);
__externref_table_free(p);
```
Ideally after optimizations the output would be the same as for the following:
```C
use(&x);
```
In order for this to happen, we must compile with the attributor and it must
prove one of the following two conditions:
1. `use()` never unwinds and never runs forever, or
2. `use()` does not capture or free the pointer.

Unfortunately, in order for `use()` to do anything useful with `p` it probably
has to give it to a host function and the Attributor probably can't prove
anything about what happens from there.