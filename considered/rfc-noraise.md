# `No_Raise` aspect

- Feature Name: generalized-finalization
- Start Date: 2020-11-23

## Summary

A new aspect, `No_Raise`, is introduced on subprograms.

## Motivation

In order to simplify the implementation of [Generalized
finalization](rfc-generalized-finalization.md), the semantics imposed in the
reference manual in section 7.6.1 need to be simplified. However we don't want
the semantics of raising an exception in a `Finalize` call to leave the program
potentially running in an inconsistent state, hence the necessity for this
aspect.

We're also taking inspiration from other languages such as Rust and C++, where in general:

* Raising an exception from a destructor is seen as a bad practice/critical error
* In most cases, it is preferable to either write your code to never have an
  exception bubble out of the destructor/abort the program in case of
  exception.

> [!NOTE]
> See https://github.com/rust-lang/lang-team/issues/97,
> https://blog.ycshao.com/2012/03/22/effective-c-item-8-prevent-exceptions-from-leaving-destructors/,


With this aspect, we enforce the implementations of `Finalize` to be
`No_Raise`, which means that if any exception is raised in `Finalize`, we mark
the program as being in an inconsistent state via an `Assertion_Error`.

This feature can be used more generally:

* As a way to annotate that you don't expect a specific subprogram to raise
  exceptions. Coupled with programming best practices, such as not using
  catch-alls, but only catching specific exceptions class, and never catching
  certain class of errors such as assertion errors or program errors, this will
  enforce safety oriented programming, allowing you to catch some errors
  earlier in the development cycle.

> TODO for a SPARK person: Can this be useful from a specification/proof
> perspective ? Or is this completely redundant in SPARK since it is the
> default already ?

## Guide-level explanation

A new aspect, `No_Raise`, is introduced on subprograms.

Should a subprogram with such an aspect have an exception be raised and not be
caught in the subprogram, an `Assertion_Error` will be raised by the
subprogram.

A new check category is introduced (See ARM 11.5), `No_Raise_Checks`. The above
assertion error is subject to this check category, and can hence be deactivated
by deactivating the check category, or by disabling run-time checks.
