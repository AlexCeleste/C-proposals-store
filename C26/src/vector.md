    Proposal for C2y
    WG14 3837

    Title:               Simple Vector Types
    Author, affiliation: Alex Celeste, Perforce
    Date:                2026-06-01
    Proposal category:   New feature, adoption of existing practice
    Target audience:     Library developers, application developers

## Abstract

Several compilers support vector types, which are array-like sequences that allow
element-wise arithmetic on their stored element values using regular infix expression
syntax.

Although originally intended to enable SIMD optimizations, the expressiveness
afforded by being able to make certain loops implicit is attractive in its own right.
We propose a close-to-direct adoption of existing practice, with small modifications
to account for the dialect features that cannot be directly imported into Standard C,
of the GNU C Vector Extensions language feature, turning it into a generalized
vector extensions language feature that should be able to target a wide variety
of backend instructions.

The motivation for this adoption is the expressiveness of the syntax rather than the
optimization opportunities or deep ties to intrinsics it might also allow.

-------------------------------------------------------------------------------

# Simple Vector Types

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3837
    Revises:      N/A
    Date:         2026-06-01

## Summary of Changes

### N3837
 - original proposal

## Introduction

The GNU C dialect, as supported by GCC and Clang, provides [an extension][2] allowing for _vectors_ of
objects, supporting element-wise operations across array-like blocks of arithmetic-typed values.

GNU C enables this by using an attribute to mark a declaration that would otherwise declare an
identifier of scalar type, to actually have some wider footprint. The object is used with syntax
that would mostly (apart from initialization and element access) be valid for the underlying/declared
scalar type, and the compiler extends the operations, ideally using SIMD instructions, across all
elements in the vector.

Although these vectors are strictly aggregates, they are not arrays and have value-copy semantics.
This primarily means they can be intuitively used in nested infix expressions, rather than needing to
"root" every operation against an address, which is a weakness of some function libraries that work
with arrays (as well as having less surprising behaviour when passed to or returned from functions).

Therefore,

    typedef int v4si __attribute__((vector_size (16))); // 16 bytes -> 4 x i32
    
    v4si v1 = { 1, 2, 3, 4 };
    v4si v2 = { 2, 2, 2, 2 };
    
    v4si v3 = v1 * v2; // v3 contains { 2, 4, 6, 8 }
    v3 += v1;          // v3 contains { 3, 6, 9, 12 }

and with implicit expansion of scalar operands

    v4si v4 = v1 * 3;  // contains { 3, 6, 9, 12 }
    v4si v5 = v1 <= 2; // contains { 1, 1, 0, 0 }

etc. The feature really is simple to use and think about!

Vector types are supported by many major compilers, and there are several subtly different dialects
that differ mostly in how they relate to the _generated SIMD intrinsics_ rather than in the
readable semantics they present to the user (for instance, the AltiVec dialect differs by not allowing
the user to specify a number of elements). Most of the dialects are very similar in the interesting
features on the language side, with some small divergence in which exact arithmetic operators are
supported. In terms of actually supporting vector types with element-wise arithmetic, there is
very widespread praxis from GCC, Clang, IBM, and others.

For this proposal, we aim to work towards a standardization and unification of existing practice,
but coming at the feature from the angle of linguistic expressiveness, rather than of fitting
operations exactly to target SIMD instructions. If an element-wise vector operation doesn't fit in
a SIMD register or perhaps the right instruction doesn't exist, it's still correct for the compiler
to emit a hidden loop, and it may still be able to do interesting things at optimization time with
that loop having proved what it does and doesn't access. But for this proposal the interesting
aspect of the feature is _usability_; we consider element-wise operations to be fundamentally _more
readable_ than loops, regardless of their optimization properties, and approach the language-level
integration from that angle.

## Rationale

The primary motivation for this proposal is economy of expression, rather than necessarily expecting
code written using vector operations to be more efficient. It does follow that a compiler may be
able to unroll-and-reorder better if it is given slightly stricter information about the intended
traversal of a structure, but in practice the concern is more that by taking away a need to _write_
loops in one position, it is easier for the user to express the "right thing", and clearer when they
do so. Eliminating _syntactic_ loops from vector operations, regardless of whether the loop is
really eliminated or not in the backend, means one less thing for the reader to reason about.

