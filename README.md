# Tartan Bitfield

[![Crate](https://img.shields.io/crates/v/tartan-bitfield)](https://crates.io/crates/tartan-bitfield)
[![Docs](https://img.shields.io/docsrs/tartan-bitfield)](https://docs.rs/tartan-bitfield)
[![Build](https://github.com/cimbul/tartan-bitfield/actions/workflows/build.yml/badge.svg)](https://github.com/cimbul/tartan-bitfield/actions/workflows/build.yml)
![License](https://img.shields.io/crates/l/tartan-bitfield)

This crate can be used to define structures with accessors for particular bits or
bit ranges. Useful for dealing with registers and protocols.

## Features

  * **Performance**: Generated code is nearly identical to hand-rolled bit twiddling.
  * **Safety**: Absolutely no unsafe code in the implementation or usage.
  * **Portability**:
    * `#![no_std]`-compatible out of the box.
    * Unlike bit fields in C, the layout is predictable. See the section about
      endianness below.
    * Unlike Rust's `#[repr(packed, C)]`, each field has explicit bit ranges, rather
      than relying on the ordering and size of other fields within the struct. While
      specifying bit numbers may seem tedious, it can eliminate surprises, and it
      usually corresponds directly to the way registers and protocols are defined in
      datasheets.
  * **Convenience**:
    * Single-bit flags and multi-bit fields can be defined in the same structure.
    * Bit ranges can be accessed as non-primitive, non-integer types (including other
      bitfield structs) using appropriate [`Into`] and [`From`] implementations.
    * The structs implement all the traits you would expect. See the documentation
      for [`bitfield`]. A [`bitfield_without_debug`] macro is also available if you
      want to provide your own debugging output.
    * Accessors can be defined in a trait, which is useful for registers where where
      some fields are common, but others are only defined in certain states. See
      [`bitfield_accessors`].

## Example

```rust
bitfield! {
    // The structure will be a wrapper for a u32 value.
    pub struct Example(u32) {
        // Accessors for field `a` will refer to the first four least significant
        // bits of the wrapped value, bits 0, 1, 2, and 3. Note that like normal
        // Rust ranges, the end of the range is *exclusive*.
        //
        // The accessors will be public, and will take/return the four bits as a `u8`.
        [0..4] pub a: u8,

        // No accessors cover bits `4..6`. This is legal and can be used for reserved
        // bits. However, these bits will still affect equality for the struct as a
        // whole.

        // Accessors for field `b` will refer to the twelve bits starting at bit 6,
        // but they will not be public. They will take/return the 12 bits as a `u16`.
        [6..18] b: u16,

        // Note that this bit range overlaps with `b`. This is allowed.
        [16..20] pub c: u8,

        // Accessors for field `d` will take/return a boolean and refer to a single
        // bit. Note that the `bool` is implied and not specified after the name.
        [25] pub d,

        // This will cover the 6 most significant bits of the wrapped value, but
        // the getters will take/return a `SubFields` struct instead of `u8`. This is
        // useful for nested bitfields, but the `A as B` syntax works for any `B`
        // which implements `Into<A>` and `From<A>`.
        [26..32] pub e: u8 as SubFields,
    }
}

bitfield! {
    // All accessors on this field use booleans and refer to single bits
    pub struct SubFields(u8) {
        [0] pub zero,
        [1] pub one,
        [2] pub two,
        [3] pub three,
        [4] pub four,
        [5] pub five,
    }
}


// The struct can be initialized with a u32 value
let x = Example(0xfa84_9e1b);
assert_eq!(x.a(), 0xb_u8);
assert_eq!(x.b(), 0x278_u16);  // Private, but still can be used within the module
assert_eq!(x.c(), 0x4_u8);
assert_eq!(x.d(), true);
assert_eq!(x.e(), SubFields(0x3e_u8));
assert_eq!(x.e().zero(), false);
assert_eq!(x.e().five(), true);

// It can also be converted Into and From its underlying representation
let n: u32 = x.into();
let y: Example = n.into();
assert_eq!(n, 0xfa84_9e1b);
assert_eq!(x, y);

// Setters are all prefixed with `set_`. They have the same visibility as the getters.
let mut z = Example::default();
z.set_a(0xb);
z.set_b(0x278);
z.set_c(0x4);
z.set_d(true);
z.set_e(SubFields(0x3e));
assert_eq!(z, Example(0xfa04_9e0b));

// Reserved ranges influence equality, and they are all zero on `z`.
assert_ne!(z, x);
```

For lots more examples, see the [Tartan OS](https://github.com/cimbul/tartan-os)
project that this crate was spun off from.

## Endiannness and Bit Numbering

Each bitfield wraps an underlying integer type. In the example above, `Example(u32)`
wraps a `u32`. Bit numbers within the macro refer to the bits of the logical _value_,
starting from the least significant bit = 0. They are not dependent on the order of
the bytes of the `u32` representation in memory, a.k.a. endianness.

The endianness of the underlying value is platform dependent. This is no different
than any other integer value, and the context determines whether you need to worry
about it.
  * If the underlying representation is a `u8`, then byte order is irrelevant.
  * If you are reading from a register, it's likely you want native byte order and
    don't need to do anything special.
  * If you are working with a network or bus protocol, it's likely you are serializing
    or deserializing from a byte array. To convert using a specific endianness
    regardless of platform, use the normal methods: for example, the builtins
    [`u32::from_be_bytes`] and [`u64::to_le_bytes`], or a crate like
    [byteorder](https://docs.rs/byteorder/latest/byteorder/).

## Alternatives

I have been using this in my personal OS project for a while, and it meets my needs
better than other options. But you may be interested in a few other crates:
  * [bitfield](https://github.com/dzamlo/rust-bitfield): Similar approach for
    accessors and bit ranges, but a less obvious syntax.
  * [bitflags](https://docs.rs/bitflags/latest/bitflags/): Works well for single-bit
    flags that can be viewed as a collection.
  * [bitvec](https://docs.rs/bitvec/latest/bitvec/): Another collection type to view
    memory as a sequence of bits. Less focused on defining domain-specific structs.
  * [modular-bitfield](https://docs.rs/modular-bitfield/latest/modular_bitfield/):
    Field ordering and widths determine bit ranges. Structs can only be converted to
    byte arrays, and only in little-endian order, regardless of platform endianness.
    This can be undesirable when working with registers.
  * [packed_struct](https://docs.rs/packed_struct/0.10.0/packed_struct/): Lots of
    options, including bit numbering and endianness conversions. Structs are held
    in unpacked form in memory, and only converted to packed form for serialization.
    Depending on your access patterns, this may be better or worse (or it may not
    matter at all).
      * For an analogue to `pack_struct`'s `PrimitiveEnum`, see the
        [`tartan-c-enum`](https://github.com/cimbul/tartan-c-enum) crate.

## Installation

Add to your Cargo.toml:
```
[dependencies]
tartan-bitfield = 1.0.0
```

## Development

This is a pretty standard Rust library using Cargo.

### Tests

```
cargo test --all-targets
```

### Benchmarks

```
cargo bench
```

### Formatting/Linting

```
cargo fmt
cargo clippy --all-targets
```

## License

Licensed under either of

 * Apache License, Version 2.0
   ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license
   ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.

---

<small>This README was generated from doc comments. Use `cargo readme` to refresh it.</small>
