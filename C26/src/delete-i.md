    Proposal for C2y
    WG14 3581

    Title:               Remove the imaginary I, v2
    Author, affiliation: Alex Celeste, Perforce
    Date:                2025-02-25
    Proposal category:   Feature removal
    Target audience:     Library implementers, users

## Abstract

The enforced definition of `I` frequently obstructs the intent of the user,
replacing identifiers intended to declare other entities with a syntax error.
Audience feedback suggests that this is more common than intentional use of
the feature, so the feature should be removed.

As of Fall 2024, a macro is no longer required, as C2Y provides a true literal.

-------------------------------------------------------------------------------

# Remove the imaginary I, v2

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3581
    Revises:      N3390
    Date:         2025-02-25

## Summary of Changes

### N3581
 - finalized with Committee feedback, recommending delete outright without replacement
 - rebase on latest working draft

### N3390
 - original proposal
 
## Introduction

During the discussion of [N3274][1] and its predecessors, a comment was raised
in WG14 session that the macro `I` was a frequent source of nuisance to both C
and C++ programmers.

For example, in C:

    // (from the root file)
    #include <complex.h>
    
    // (potentially in another header, not intending to use complex)
    enum {
      A, B, C, D,
      E, F, G, H,
      I, J, K, L,  // whoops syntax error
    };

or in C++:

    template <typename I>  // really weird syntax error
    void with_range (I begin, I end) {
      ...
    }

and so on. Because the name is a macro, scope is no help in preventing the error
(except the limited "scope" of a macro parameter, which is not symmetric with the
common use of single-letter names in C++ templates).

Anecdotally, it seems that in usage, this kind of error is at least as common as
intentional use of the feature (obviously it does not generally persist into saved
projects). Reserving a single-letter name is surprising and unreasonable,
especially through a macro.

C2Y introduced genuine literals for complex numbers in [N3298][2], while C++ has
provided the literals as a library feature (via the UDL core language functionality)
since [N3779][3] added them to C++14. Therefore, in both languages it is now possible
to write:

    auto cf = 1.0i;

declaring `cf` as an object of type `_Complex double` or `std::complex <double>`,
without needing to write

    auto cf = 1.0 * I;

(which could potentially even have a different type since the type of `I` is
implementation defined)

In the presence of the new features the standard macro is therefore redundant.
As a feature test, `_Complex_I` would remain available and makes more sense to use
in that way.

At the Spring 2025 meeting the Committee expressed a preference to delete the
macro without a `constexpr` replacement, as it is easy for the user to define
for themselves.

## Alternative

Because C23 added `constexpr`, an alternative that would allow most arithmetic
uses of `I` to continue working unchanged might be:

    // in complex.h:
    constexpr auto I = _Complex_I;

This would eliminate the surprising syntax errors caused by macro replacement,
although the name itself would still be reserved in the global scope.

Because the underlying definition of `I` is left to the implementation (and requires
builtin magic or extensions prior to C2Y's introduction of true literals), it
seems unlikely that much code relies on the textual replacement of the `I` macro
being stable in any particular form.

Optionally, the header could still define

    #define I I

to act as a feature test. Whether this is useful or not is unclear, but it does
not have the same problem as a macro that would expand to expression tokens.

## Impact

Existing user code presumably relies on `I`.

Library implementations should guard the definition of the macro behind a version
check to continue to provide compatibility with C23 and earlier. This is not
particularly burdensome.

Code that uses `I` and wants to continue to compile in C2Y can always simply add
the definition explicitly:

    #include <complex>
    #define I _Complex_I
    // or
    constexpr auto I = _Complex_I;

Although implementations are generally free to declare more entities in their headers
than just those specified by the Standard, for this proposal to have a useful impact,
the macro should actually be removed rather than left in place anyway. Implementations
should be encouraged to guard it with a version check or WANT flag rather than take
no action and leave the definition present as an extension (which would be conforming,
but not helpful).

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3550][0]. Bolded text is new text when inlined into an existing sentence.

Modify 7.3 "Complex arithmetic `<complex.h>`":
  
Modify paragraph 3, deleting the references to `I` as a macro:

> The macro `complex` expands to `_Complex`; the macro `_Complex_I` expands to an
> arithmetic constant expression of type `float _Complex` with the value of the 
> imaginary unit;<sup>227)</sup>. Notwithstanding the provisions of 7.1.3
>
> - a program may undefine and perhaps then redefine the macro `complex`;
>
> ...

Replace references to `I` in the rest of the text with `1.0fi`:

Modify 6.7.11 paragraph 28:

>  **EXAMPLE 1**  Provided that `<complex.h>` has been included, the declarations
>
>     int j = 3.5;
>     double complex c = 5 + 3.0i;
>
> define and initialize `j` with the value 3 and c with the value 5.0 + _i_ 3.0.

(Editorial note: this renames the variable `i` in the example to `j`, to avoid confusion
with the imaginary unit invoked by the text description of the subsequent line)

Modify 7.3.9.5 "The `cproj` functions", example at the end of paragraph 2:

>     INFINITY + 1.0fi * copysign(0.0, cimag(z))

Modify G.3.2 paragraph 5 as appropriate w.r.t the changes in [N3274][1]:
delete the second prose sentence of paragraph 5, and replace `I` in the
`return` statement of the example with `1.0fi`.

## Questions for WG14

Does WG14 want to remove the `I` macro from `<complex.h>` in C2y?

Would WG14 like to replace the `I` macro with a C declaration of a named constant
`I` that does not introduce a macro?

## References

[C2y public draft][0]  
[N3274 Remove imaginary types, v3][1]  
[N3298 Introduce complex literals, v2][2]  
[N3779 User-defined literals for std::complex][3]  

[0]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3550.pdf
[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3274.pdf
[2]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3298.htm
[3]: https://isocpp.org/files/papers/N3779.pdf