Adding a new tool of expression may also help to guide the user into expressing their problem such
that it can be expressed using the new form, which may lead it to become _more_ strictly parallel
or SIMD in structure than an original design that relied on explicit loops. While the underlying
optimization power of the compiler doesn't change, providing vector-style expressiveness potentially
encourages giving it programs which are _structured to be_ easier to optimize, untangling logic at
the design stage instead of in the tooling backend.

### For changes from existing practice

_Some_ invention is needed in order to bring this feature into the Standard language because the
original design is based on attribute syntax, which is currently not used in the Standard to
describe any feature that actually changes the semantics of correct code.

A Standard attribute cannot change a scalar type into a vector type as this completely changes
its footprint; even though (the point of the features is that) most of the operations _as-written_
would appear to have the same infix syntax, the size, layout, and promotion properties of the types
described would completely change if the attribute was removed.

A keyword is necessary to achieve the same thing in Standard C as the attribute achieves in GNU C.

The other change (constraints on the size) is a logical extension of the functionality that any
compiler which supports the core feature can definitely handle with only minimal, frontend,
changes. GCC itself documents that it has de-facto support for relaxed size constraints
("... causes GCC to synthesize the instructions using a narrower mode"); the supported
combinations are targeted towards hardware support, whereas the interest of this proposal is in
expressiveness. This can be addressed with a Recommended Practice to warn when the combination
specified by the user would risk being inefficient.

## Proposal

We propose to add a new derived type family, the _vector_ types, to the language, and associated
operations on objects of these types using infix syntax.

### Vector types

