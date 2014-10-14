- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

* Remove reflection from the compiler
* Remove `libdebug`
* Remove the `Poly` format trait as well as the `:?` format specifier

# Motivation

In ancient Rust, one of the primary methods of printing a value was via the `%?`
format specifier. This would use reflection at runtime to determine how to print
a type. Metadata generated by the compiler (a `TyDesc`) would be generated to
guide the runtime in how to print a type. One of the great parts about
reflection was that it was quite easy to print any type. No extra burden was
required from the programmer to print something.

There are, however, a number of cons to this approach:

* Generating extra metadata for many many types by the compiler can lead to
  noticeable increases in compile time and binary size.
* This form of formatting is inherently not speedy. Widespread usage of `%?` led
  to misleading benchmarks about formatting in Rust.
* Depending on how metadata is handled, this scheme makes it very difficult to
  allow recompiling a library without recompiling downstream dependants.

Over time, usage off the `?` formatting has fallen out of fashion for the
following reasons:

* The `deriving`-based infrastructure was improved greatly and has started
  seeing much more widespread use, especially for traits like `Clone`.
* The formatting language implementation and syntax has changed. The most common
  formatter is now `{}` (an implementation of `Show`), and it is quite common to
  see an implementation of `Show` on nearly all types (frequently via
  `deriving`). This form of customizable-per-typformatting largely provides the
  gap that the original formatting language did not provide, which was limited
  to only primitives and `%?`.
* Compiler built-ins, such as `~[T]` and `~str` have been removed from the
  language, and runtime reflection on `Vec<T>` and `String` are far less useful
  (they just print pointers, not contents).

As a result, the `:?` formatting specifier is quite rarely used today, and
when it *is* used it's largely for historical purposes and the output is not of
very high quality any more.

The drawbacks and today's current state of affairs motivate this RFC to
recommend removing this infrastructure entirely. It's possible to add it back in
the future with a more modern design reflecting today's design principles of
Rust and the many language changes since the infrastructure was created.

# Detailed design

* Remove all reflection infrastructure from the compiler. I am not personally
  super familiar with what exists, but at least these concrete actions will be
  taken.
  * Remove the `visit_glue` function from `TyDesc`.
  * Remove any form of `visit_glue` generation.
  * (maybe?) Remove the `name` field of `TyDesc`.
* Remove `core::intrinsics::TyVisitor`
* Remove `core::intrinsics::visit_tydesc`
* Remove `libdebug`
* Remove `std::fmt::Poly`
* Remove the `:?` format specifier in the formatting language syntax.

# Drawbacks

The current infrastructure for reflection, although outdated, represents a
significant investment of work in the past which could be a shame to lose. While
present in the git history, this infrastructure has been updated over time, and
it will no longer receive this attention.

Additionally, given an arbitrary type `T`, it would now be impossible to print
it in literally any situation. Type parameters will now require some bound, such
as `Show`, to allow printing a type.

These two drawbacks are currently not seen as large enough to outweigh the gains
from reducing the surface area of the `std::fmt` API and reduction in
maintenance load on the compiler.

# Alternatives

The primary alternative to outright removing this infrastructure is to preserve
it, but flag it all as `#[experimental]` or feature-gated. The compiler could
require the `fmt_poly` feature gate to be enabled to enable formatting via `:?`
in a crate. This would mean that any backwards-incompatible changes could
continue to be made, and any arbitrary type `T` could still be printed.

# Unresolved questions

* Can `core::intrinsics::TyDesc` be removed entirely?