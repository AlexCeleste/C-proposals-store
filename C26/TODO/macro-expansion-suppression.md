
Proposal for C2x
WG14 ####

Title:               Suppressed macro expansion
Author, affiliation: Alex Gilding, Perforce
Date:                2020-08-09
Proposal category:   New features
Target audience:     Compiler/tooling developers

## Abstract

TODO


# Suppressed macro expansion

Reply-to:            Alex Gilding (agilding@perforce.com)
Document No:         N####
Revises Document No: N/A
Date:                2020-08-09

## Summary of Changes

### N####
- original proposal

## Introduction

This proposal suggests a mechanism by which expansion of certain identifiers defined as macros may
be suppressed on a contextual basis, when they would be considered valid candidates for expansion
according to the current rules followed by the preprocessor. This makes it easier for the same
identifier to co-exist in the preprocessing macro space, and a C language name space.

Defining a name in the macro space effectively adds it to all global namespaces and also makes it
impossible to override locally via any in-language mechanism. This is not desirable for short names.
Adding a way to prevent this goes some way to addressing the "commandeering" of names by the library
that requires them to be reserved for all uses, mitigating the impact of the
reserved-identifier-name-explosion.

## Rationale

Identifiers introduced by the C standard library may be provided as macros rather than as
in-language entities, or in addition to them. It is common for lightweight functions (such as the
`<ctype.h>` character queries) to be provided in this way, and is more or less necessary for type
generic functionality such as `<tgmath.h>`, and the operations provided by `<stdatomic.h>`.

Once a name has been defined as a macro, it is difficult to also use it in other contexts. This is
already sometimes encountered as an obstacle in C code, when accidentally trying to use a macro
name (especially an object-like macro name) in a context that is not supposed to refer to the
"global" macro definition. Because any name from the library can be provided as a macro, this is
also a hard obstacle to combatting the "reserved name explosion" by introducing a C++-like
namespace feature, to potentially gate new names inside `std::` or `std::fp::` - to be looked up
inside the namespace by language-level scope resolution, the name must not have already been erased
by expansion at preprocessing-time and replaced with some other expression syntax that would not
make sense to use as name lookup.