This mostly adopts existing practice as-is, based mainly on [the GCC feature][2] provided by the
`vector_size` attribute. Similar extensions are available in Clang, IBM, and other compilers,
which mostly differ in that they instruct the compiler to target specific target features or
intrinsics; in terms of the _language_ features provided, [Clang's builtin support for NEON][3]
and the [AltiVec syntax][4] are extremely similar and have enough of a common subset that all of
them form essentially the same prior art. This gives C an opportunity to standardize and unify
incompatible but morally similar vector notations!

Since Standard attribute syntax is not appropriate, we need to add a keyword and syntax for it.
The obvious keyword is `_Vector` (AltiVec actually does just use `vector`) and there are perhaps
two obvious arrangements to accept a constant size operand and derive an element type:

    int v4si __attribute__ ((vector_size (16)));  // GCC syntax (specifies bytes)
    
    _Vector (int, 4) v4si;  // vector of four ints
    _Vector (4) int  v4si;  // (which might be 16 bytes, or might be more or less)

The second is more aesthetically pleasing, but the first form has the advantage that we can
compose it as a macro based on the GCC attribute for testing purposes.

We ideally want to avoid making the type part of the declarator syntax because unlike arrays,
vectors are an object type with _value copy semantics_ (and whatever we choose here should be
consistent with any future proposals for a value-oriented `_Array` type). The type is therefore
useful without being rooted in an identifier declaration, unlike arrays and array decay.

In a deviation from the existing features, which are all aimed at targeting _platform_ vector
features (and therefore specify some number of _bytes_ to fit single vector units; AltiVec
doesn't even allow specifying the number it fixes it to the target size), we explicitly want
to allow any number of _scalar elements_ - and to express the vector in terms of elements,
rather than as a byte footprint. This is because of the slightly different usability goals.
GCC _et al_ are certainly capable of rewriting a vector to a combination of other vectors and
do not do this by default because of the assumptions about the _intent_ implied by the feature.
Clang appears to relax this constraint already, although _use_ of the permissiveness is limited
by source code compatibility with GCC.

All of the three main prior art features offer essentially the same core functionality: after
defining an object of vector type, the user can operate on it using the same infix syntax that
they would for a single scalar element, broadcasting the operation across all elements and
producing a new vector as a result value containing each of the individual element results as
its elements. Because this result has copy semantics, it can be the immediate operand to another
operation, simplifying the expression to the same readable form that it would have over scalars,
and making the loops over the elements implicit (which the compiler can easily combine).

All three syntaxes support the arithmetic and relational operators, with the intuitive results.
They support initialization via a braced initializer list of scalar elements or by assignment
from a vector value, and access to individual operators using the subscript operator. There is
some divergence around support for logical operators (including the ternary operator) - one
potentially surprising aspect of allowing the `&&`, `||` or `?:` operators to be used with
vectors is that their short-circuiting behaviour is no longer consistent with the behaviour
with scalar operands (i.e. if the first operand to `?:` is a vector containing both true and
false values, _both_ the second and third operands will have to be evaluated to populate the
result). For now, this is enough to not include these operators.

None of the syntaxes allow implicit coercion or promotion, as the result of any vector operation
needs to be a vector of the same element type (so small integers can't promote to `int`-like
result).
GCC and Clang do allow a _scalar_ of the same element type (or a constant within range) to be
expanded into a vector for use as the other operand of a binary operator, which seems
suitable to include.
Converting a vector to a different vector with the same number of elements and a different
element type _is_ supported, but unfortunately isn't achieved using a cast in the general case.
For the time being we propose a generic library function to fill this role and to not
standardize vector casting.

The result is the ability to write code like:

    _Vector (4) int v4i1 = { 1, 2, 3, 4 };
    auto v4i2 = v4i1 + 2; // contains { 3, 4, 5, 6 }, expand rhs 4x
    
    auto v4i3 = v4i1 * v4i2; // not a dot or cross, contains { 3, 8, 15, 24 }
    auto v4i4 = v4i3 > v4i1; // contains { 0, 1, 1, 1 }
    
    auto v4i5 = (v4i1 + 2) * (v4i3 > v4i1); // nesting works, result is { 0, 4, 5, 6 }
    
    _Vector (4) short v4s1 = { 1, 2, 3, 4 };
    v4si + v4i1;  // error, operand types are not compatible
    
    _Vector (5) v5i1 = { 1, 2, 3, 4, 5 }; // implementation-defined size,
                                          // probably bigger than int[5]

## Alternatives

### Array notation for vectorization

A much more complicated and in-depth proposal for _generalized_ vector operations on
existing array types was put forward by Múgica in [N3717][5] ("ANFV"). This works to standardize
a different set of existing practice coming from OpenMP, Cilk, earlier work by the CPLEX
study group, and older research.

That proposal (and its predecessors) enables writing most of the same _simple_ vector
expressions that the GCC vector extensions do, in terms of the far more powerful _selection_
operators. However, while selections can express many things that are not directly expressible
at all with GCC-style vectors, they do make the simple operations a lot more verbose.

We feel like the two features can easily coexist in the language: vector types provide an
additional derived type that would still be suitable to use as the operand of _range selection_
where necessary (in exactly the same way that ANFV currently uses array types), while hiding
the need to use it for the simple cases, by making copies and full-range selections implicit.
We would like to see both features adopted if possible to give users the best combination of
expressive power (through the more complicated operations enabled by range selection) with
readability (using simple infix expressions over vector type values when uniform operations
are desired), especially if the operations in ANFV were extended to work on vector types as well.

ANFV seeks adoption as a TS, whereas N3837 targets direct integration into the core language.

### Operator overloading

An alternative approach that could enable defining such element-wise operations entirely as
library features would rely on operator overloading. This is very commonly used in C++ and
is both provided in the Standard Library as [`valarray`][6], which is oriented more towards the
"expressiveness" axis that concerns this proposal; and often seen as the user-friendly syntax
wrapper around platform intrinsics (like ARM NEON).

WG14 strongly rejected pursuing operator overloading in the Spring 2024 meeting, so while
this approach provides interesting context, it is not a suitable direction for development.

## Impact

This feature _can_ be implemented entirely in a compiler frontend. Broadcast arithmetic
operations can always be correctly implemented as a loop over an array, and do not require any
special SIMD hardware or support in the backend.

Previous approaches to this kind of functionality have always focused on the optimization and
ability to emit vector instructions, which is secondary to the intent of _this_ proposal;
however, because it is strongly based on existing practice that _did_ intend to enable actual
vectorization at runtime, we know that tools are able to produce such instructions. It seems
like a sensible implementation strategy would be to identify the cases where an element and
size combination maps to an existing known hardware type and use it if so; or synthesize it
from a combination of operations if not (alongside a recommendation to warn).

By specifying the layout of vector types in terms of structures and arrays, we _hope_ that the
existing ABI rules should be able to handle the new type derivation without too much difficulty.

## Future directions

The current proposed wording treats vector types as a top-level derivation and first-class
kind of types within the type system. Should they be treated as part of a type family alongside
the array types, to simplify the wording? Is a given vector type a subtype of the corresponding
array type?

If vector types are adopted, other features can usefully be described in terms of them.
One direction for further investigation would be to allow structs to be defined with the
`_Vector` keyword acting as a specifier that all members, despite being named, have a vector-like
layout and that objects of the struct type can be safely reinterpreted as vectors or as arrays.

Vectors with elements of boolean or non-arithmetic type are potentially interesting but not
currently supported (Clang supports vectors of `bool`). Should we allow a vector of structures
and broadcast the member access operator to produce a value vector containing a copy of that
member within each element, for instance? We might also allow a vector of pointers to be
dereferenced, which would enable scatter/gather operations with syntax resembling assignment
through a single pointer. This is supported _in hardware_ on some targets, but does not appear
to have much precedent as language syntax.

The questions around allowing casting from or to vector type, and around allowing ternary or
short-circuiting operators on operands of vector type, should be revisited.

There may be potential to allow unsequenced functions accepting a scalar argument to be
"externally" vectorized by passing them an argument of vector type.

There seems to be no particular reason _not_ to allow designators in vector initializers,
but this is not supported by prior art.

## Proposed wording

The proposed changes are based on the latest public draft of C23, which is
[N3886][0]. Bolded text is new text when inlined into an existing sentence.

### Language changes

Modify 6.2.5 "Types", paragraph 30, adding a new bullet point after the definition of
_array type_:

> - A _vector type_ is similar to an array type, being derived from an element type and describing
>   a constant number of contiguously allocated objects.
>   The element type is a signed or unsigned integer type, or a real floating type.
>   A vector type is said to be derived from its element type, and if its element type is _T_, the
>   vector type is sometimes called "vector of _T_". The construction of a vector type from an
>   element type is called "vector type derivation".
>   A vector of _T_ with _C_ elements has a _corresponding array type_, _T_`[`_C_`]`.
>   A vector type is a complete object type.

(ideally aiming to avoid repetition of the first part of the description; although the other bullet
points so all repeat the words w.r.t "derivation")

Modify paragraph 32:

> Arithmetic types, pointer types, and the `nullptr_t` type are collectively called scalar types.
> Array **, vector,** and structure types are collectively called aggregate types.

No addition is made to 6.2.7, because two vector types are _only_ compatible if they are the same.

Modify 6.2.8 "Alignment of objects", last sentence of paragraph 2:

> Whether any **vector or** atomic types have fundamental alignment is implementation-defined.

Modify the end of footnote 37 in the same way:

> <small>... , or **a vector or** atomic type that does not have a fundamental alignment.</small>

Modify 6.3.3.3 "Pointers", adding two new paragraphs after paragraph 10:

> A pointer to a vector type can be converted to a pointer to an array type with the same
> element type and the same number of elements. Accessing the elements of this array type
> through the converted pointer accesses the corresponding elements of the original vector.
>
> NOTE 4  The reverse does not hold, as a vector may have a larger footprint or stricter alignment
> requirements than the corresponding array type.

Modify 6.5.1  "General" (within "Expressions"), paragraph 9, adding two bullets to the end of the
list (cleaning up preceding commas):

> - the corresponding array type to a vector type compatible with the effective type of the object, or
>
> - a qualified verison of the corresponding array type to a vector type compatible with the effective
>   type of the object.

Add a new section to 6.5.1:

> **6.5.1.1 Vector operations**
>
> **Description**
>
> The constraints restricting the types of operands in subsequent sections do not apply to the use
> of operands with vector types, instead applying to the use of the operator with the scalar element
> type after an implicit rewrite according to the semantics below.
>
> **Constraints**
>
> If both operands of a binary operator have vector type, the types shall be the same.
>
> If only one operand of a binary operator has vector type, the other operand shall have the
> same type as the element type of the vector operand.
>
> **Semantics**
>
> The unary operators `+`, `-`, `~` and `!`, and the prefix and postfix increment and decrement
> operators, accept an operand of vector type.
>
> The result of one of these operators `@UOP` being applied to an operand `_operand` of type
> `_Vector (C) E` is as if a function with the following body had been evaluated:
>
>     _Vector (C) E _result;
>     for (int _i = 0; _i < C; ++ i) {
>       _result[_i] = (E) @UOP _operand[_i];
>     }
>
> ...with the final value of the operator being equivalent to the value of `_result`.
> The expression providing the value of `_operand` is only evaluated once.<sup>footnote)</sup>
>
> <small>footnote) `_operand` and `_result` are purely illustrative; no such variables are actually
> defined.</small>
>
> The binary operators `*`, `/`, `%`, `+`, `-`, `<<`, `>>`, `&`, `|`, and `^`; and the relational
> and equality operators, accept one or both operands having a vector type.
>
> The result of one of these operators `$@OP` being applied to operands `_lhs` and `_rhs` of type
> `_Vector (C) E` is as if the following had been evaluated:
>
>     _Vector (C) E _result;
>     for (int _i = 0; _i < C; ++ i) {
>       _result[_i] = (E) (_lhs[_i] @BOP _rhs[_i]);
>     }
>
> ...with the final value of the operator being equivalent to the value of `_result`.
> The expressions providing the values of `_lhs` and `_rhs` are only evaluated once.
>
> If only one operand has vector type, the other operand `_scalar` is first converted, as if the
> following had been evaluated:
>
>     _Vector (C) E _conv;
>     for (int _i = 0; _i < C; ++ i) {
>       _conv[_i] = (E) _scalar;
>     }
>
> The expression providing the value of `_scalar` is only evaluated once.
>
> The assignment operators accept one or both operators having a vector type.
>
> NOTE  No rewriting rules apply to the assignment operators; the compound assignment operators
> are already defined in terms of the equivalent binary operator, which is rewritten as above.
>
> **Forward references:** compound assignment (6.5.17.3), vector specifiers (6.7.3.7).

