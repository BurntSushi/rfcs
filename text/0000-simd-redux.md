- Feature Name: simd-redux
- Start Date: 2017-03-13
- RFC PR:
- Rust Issue:

# Table of contents

* [Summary][summary]
* [Motivation][motivation]
* [How We Teach This][how-we-teach-this]
* [Detailed design][design]
* [Drawbacks][drawbacks]
* [Alternatives][alternatives]
* [Unresolved questions][unresolved]

# Summary
[summary]: #summary

This RFC proposes the stabilization of explicit support for vendor dependent
intrinsics and a very small API for vendor independent SIMD operations. Vendor
dependent intrinsics are normal Rust functions that roughly match the
interfaces defined by various vendors, such as
[Intel's intrinsics](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)
or
[ARM's NEON intrinsics](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0491h/CIHJBEFE.html).

The primary goal of this effort is to provide explicit access to vendor
dependent intrinsics to programmers on stable Rust. The secondary goal of this
effort is to start the experimentation of higher level SIMD APIs in the Rust
ecosystem. This is the *beginning* of SIMD on Rust, not the end.

# Motivation
[motivation]: #motivation

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
reaching that discovering a new algorithm using specific SIMD dependent
intrinsics is generally considered to be a publishable result. Therefore, it is
paramount that a low level language like Rust provide access to a *familiar*
API that provides vendor dependent intrinsics.

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