This problem is frequently encountered in C++ by Visual Studio users, for instance as in
[[1]](https://stackoverflow.com/q/2789481/1366431):

    inline void columns (int count = 1)
    {
      column = std::max (1u, column + count);
    }

The user had included `<windows.h>`, which defines `min` and `max`, causing the apparent function
call in the user source to be expanded to invalid syntax, producing confusing errors:

> ...\position.hh(83): error C2589: '(' : illegal token on right side of '::'
> ...\position.hh(83): error C2059: syntax error : '::'

Solutions include defining a guard macro to prevent `<windows.h>` from doing this, or to suppress
expansion by writing the function call so that it does not use a syntactic form also valid for
macro expansion:

    column = (std::max) (1u, column + count);

Since the entire rationale for the function-like macro syntax was to resemble function calls,
however, intentionally segregating every function call in this way defeats a core integration of
the features headers are supposed to provide. It is also unwieldy and impractical, and would not
work for object-like definitions in any case - it only works in the case where the macro expansion
requires a minimum of two separate tokens to be invoked, because the in-language call allows them
to be separated.

If namespaces were introduced to C, this problem would apply to every name in the standard library,
making the workaround extremely tedious. A separate guard macro for each name, functionality group,
or even header, would not be practical to use.

## Proposal

A new predefined macro name, `__NOEXPAND_AFTER__`, should be added to the list of mandatory names
provided by the preprocessor. The macro is defined as a whitespace-separated list of preprocessor
tokens. If one of these tokens occurs in the tokenized program source (phase 3) immediately before
an identifier, the identifier's token is permanently marked as not eligible for macro expansion.

For example:

    #include <ctype.h>

    namespace my {
      int (isalpha) (int); // parens still needed for declaration
    }

    int f1 (char c) {
      isalpha (c);     // maybe a macro, maybe not
      (isalpha) (c);   // traditional ugly way of ensuring non-macro
      my::isalpha (c); // would previously have been a syntax error, now suppressed, finds my::isalpha
      ::isalpha (c);   // not expanded, therefore standard library non-macro implementation
    }

    #define MY_ISALPHA_1(C) my::isalpha (C) // :: precedes isalpha at phase 3 - will not expand

    #define SEP ::
    #define MY_ISALPHA_2(C) my SEP isalpha (C) // doesn't precede at phase 3
                                               // if isalpha is a macro, will be a syntax error


By default, if the preprocessor is configured to preprocess a C source file (i.e. `__STDC__` is
defined, and `__cplusplus` is not defined), `__NOEXPAND_AFTER__` contains the namespace separator
punctuator `::`. If the preprocessor is instead configured to preprocess a C++ source file (i.e.
`__cplusplus` is defined), `__NOEXPAND_AFTER__` is either defined as empty, or not defined.

`__NOEXPAND_AFTER__` may be redefined and undefined by the user using the `#undef` and `#define`
directives. If it is redefined, the new set of tokens will be used for macro suppression from the
line following the directive. If `__NOEXPAND_AFTER__` is not defined, no token will be detected
as suppressing macro expansion; this is the same as if it is defined but empty.

For example:

    #undef  __NOEXPAND_AFTER__
    #define __NOEXPAND_AFTER__ :: Local // just add a suppression keyword

    namespace my {
      int Local isdigit (int); // one way to avoid the declaration problem
    }

    #undef  __NOEXPAND_AFTER__
    #define __NOEXPAND_AFTER__ int  // valid, "int" will now block macro expansion instead of ::
                                    // (this isn't a good idea, but)

    int f2 (int isalpha);       // will not expand to a syntax error
    int f3 (int isalpha (int)); // will also not expand to a syntax error

    #undef  __NOEXPAND_AFTER__
    #define  __NOEXPAND_AFTER__ ( [ { // first token inside any kind of brackets won't expand
                                      // (this REALLY isn't a good idea, but)
    ...
    {
      foo (isalpha (k));              // won't use a macro here, ever
      foo (isalpha (k), isalpha (j)); // won't use a macro the first time, will the second
      a[isalpha (k)];                 // similarly
    }

Allowing this improves the potential for interop between C and C++, where C code has to inline
data definitions or declarations from other languages, implementation-specific extensions (such as
`$` or `@` for implementations supporting address-literals), and so on. It may also make the
preprocessor more useful for other languages (for instance, in Objective-C it could make sense to
define to `:: @`, and to just `@` in Objective-C++). On an individual program level, adding `.` to
the list of suppressing identifiers would help separate macros (which resemble "ordinary" objects)
from the member namespace, which might alleviate confusion; however this should probably not be
enabled by default.

If two tokens appear adjacent in the replacement list of `__NOEXPAND_AFTER__` without whitespace
separation, the behaviour is implementation-defined.

TODO: probably need a way to store/push the "current" content of `__NOEXPAND_AFTER__`.

## Alternatives

[Shnell](http://shnell.gforge.inria.fr/), by Jens Gustedt, is used by Modular C to provide a
namespace functionality. This is a source stream transformer language which runs over sources
before they are handed to C pipeline. The advantage of this approach is that since shnell runs
before preprocessing, it is not affected by macro definitions (it is also a much more powerful
general-purpose language extension tool; however, this is limited to the namespace case). The
disadvantages are that it introduces yet another translation phase, complicating the pipeline;
that name lookup is separated from the in-language lookup (requires abbreviations rather than
scope-based); and that preprocessing cannot be used to compose names. It is also less consistent
with C++ usage, and instead branches out in a slightly different dialect direction.

## Impact

There would be no impact on any existing C programs, because in language versions prior to C2x, the
namespace separator token `::` is not a valid source token and does not form a part of any valid
C source.

A C++ program would be affected if `__NOEXPAND_AFTER__` is defined with content. However, since
idiomatic C++ prefers to use in-language facilities for generic programming, and the standard 
library does not introduce proportionally as many "transparent" macros to provide functionality,
C++ code is less heavily affected by the problem it aims to solve. Therefore, by default, there is
no impact on any existing C++ programs.

C++ is sometimes written with a metaprogramming style that uses macros to generate in-language
constructs, and since the namespace separator is already present in the language, often this can
mean expanding to namespace members, pointers to member functions, values from enumerations, and
other contexts where an identifier may be inserted after `::`. Therefore, the proposal makes the
value of `__NOEXPAND_AFTER__` language-dependent; if `__cplusplus` is defined, it should not be
defined with content by default.

While this is asymmetric by default, the problem faced by the idiomatic code written in the two
languages is asymmetric by default as well; defaulting to on for C and off for C++ targets each
more appropriately than a single value.

## Proposed Wording

The wording proposed is a diff from the ISO/IEC 9899-2018 archived public draft. Bolded text is
new text.

Modify 6.10.3 p9:

> A preprocessing directive of the form
>
>     #  define  identifier  replacement-list  new-line
>
> defines an _object-like macro_ that causes each subsequent instance of the macro name ^171) **that
> is not immediately preceded by a _no-expand token_** to be replaced by the replacement list of
> preprocessing tokens that constitute the remainder of the directive. The replacement list is then
> rescanned for more macro names as specified below.

Modify 6.10.3 p10:

> A preprocessing directive of the form
>
>     #  define  identifier  lparen  identifier-list opt  )  replacement-list  new-line
>     #  define  identifier  lparen  ...  )  replacement-list  new-line
>     #  define  identifier  lparen  identifier-list,  ...  )  replacement-list  new-line
>
> defines a _function-like macro_ with parameters, whose use is similar syntactically to a function
> call. The parameters are specified by the optional list of identifiers, whose scope extends from
> their declaration in the identifier list until the new-line character that terminates the
> `#define` preprocessing directive. Each subsequent instance of the function-like macro name **not
> immediately preceded by a _no-expand token_, and** followed by a `(` as the next preprocessing
> token**,** introduces the sequence of preprocessing tokens that is replaced by the replacement
> list in the definition (an invocation of the macro). The replaced sequence of preprocessing tokens
> is terminated by the matching `)` preprocessing token, skipping intervening matched pairs of left
> and right parenthesis preprocessing tokens. Within the sequence of preprocessing tokens making up
> an invocation of a function-like macro, new-line is considered a normal white-space character.

Add a new paragraph after 6.10.3.4 p2:

> If a _no_expand token_ appears in the replacement list, it no longer operates as a _no-expand
> token_ after expansion of the macro in which it appears, even if it is later (re)examined in a
> context which would place it before the name of a defined macro.

Add two new EXAMPLEs after 6.10.3.4 p4:

> EXAMPLE  macro replacement can be suppressed within a replacement list by a _no-expand token_.
> For example, given the following macro definitions:
>
>     #define A 1
>     #define B 2
>     #define C 3
>     #define D 4
>     #define F(J, K) A ::B J ::K
>
> the token sequence
>
>     F(C, D)
>
> expands to
>
>     1 B 3 K
>
> with the default _no-expand token_ set.

> EXAMPLE  macro replacement cannot introduce a _no-expand token_ to the expanding context. For
> example, given the following macro definitions:
>
>     #define A 1
>     #define SEP ::
>
> the token sequence
>
>     SEP A
>
> expands to
>
>     :: 1
>
> The `::` expanded from `SEP` does not act as a _no-expand token_ outside the definition of
> `SEP`.

Add a new section after 6.10.3.5:

### 6.10.3.6  Suppressed expansion

> A macro may be prevented from being replaced if its name appears syntactically immediately after
> a _no-expand token_, in the token sequence produced by translation phase 3. A macro name that has
> been prevented from expanding appears remains in the resulting token sequence as an identifier.
>
> A _no-expand token_ is identified as any token appearing in the replacement list of the
> predefined macro `__NOEXPAND_AFTER__` at the point where an identifier is considered for expansion.
> The macro `__NOEXPAND_AFTER__` may be undefined, or redefined with a new replacement list
> containing a new whitespace-separated list of _no-expand tokens_. The starting value of
> `__NOEXPAND_AFTER__` is the singular token `::`. If the replacement list of `__NOEXPAND_AFTER__`
> contains two adjacent non-whitespace-separated tokens, the behaviour is implementation-defined.
>
> The _no-expand token_ may be replaced by other tokens; the following macro name is prevented
> from expanding regardless. If the _no-expand token_ is not itself replaced, it also forms part of
> the resulting token sequence.
>
> An identifier preceded by a _no-expand token_ that does not name a macro is unaffected.

> EXAMPLE  by default, the `::` token prevents macro expansion. For example, given the following
> macro definitions:
>
>     #define A 1
>     #define B(N) N
>     #define C A
>     #define D ::A
>
> the token sequences
>
>     ::A
>     ::B(0)
>     C
>     D
>
> expand to
>
>     ::A
>     ::B(0)
>     1
>     ::A

> EXAMPLE  if the macro `__NOEXPAND_AFTER__` is redefined, different tokens may prevent macro
> expansion. For example, given the further definitions:
>
>     #undef  __NOEXPAND_AFTER__
>     #define __NOEXPAND_AFTER__ Ket Sup
>
> the token sequences
>
>     Ket A
>     Sup B(0)
>     C
>     D
>
> expand to
>
>     Ket A
>     Sup B(0)
>     1
>     1

> EXAMPLE  _no-expand tokens_ affect the syntactically-following identifier regardless of whether
> they are themselves subject to further expansion. For example, given the further definitions:
>
>     #define Ket Expanded
>     #define Sup
>
> the token sequences
>
>     Ket A
>     Sup B(0)
>
> expand to
>
>     Expanded A
>     B(0)

Add a new entry to the end of 6.10.8.1 p1:

> `__NOEXPAND_AFTER__`  The token `::`, intended to indicate that an immediately-following
>  identifier is not subject to macro expansion.

## References

[C18](https://web.archive.org/web/20181230041359if_/http://www.open-std.org/jtc1/sc22/wg14/www/abq/c17_updated_proposed_fdis.pdf)

[C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4713.pdf)

[Stack Overflow)(https://stackoverflow.com/q/2789481/1366431)

[schnell](http://shnell.gforge.inria.fr/)

[Modular C](http://cmod.gforge.inria.fr/)