Modify 6.5.3.2 "Array subscripting", first sentence of paragraph 1:

> A postfix expression followed by an expression in square brackets `[]` is a subscripted designation
> of an element of an array **or vector**.

Modify paragraph 2:

> One of the operands shall have type "pointer to complete object _type_" or "array of _type_" **or**
> **"vector of _type_"**, the other operand, called the subscript, shall have integer type, and the
> result has type "type". If one of the two operands has array **or vector** type and the subscript
> is an integer constant expression, the value of the subscript shall not be negative.

Modify paragraph 4:

> Otherwise, let `E` be the operand of array **or vector** type and let `m` be the value of the
> subscript; the ~array~ subscript expression designates the `m+1`<sup>st</sup> element of the array **or vector**
> designated by `E` (i.e., the count of the subscript is zero-based, the first element being designated
> by the subscript 0). If `E` is an lvalue, the expression is an lvalue; otherwise, the expression is not
> an lvalue and its type is the unqualified, non-atomic version of the ~array’s~ element type. `m` shall
> not be negative and shall be less than the length of the array **or vector,** or equal to it; it shall
> only equal the length of the array **or vector** if the `[]` operator is followed by ...

Modify 6.5.4.5 "The `sizeof`, `_Countof` and `alignof` operators", second sentence of paragraph 1:

