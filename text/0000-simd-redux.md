- Feature Name: simd-redux
- Start Date: 2017-03-13
- RFC PR:
- Rust Issue:

# Table of contents

* [Summary][summary]
* [Terminology][terminology]
* [Motivation][motivation]
    * [Vendor independent interface][motivation-independent]
    * [Vendor dependent intrinsics][motivation-dependent]
    * [A brief survey][motivation-survey]
* [Detailed design][design]
* [How We Teach This][how-we-teach-this]
* [Drawbacks][drawbacks]
* [Alternatives][alternatives]
* [Unresolved questions][unresolved]

# Summary
[summary]: #summary

This RFC proposes the stabilization of explicit support for vendor dependent
intrinsics and a small API for vendor independent SIMD operations. Vendor
dependent intrinsics are normal Rust functions that roughly match the
interfaces defined by various vendors, such as
[Intel's intrinsics](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)
or
[ARM's NEON intrinsics](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0491h/CIHJBEFE.html).

The primary goal of this RFC is to provide explicit access to vendor dependent
intrinsics to programmers on stable Rust. The secondary goal of this RFC is to
start the experimentation of higher level SIMD APIs in the Rust ecosystem. This
is the *beginning* of SIMD on Rust, not the end.

# Terminology
[terminology]: #terminology

Before getting into the meat of the RFC, it would be a good idea to cover some
basic terminology used throughout this RFC.

* **SIMD** - SIMD stands for Single Instruction, Multiple Data. It is an
  umbrella term to refer to a class of CPU instructions that exploit hardware
  level parallelism to operate on multiple piece of data simultaneously.
* **SIMD vector** - Also sometimes written as simply "vector," is the unit
  of data that SIMD CPU instructions act upon. An example of a SIMD vector is
  `f32x4` (or Intel's equivalent, `__m128`), which is a sequence of 4 32-bit
  single-precision floating pointer numbers, occupying exactly 128-bits.
  Vectors come in many shapes and sizes, for example a `u8x16` is a 128-bit
  vector of 16 unsigned 8-bit integers and a `i32x8` is a 256-bit vector of
  8 signed 32-bit integer.
* **lane configuration** - A lane configuration refers to the number of units
  in a SIMD vector. For example, a `f32x4` vector has 4 32-bit lanes, and
  a `i8x32` has 32 8-bit lanes.
* **vendor** - A company that produces CPUs, e.g., Intel and ARM.
* **vendor independent API** - An API defined in terms of SIMD vectors that
  works on all supported platforms and *may* use special SIMD instructions on
  some platforms. For example, adding two vectors of `f32x4` can be done on
  CPUs with Intel SSE2 support in a single CPU instruction. If no such
  analogous support is available, then the addition of two vectors is done
  using normal instructions. The point of a vendor independent API is that it
  *abstracts away* the portability concerns of SIMD.
* **vendor dependent API** - An API defined in terms of specific instructions
  made available by vendors. Using a vendor dependent API is not portable,
  which means all uses must somehow check that the APIs used are actually
  available on the platform where the program is executed. A vendor dependent
  API is typically defined by the vendor itself using C functions and types,
  however, this RFC proposes its own mechanical translation of the API to Rust.
* **instruction set extensions** - A set of CPU instructions that have been
  added to the set of instructions normally supported by a CPU vendor. For
  example, `x86_64` corresponds to a specific set of CPU instructions that
  are guaranteed to be available on all CPUs advertising `x86_64` support.
  Over the years, new instructions have been added on top of this set as new
  CPUs have been released. This means that the set of instructions supported
  by any given `x86_64` CPU varies depending on the specific model of the CPU.
  **Note that instruction set extensions are mostly operations that act upon
  SIMD vectors.** Even though not all extensions are SIMD, we still sometimes
  use SIMD as an umbrella term to refer to them.
* **SSE** - SSE stands for Streaming SIMD Extensions. SSE is an example of
  a set of instruction set extensions for `x86` and `x86_64`. There are several
  versions of SSE that have been released over the years. Confusingly, `x86_64`
  actually includes SSE2 as part of its base set of instructions, which means
  every single `x86_64` CPU is guaranteed to have all SSE and SSE2
  instructions. Instructions introduced in SSE3, SSSE3, SSE4.1, SSE4.2, etc.,
  are not guaranteed to be available on any given `x86_64` CPU.
* **NEON** - NEON is an instruction set extension, just like SSE, except it's
  for ARM CPUs, not Intel.
* **compiler intrinsic** - A special type of function made available by the
  compiler. (**Note**: This RFC does not propose the stabilization of any
  compiler intrinsics.)
* **vendor intrinsic** - A normal Rust function whose API mechanically matches
  the API specified by a vendor.

# Motivation
[motivation]: #motivation

## Vendor dependent intrinsics
[motivation-dependent]: #motivation-dependent

To a first approximation, vendor dependent intrinsics provide a *reliable*
way of executing specific CPU instructions. These CPU instructions are often
desirable to use because they typically offer some kind of performance benefit.
Indeed, most vendor dependent intrinsics offer something called SIMD, or
"single instruction, multiple data." Namely, SIMD intrinsics offer access to
CPU instructions that exploit hardware level parallelism to operate on multiple
parts of some data at the same time.

The key word to focus on here is "reliable." In today's world, optimizing
compilers will introduce SIMD instructions in your program where applicable.
This process is generally referred to as *autovectorization*. But, optimizers
are generally opaque, which can make it hard to be certain that you're making
the most effective use of your CPU. Moreover, there are *thousands* of vendor
dependent intrinsics, and trying to convince an optimizing compiler to use a
specific one in any given circumstance may be fruitless. Some intrinsics are so
specialized that it may be very hard for a compiler to use it automatically.

Therefore, the fundamental role of vendor dependent intrinsics is to provide
programmers with a *reliable* way of executing a particular CPU instruction.

At a higher level, the actual use cases for these specialty instructions are
boundless. SIMD intrinsics are used in graphics, multimedia, linear algebra,
text search and more. The ubiquity of vendor dependent intrinsics is so far
reaching that discovering a new algorithm using specific SIMD intrinsics is
generally considered to be a publishable result. Therefore, it is paramount
that a low level language like Rust provide access to a *familiar* API that
provides vendor dependent intrinsics.

## A Brief Survey
[motivation-survey]: #motivation-survey

To further motivate this RFC, we should take a brief look to see how the
existing unstable SIMD features in Rust are being used.

* The [`simd` crate](https://github.com/rust-lang-nursery/simd) itself
  obviously makes heavy use of unstable SIMD features to provide a convenient
  high-level cross platform interface to SIMD vectors. More will be said about
  this crate later, but a goal of this RFC is to permit this crate to compile
  on stable Rust while exposing the same API it does today.
* The [`encoding_rs` crate](https://github.com/hsivonen/encoding_rs) uses
  the `simd` crate to assist with speedy decoding. It also makes use of a few
  vendor dependent intrinsics using the existing `platform-intrinsics`
  infrastructure. Unfortunately, it uses a special LLVM intrinsic called
  `simd_shuffle16`, which isn't part of the proposed API in this RFC. Other
  than that, this crate should be compilable with SIMD support on Rust stable
  with the plan in this RFC.
* The [`bytecount` crate](https://github.com/llogiq/bytecount) uses the `simd`
  crate with AVX2 extensions to accelerate counting bytes.
* The [`regex` crate](https://github.com/rust-lang/regex) uses the `simd`
  crate with SSSE3 extensions to accelerate multiple substring search.
  (See also the [`teddy` crate](https://github.com/jneem/teddy) which makes
   use of LLVM intrinsics like `simd_shuffle16` and `simd_shuffle32` directly.)
* The [`bitintr` crate](https://github.com/gnzlbg/bitintr) uses the
  `platform-intrinsic` infrastructure to access `bmi2` intrinsics which should
  be made available through Rust's standard library support for vendor
  dependent intrinsics.

There are other crates in the Rust ecosystem that use SIMD and all of them
share a single crucial bit in common: the SIMD functionality only works on
nightly Rust. There is a clear and pressing desire among Rust programmers to
get the most out of their CPUs!


# How We Teach This
[how-we-teach-this]: #how-we-teach-this

What names and terminology work best for these concepts and why?
How is this idea best presented—as a continuation of existing Rust patterns, or as a wholly new one?

Would the acceptance of this proposal change how Rust is taught to new users at any level?
How should this feature be introduced and taught to existing Rust users?

What additions or changes to the Rust Reference, _The Rust Programming Language_, and/or _Rust by Example_ does it entail?

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