> ... that designates a bit-field member.
> The `_Countof` operator shall only be applied to an expression that has a complete array type,
> **or a vector type,** or to the parenthesized name of such a type. The `alignof` operator ...

Modify paragraph 5:

> The `_Countof` operator yields the number of elements of its operand. The number of elements is
> determined from the type of the operand. The result is an integer. **If the type is a vector**
> **type, the expression is an integer constant expression. Otherwise, the type is an array type;**
> **if** the number of elements of the array type is variable, the operand is evaluated; otherwise,
> the operand is not evaluated and the expression is an integer constant expression.

Modify the start of paragraph 8:

> EXAMPLE 2  The use of **the** `_Countof` operator is to compute the number of elements in an array
> **or vector**. Similar results ...

Modify 6.7.3 "Type specifiers", paragraph 1, adding  
  _vector-type-specifier_

...at the end of the list for _type-specifier_.

Add a new section after 6.7.3.6 "Typeof specifiers":

> **6.7.3.7  Vector specifiers**
>
> **Syntax**
>
> _vector-type-specifier_:  
>   `_Vector (` _constant-expression_ `)` _type-specifier_
>
> **Constraints**
>
> The _constant-expression_ shall have a value greater than zero.
>
> The _type-specifier_ shall specify a signed integer type, unsigned integer type, or real floating type.
>
> **Semantics**
>
> As discussed in 6.2.5, a vector type is a type consisting of a constant number of contiguously
> allocated objects.
>
> A vector type of _C_ elements of type _T_ shall have the same representation as the following
> structure type:
>
>     struct {
>       alignas (implementation defined)
>       T __elems[C];
>     };
>
> NOTE An object of vector type therefore fundamentally has the same underlying representation as 
> an object of the corresponding array type, except that the vector type may have stricter alignment
> requirements, and may have trailing padding.
>
> A vector type is similar to an array type, but is always copied by value on assignment, similar
> to an object of structure type. Although the individual elements of a vector can be accessed by
> subscript expressions in the same way as elements of an array, the main use of a vector is to
> reduce references to individual elements by _vectorizing_ expressions to apply to each element
> of the vector at once, as described by 6.5.1.1.
>
> EXAMPLE 1  Using an object of vector types makes it possible to express the same arithmetic
> operation affecting each element of the vector, without needing to write an explicit loop:
>
>     _Vector (4) short v4a = { 1, 2, 3, 4 };
>     _Vector (4) short v4b = { 2, 2, 1, 3 };
>     _Vector (4) short v4c = v4a * v4b;  // v4c contains { 2, 4, 3, 12 }
>     
>     // infix operators can be nested intuitively
>     _Vector (4) short v4d = (v4a + v4b) - v4c; // v4d contains { 1, 0, 1, -5 }
>     _Vector (4) short v4e = !v4d; // v4e contains { 0, 1, 0, 0 }
>     
>     _Vector (4) short f4f = v4a + !v4e; // v4f contains { 1, 3, 3, 4 }
>
> The result of these operations is always a vector value of the same type as the operands, with
> the pairwise result of the named binary operator applied to each pair of elements, or the result
> of the named unary operator applied to each single element.
>
> EXAMPLE 2  Operations with a vector operand can accept a scalar expression as the other operand;
> if so, the scalar is "expanded" to be used as every element of what would be the "other" vector:
>
>     _Vector (3) float v3a = { 1.0f, 2.0f, 3.0f };
>     
>     _Vector (3) float v3b = (v3a * 2.0) + 1.0; // contains { 3.0f, 5.0f, 7.0f }
>
> Like the vector operand, the scalar expression is only evaluated once:
>
>     long get () { static long r = 0; return ++ r; }
>     
>     _Vector (3) long v3l = { 1, 2, 3 };
>     v3l *= get (); // contains { 1, 2, 3 }, not { 1, 4, 9 } or anything
>     v3l *= get (); // contains { 2, 4, 6 }
>     v3l *= get (); // contains { 3, 6, 9 }
>
> EXAMPLE 3  The size and alignment of an object of vector type might be different from the size
> and alignment of an object of the corresponding array type:
>
>     _Vector (5) int v5 = { 1, 2, 3, 4, 5 }; // some implementations will pad this to 8
>     int a5[5] = { 1, 2, 3, 4, 5 };
>     
>     static_assert (sizeof v5 >= sizeof a5); // cannot be less than the array
>     static_assert (alignof (typeof (v5)) >= alignof (typeof (a5)));
>     
>     static_assert (sizeof v5 == sizeof a5, "this might fail");      // implementation-defined
>     static_assert (alignof (typeof (v5)) == alignof (typeof (a5))); // implementation-defined

Modify 6.7.4 "Type qualifiers", first sentence of paragraph 10:

> If the specification of an array **or vector** type includes any type qualifiers, both the array
> **or vector** and the element type are so-qualified.

Add a new paragraph to 6.7.11 "Initialization", after paragraph 7:

> The initializer for a vector shall be either a single expression that has compatible type
> or a brace-enclosed list of initializers for the elements. The initializer for a vector
> shall not contain designators.

(NOTE not immediately obvious that there's a reason for this other than lack of prior art)

Modify paragraph 17:

> If the initializer for a struct **, union, or vector** is a single expression, the initial value
> of the object, including **any** unnamed bit-fields, is that of the expression.

Add a new paragraph after paragraph 45:

> EXAMPLE 17  A vector can be initialized with a single expression of vector type, or a braced list
> of initializers for its elements:
>
>     _Vector (5) int va = { 1, 2, 3, 4, 5 };
>     _Vector (5) int vb = va;
>     _Vector (5) int vc = { va[0], va[2], va[4] }; // contains 1, 3, 5, 0, 0
>
> Any elements not explicitly initialized are subject to default initialization.
> There are no designators to identify specific vector elements.

### Library changes

Considering that the name `<stdvector.h>` might be confusing to users, for now we put the one
supporting function into `<stdlib.h>`, which already has some conversion functions.

Add a new section to 7.25 "General utilities `<stdlib.h>`":

> **7.25.X  Vector conversion functions**
>
> **7.25.X.1  The `stdc_convertvector` macro**
>
> **Synopsis**
>
>     #include <stdlib.h>
>     ToVec stdc_convertvector(FromVec vec, ToVec);
>
> **Description**
>
> The `stdc_convertvector` macro accepts a value of vector type _FromVec_, and a type name describing
> a vector type _ToVec_ with the same number of elements as _FromVec_, and returns a new vector with
> the arithmetic values of `vec` converted to the element type of _ToVec_.
>
> If converting any value of `vec` to the element type of _ToVec_ would invoke undefined behavior,
> the behavior of the whole conversion is undefined.
>
> **Returns**
>
> A value of type _ToVec_ containing elements converted from the value of `vec`.
> If _FromVec_ and _ToVec_ are the same type, a distinct value from `vec` is returned.
> The result is not an lvalue.

(This function is essential because it fills in for the missing cast expressions.
Additional _feature_ library functionality such as shuffles can potentially be added later.)

## Questions for WG14

Would WG14 like to add a "vector type" feature along the lines of the N3837 primary feature
proposal to C2y?

Would WG14 like to add a "vector structs" feature along the lines of the N3837 secondary
feature proposal to C2y?

What is WG14's preferred spelling for the vector type specifier?

## References

[C2y latest public draft][1]  
[GCC Vector Extensions][2]  
[Clang vector extensions][3]  
[AltiVec vector syntax documentation (IBM)][4]  
[Múgica, Array Notation for Vectorization][5]  
[C++ `std::valarray`][6]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3886.pdf
[2]: https://gcc.gnu.org/onlinedocs/gcc-16.1.0/gcc/Vector-Extensions.html
[3]: https://clang.llvm.org/docs/LanguageExtensions.html#vectors-and-extended-vectors
[4]: https://www.ibm.com/docs/en/xl-c-aix/13.1.0?topic=specifiers-vector-types-extension
[5]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3717.pdf
[6]: https://en.cppreference.com/cpp/numeric/valarray
