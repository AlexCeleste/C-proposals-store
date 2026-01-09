    Proposal for C2y
    WG14 3579

    Title:               auto as a placeholder type specifier, v2
    Author, affiliation: Alex Celeste, Perforce
                         Aaron Ballman, Intel
    Date:                2025-01-31
    Proposal category:   Feature enhancement
    Target audience:     Compiler implementers

## Abstract

Implementation experience in leads maintainers to request two main changes to 
the way the `auto` specifier works: it should become a type specifier rather than 
remaining a storage class specifier, because it does not describe a storage class 
any more; and it should permit partial deduction of the type of the initializing 
value by allowing derived types to derive from the placeholder it describes.

Both of these changes should improve compatibility with C++, the first mostly for 
implementers and readers of the text; the second to better conform to common user 
expectations about how the feature should be available.

This proposal originated as a response to US National Body Comments 121, 122 and
123 against the CD ballot draft of the C23 Standard.

-------------------------------------------------------------------------------

# `auto` as a placeholder type specifier, v2

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3579
    Revises:      N3472
    Date:         2025-01-31

## Summary of Changes

### N3579
 - rebase against latest C2Y draft
 - remove unnecessary italicization
 - apply other reflector feedback
 - completely remove inference of array dimension from bracing structure
   (except where it already exists in the language) - braces now require an
   array declarator, removing all ambiguity

### N3472
 - rebase against latest C2Y draft (including fixing some completely wrong
   section numbers)
 - remove the restriction against multiple underspecified identifiers in a
   single declaration; clarify when they become complete
 - clarify that using the placeholder type specifier is not underspecified unless
   type inference occurs
 - don't inadvertently allow "the same" type to cover VMTs
 - provide rules for inferring an array type, and for partial deduction wrt.
   derived-declarator-types
 - improve examples

### N3339
 - rebase wording on C2y draft
 - address some feedback which arose during the comment resolution process
 - _not_ addressing the "same type" question at this time, left to other papers

### N3076
 - National Body Comments 121, 122 and 123 request changes to the specification
   and mechanism of the `auto` specifier from the version which was adopted

### N3007
 - version of `auto` which was adopted into C23
 
## Introduction

The adopted specification for `auto` aimed to meet two goals:

- not break any existing code that provided explicit type specifiers (code relying
  on implicit type specifiers is no longer allowed);
- inherit the exact semantics of the feature as already implemented by GCC and
  Clang via the `__auto_type` specifier found in the GNU C dialect.

`auto` can therefore be combined with other type specifiers, but as it is currently
provided it is not completely compatible with the C++ feature. This is because the
C++ feature specified in [C++23][2]/[dcl.type.auto.deduct] relies on wording from the
definition of templates (C++23/[temp.deduct.call]), which are completely missing
from C; an equivalent feature needs to be reconstructed instead of using a common
specification.

However, the missing specifications also leave the C feature far more limited in
scope, so reconstructing only the parts relevant to object definitions is not as
difficult as initially feared.

Additionally, changing the definition of `auto` from a storage class to a type
specifier is separately possible and comparatively easy. This is a prerequisite
task for partial deduction which can be integrated separately.

## Proposal

### `auto` as a type specifier

[US NB Comment 121][3] against the C23 CD ballot requested that `auto` be made into a type
specifier rather than being left as a storage class. Treating it as a storage class seems to
make unnecessary complexity for implementers and introduces incompatibility with C++. It leaves
in the ambiguity of declarations with no type at all still needing one to be implicitly added
by the compiler (implicit `int` becoming implicitly-deduced, but still syntactically implicit),
makes the specification more complicated, and serves no purpose because the specifier does
not describe a storage class (it can be used with static and thread storage durations).

Changing the specification to refer to a type specifier simplifies all of this, and is also
simpler for implementers, who can choose to interpret the keyword as one of two internally-distinct
specifiers depending on the language mode, rather than always interpreting it as a storage class
specifier with different effects (i.e. one grammar can contain two different syntactic `auto`
side-by-side, resolved at keyword-translation time). This is interesting to developers also
shipping combined C++ compilers who would like to be able to reuse as much of their internals
as possible across both language frontends.

We propose that the language introduce the term “placeholder type specifier” as found in C++.
A placeholder type specifier is simply a syntactic type specifier that has some additional
constraints on where it may appear (i.e. not in a _type-name_). This is an italicized term of art.

This would change all references to `auto` as a storage class specifier to refer to it as a
(placeholder) type specifier, and all references to "declarations that omit a type" (of which
there are now none), to "declarations that contain a placeholder type specifier". We allow the
placeholder type specifier to form part of any type specifier multiset, where it is simply
ignored if it is not the only component of the type specifier, which subsumes usages like
`auto int x = 10;` by making the type `auto int` (in which `auto` is ignored, and therefore
the specified type is just `int`).

### Support pointer and array declarators

[US NB comment 122][3] requested that C explicitly require support for pointer and array declarators.

Using explicit pointer declarators is common practice in C++, and users would be surprised not to
find it in C as well. C++ also supports deduction of function return types and array element types
in pointer-to-function and pointer-to-array declarators (but not of function parameter types).

Note that C++ deduces the type of a braced-initializer to be a std::initializer_list, not
an array. The syntax `auto a[] = { ... };` is therefore not valid in C++. However, lacking this
library feature, C users will expect arrays to substitute for it; and in any case there is no
additional complexity in allowing this form so long as we do not bother with complicated element
coercion rules, and instead require the elements to be explicitly compatible. (It would actually
introduce more wording to disallow this.)

This form was explicitly requested by the US NB so it is included here. The comments from Intel
were not concerned about a C++ compatibility issue here because definitions are mostly at block
scope, not at file scope where they might appear in shared headers.

Therefore, building on the previous resolution, we propose to allow _derived types_ to be
constructed from the _placeholder type_, so long as the sequence of type and declarator derivations
matches the same outermost sequence of derivations in the type of the initializer expression.

This trivially and elegantly allows the following forms:

    auto * p1 = ...
    auto p2[] = { ... }
    auto (*p3)[3] = ...;
    auto (*p4)(void) = ...;

and disallows

    atomic (auto) a = …;
    int (*f)(auto) = …;

Note that while the `atomic` derived type is not allowed because it uses `auto` as a type-name,
atomically-_qualifying_ the declared object is perfectly OK. This is a rare difference in the
behaviour of the `atomic` specifier and qualifier.

This serves to remove the implementation-defined behaviour in 6.7.10 "Type inference", and
the need for associated footnote 164.

At the Spring 2025 meeting, WG14 voted to only allow the _sizes_ of array derivations to be
deduced; the square brackets can be empty (or [unspecified, per N3427][5]), but they must be
present to indicate that a corresponding level of braces in the initializer is intended to
produce the array derivation. This substantially simplifies the deduction rules and is
consistent with existing deduction of array types from a braced initializer.

This paper assumes it will be integrated before [N3427][5], if both are accepted.
There are some areas of overlap that will need resolution by a second version of N3427.

## Impact

As integrated into C23, the `auto` specifier was intended to be an **exact** standardization of
the GNU C `__auto_type` specifier, renamed to look like the C++ keyword. This has the advantage
of being completely backed by long-established practice.

Some compatibility changes did sneak into the implementation of the tools, which were not intended
by the original proposal:

    void f (int x) {
      auto s1 = (struct { int y; }){ .y = x };  // GCC error, Clang OK
    
      __auto_type s2 = (struct { int z; }){ .z = x }; // both OK
    }

Whether to accept the first declaration is currently implementation-defined.

This proposal significantly extends the feature to more closely match it as it appears in C++.
This is not invention, but it is also not the exact GNU C feature we initially promised to
standardize. This proposal is however the direct result of implementation experience in Clang
of [the feature as it was standardized into C23][4].

Feedback from implementers seemed to indicate that being more generous with the syntax to allow
additional C++-ish forms (i.e. `auto * px = ...`) is preferable to sticking to exact C dialect
practice, because the C++ practice is more useful to them.

Implementation experience with the different requirements of the C and C++ features also led
Clang and QAC developers to conclude that a type-specifier based implementation would be simpler
than a storage-class based implementation. Although this seems like it should be a private
implementation detail, for a tool like Clang this distinction is important because of the way it
makes result data available, tied to the exact syntax productions of the user code.

The restriction against underspecified declarations declaring more than one ordinary identifier
is intentionally lifted, and we clarify in the new wording that an identifier is in scope at
the end of its _init-declarator_ rather than at the semicolon. This means that code like:

    constexpr int A = 1, B = A + 1;

becomes legal, with a clearly-defined meaning. It also means that the following code is legal:

    auto A = 1, B = A + 1;

...which may seem counterintuitive, but since the type of `A` must have been fully-inferred
before the start of the declaration of `B`, this is also completely well-defined. We add a
Constraint that `A` and `B` must infer the same type for the placeholder, which should
clarify to the user how this will work in most cases.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3550][0]. Bolded text is new text when inlined into an existing sentence.

Modify 6.2.1 "Scopes of identifiers, type names, and compound literals", third
sentence of paragraph 7:

> ... just after the appearance of its defining enumerator in an enumerator list.
> An ordinary identifier that has an underspecified definition has scope that starts
> **immediately after its init declarator** is completed; if the same ordinary identifier
> declares another entity ...

Modify 6.7 “Declarations”:

Modify the first sentence of 6.7.1, paragraph 13 so that it does not imply a declaration
can have no type specifier:

> A declaration such that the declaration specifiers **designate an inferred type**
> **(6.7.3.1)** or that is declared with `constexpr` is said to be _underspecified_.

Delete the second sentence of the same paragraph to remove the restriction against more
than one identifier, and non-ordinary identifiers, in an underspecified declaration (from
"If such a declaration is not a definition ..." to "... the behavior is implementation-defined").

Delete footnote 126.

Add a forward reference to 6.7.3 "Type specifiers" for _inferred type_:

> Forward references: declarators (6.7.7), enumeration specifiers (6.7.3.3), initialization
> (6.7.11), **inferred type (6.7.3.1),** storage-class specifiers (6.7.2),
> type inference (6.7.10), type names (6.7.8), type qualifiers (6.7.4).

Modify 6.7.2 "Storage-class specifiers" to remove all mention of `auto`:

- Remove `auto` from the list in paragraph 1.
- Remove the second bullet-point, from paragraph 2, and delete footnote 128.
  Remove `auto` from what is currently the third bullet-point (about `constexpr`).
- Remove the entire second sentence ("`auto` shall ...") from paragraph 4.
- Remove `auto` from the first bullet-point in paragraph 11.
- Remove the final part of the sentence in paragraph 12 that forward-references 6.7.10
  ("and type inference ..."), moving "and" appropriately.
- Remove the entire paragraph 15.
- In footnote 132, change the typewriter-face word "`auto`" to the normal-face word
  "automatic" (this was not talking about the specifier anyway).

Modify 6.7.3 "Type specifiers":

Add a new entry to the list in 6.7.3.1 paragraph 1:

> ...  
> _atomic-type-specifier_  
> _struct-or-union-specifier_  
> _enum-specifier_  
> _typedef-name_  
> _typeof-specifier_  
> **_placeholder-type-specifier_**  

Delete the first part of the first sentence of paragraph 2, from "Except ..." up to the comma:

> **At** least one type specifier ...

Add a final bullet point to the multiset list in paragraph 2:

> ...  
> – enum specifier  
> – typedef name  
> – typeof specifier  
> **– placeholder type specifier**  

Add a new paragraph before paragraph 3:

> The placeholder type specifier may be used by itself, or appear at most once
> as part of any other type specifier multiset in the above list.

Add a new paragraph after paragraph 4:

> A placeholder type specifier that designates an _inferred type_ shall only
> appear among the declaration specifiers of a declaration with an initializer.

(this rules out use by `typedef`, `enum`, members, etc. Note that `auto` was already not
permitted to appear in parameter declarations.)

Add a reference to type inference in paragraph 5:

> Specifiers for structures, unions, enumerations, atomic types, and typeof specifiers are
> discussed in 6.7.3.2 through 6.7.3.6. Declarations of typedef names are discussed in
> 6.7.9. **Type inference is discussed in 6.7.10.** The characteristics of the other types
> are discussed in 6.2.5.

Delete paragraph 6.

Add two new paragraphs after paragraph 7:

> If the placeholder type specifier appears as part of any other multiset in the list above,
> it is ignored, and the multiset shall designate the same type as if it were not present.
>
> When the placeholder type specifier appears as the only type specifier in a
> specifier-qualifier-list, it designates an _inferred type_. The specific type designated is
> inferred from an initializer, as discussed in 6.7.10.<sup>footnote)</sup>
>
> <small>footnote) This means it only appears in the declaration specifiers of a declaration,
> and not in any other context where the name of a type is used (6.7.8).</small>

(This removes the need to point out what optional elements of a declaration appertain to, because
the designated type is now concretely associated with the `auto` specifier in the type position.)

Add forward references to 6.7.8 and 6.7.10 at the end of the section:

> Forward references: atomic type specifiers (6.7.2.4), enumeration specifiers (6.7.2.2),
> structure and union specifiers (6.7.2.1), tags (6.7.2.3), **type names (6.7.8),** type
> definitions (6.7.8), **type inference (6.7.10)**.

Modify example 4 in 6.7.7.3 "Array declarators" to remove keyword-highlighting from the word
"auto" in comments and either change it to the regular word "automatic" or delete it without
replacement (editorial discretion), as it has nothing to do with the specifier in this context.

Modify 6.7.8 "Type names":

Add a “Constraints” section before paragraph 2:

> **Constraints**
>
> A placeholder type specifier shall not appear as part of a type name.

(This is enough to rule out the use of `auto` in casts, `sizeof`, etc. contexts where it makes no
sense. It also forbids compound literals, which we may find a use case for allowing later.)

Significantly modify 6.7.10 "Type inference":

> **Syntax**
>
> _placeholder-type-specifier:_  
>   `auto`  
>
> **Constraints**
>
> A declaration for which all or part of the type is inferred shall contain
> **a placeholder type specifier (6.7.3)**.
>
> Each init declarator in such a declaration shall have the form:<sup>footnote)</sup>
>
>> _declarator_ `=` _initializer_
>
> <small>footnote) in other words, each declaration is always a definition of an object,
> with an explicit initializer.</small>
>
> An inferred type shall not be specified within the parameters of a function declarator.
>
> If the initializer is a braced initializer (6.7.11), there shall be an explicit initializer list,
> containing at least one initializer that is an assignment expression, or a braced initializer
> to which this constraint applies recursively. Each such assignment expression shall have the
> same type.<sup>footnote)</sup>
>
> <small>footnote) compatible types need not be the same.</small>
>
> If the initializer is a braced initializer, the declarator in the declaration shall be an
> array declarator.
>
> If there is more than one element of the initializer list, no initializer in the list shall
> have variably-modified type.
>
> If the declaration declares more than one ordinary identifier, the same type shall be inferred
> by each init declarator in the declaration. A declaration that declares more than one ordinary
> identifier shall not infer a variably-modified type.
>
> Applying the inferred derivation sequence to the inferred type shall produce either the
> same type as the type of the declared identifier, or a type such that its value may be converted
> to the type of the declared identifier by assignment.
>
> **Semantics**
>
> The placeholder type specifier designates an _inferred type_ that is deduced from the type
> of the initializer for an identifier.
>
> If the initializer is an assignment expression, the type of the declared identifier is the
> type of the initializer after lvalue, array to pointer or function to pointer conversion,
> additionally qualified by qualifiers and amended by attributes as they appear in the
> declaration specifiers, if any.<sup>167)</sup>
>
> If the initializer is a braced initializer, the type of the declared identifier is an array
> type, with the dimensions specified in its array declarator, and an innermost element type that
> is the type of the assignment expressions appearing within initializers of the initializer
> list,<sup>footnote A)</sup> additionally qualified by qualifiers and amended by attributes as
> they appear in the declaration specifiers, if any. If a size is not provided in the array
> declarator, it is deduced according to the rules for an array of unknown size (6.7.11).<sup>footnote B)</sup>
>
> <small>footnote A) No coercion or deduction of a common or composite type is performed,
> as the type of all initializers in the list is the same, per the constraints.</small>
>
> <small>footnote B) The type is therefore incomplete until the end of the initializer list.</small>
>
> If the declarator is an identifier,<sup>footnote)</sup> the inferred type is the type
> specified for the declarator after removing any qualifiers appearing in the declaration specifiers.
> Otherwise, the declarator is a derived declarator and the inferred type is the inferred type
> of the declarator after removing the declarator type derivation, recursively. The sequence of
> declarator type derivations and qualifiers so removed is the _inferred derivation sequence_.
>
> <small>footnote) ignoring parentheses and attribute specifiers.

NOTE that this wording does not place any explicit constraint upon nested brace levels,
because array types are not inferred from bracing level. Instead, existing constraints
from 6.7.11 apply.

... then continuing 6.7.10, delete the NOTE about defining structure and union types.

Delete existing footnote 168.

Replace the last sentence of paragraph 4 (in EXAMPLE 1):

> **The final type here is a pointer type, even though the declarator is not `*p`.**

Add a third nested braced block to EXAMPLE 2:

>         {
>             auto a = b,      // valid, uses "b" from outer scope
>                  b = a + 1;  // valid, uses completed "a" from before the comma
>         }

Add five new examples after paragraph 9 (EXAMPLE 6):

> **EXAMPLE 7**  When a variable is declared with a derived placeholder type, the inferred
> type must match the derived-from type of the initializer:
>
>     auto x = 10; // no derivation from auto
>     
>     auto * px1 = &x;       // valid, initializer is a pointer
>     auto const * px2 = &x; // valid, int * can be converted to int const *
>     auto * px3 = x;        // invalid, x does not have pointer type
>     
>     auto const cx = 10;        // valid, inferred type is int
>     auto const * pcx = &cx;    // valid, inferred type is int and the
>                                // inferred-derivation-sequence is (const *)
>     auto * pcx2 = &cx;         // also valid, inferred type is const int
>     
>     auto ** ppx1 = &px1;       // valid, initializer is a pointer to a pointer
>     auto const ** ppx2 = &px2; // valid, initializer is a pointer to a pointer to const
>                                // inferred type is int and inferred derivation sequence is (const **)
>     auto const ** ppx3 = &px1; // invalid, cannot convert to a pointer to pointer to const
>     
>     int y[10] = {};
>     
>     auto * py1 = &y;      // valid, py1 has type int(*)[10]
>     auto * py2 = y;       // valid, py2 has type int *
>     auto (*py3)[10] = &y; // valid, pointer to auto[10] has same derivations as pointer to int[10]
>     
>     int f (int, float);
>     auto * pf1 = f;              // valid
>     auto (*pf2)(int, float) = f; // valid, declared type derives from placeholder return type
>     int (*pf3)(auto, auto) = f;  // invalid, cannot derive from placeholder parameter type
>
> **EXAMPLE 8**  When a variable is initialized with a braced-initializer, it has an array type, with
> an innermost element type inferred from the non-array elements, and dimensions from its declarator:
>
>     auto a1[3] = { 1, 2 };        // type is int[3]
>     auto a2[] = { 1, 2 };         // type is int[2] (array type completed during initialization)
>     
>     auto a3 = { 1, 2, 3 };        // invalid, braces without an array dimension
>     auto a4[] = {};               // invalid, must have at least one element initializer
>     
>     auto a5[] = { [5] = 0 };      // type is int[6]
>     auto a6[3] = { [5] = 0 };     // invalid, [5] is not within the array
>     auto a7[] = { 1u, 2, 3.0 };   // invalid, element types are not the same
>     
>     auto a8[2][3] = { { 1, 2, 3 }     // type is int[2][3]
>                     , { 4, 5, 6 } }; 
>     auto a9[2][3] = { 1, 2, 3         // inner braces are optional
>                     , 4, 5, 6 };      // for multidimensional arrays
>
> **EXAMPLE 9**  Initializers nested within assignment expressions are not part of the main
> initializer and not subject to its constraints:
>
>     struct Vec3 { int x, y, z; };
>     struct Vec3 v3;
>     
>     struct Vec3 va1[2] = {
>         v3                  // OK - v3 and 1, 2, 3
>       , { 1, 2, 3 }         // can be different because nothing is inferred
>     };
>     
>     auto va2[2] = {
>         v3                  // invalid - cannot infer an element
>       , { 1, 2, 3 }         // type from inconsistent assignment expressions
>     };
>     
>     auto va3[2] = {
>         v3                  // OK - assignment expressions v3 and
>       , (struct Vec3) {     // the compound literal have the same type,
>         1, 2, 3             // <- these int expressions are part of an
>       }                     // assignment expression, not of the outer braced
>     };                      // initializer that is used for type inference
>
> **EXAMPLE 10**  When a declaration contains multiple identifiers to declare, the inferred
> type for each declarator needs to be the same, but the final object type can be different.
>
>     auto x = 10, y = 20;  // valid, both the same
>     auto w = 10, *z = &x; // valid, both infer int
>                           // even though object types are different
>     
>     auto a = 5, b = 6.0;  // invalid, infer different types
>     auto *c = &x, d = z;  // invalid, inferring different types (int and int*)
>                           // even though object types are the same
>
> **EXAMPLE 11**  When the placeholder type specifier is combined with any other type specifier,
> in the list of declaration specifiers, it is ignored and the type is not inferred:
>
>     static auto double xd = 10;        // valid, type is double not int
>     thread_local auto int xtl = 10.0;  // valid, type is int not double

Add a forward reference:

> **Forward references:** initialization (6.7.11).

If [N3580][6] has been integrated, modify 6.8.5.1, paragraph 2, to remove reference to `auto`
as a storage-class specifier:

> Storage-class specifiers **other than `constexpr` or `register`** shall not ...

and add an example:

> **EXAMPLE 3**  Because a selection header in the third form always has an initializer, it
> can be used with `auto` to deduce the object type:
>
>     if (auto x = foo ()) ...
>     
>     // is equivalent to:
>     if (auto x = foo (); x) ...
>
> The deduced object type must be convertible to a value with scalar type in order for the
> resulting statement to be valid:
>
>     if (auto a[] = { 1, 2, 3, 4 }) ...
>     
>     // is equivalent to:
>     if (auto a[] = { 1, 2, 3, 4 }; a) ... // valid, though invariant

N3580 should be integrated before this proposal if both are accepted.

Modify 6.9.1 "External definitions", second sentence of paragraph 2:

> **A placeholder type specifier** shall only appear in the declaration specifiers in an external
> declaration if **the declaration is a definition and the type is inferred from the initializer**.

(this sentence could also be removed outright, as the Constraint is expressed elsewhere too;
editor's discretion applies)

Remove implementation-defined behaviours J.3.13 (2) and (3).

Remove J.5.12 "Type inference".

(as currently specified in the text provided above, all behaviours should be well-defined; we might
choose to make some elements of this specification undefined again to allow for extensions like
`int (*f) (auto, auto) = ...`, but there is no precedent for this)

## Questions for WG14

Would WG14 like to explicitly allow declarators with forms other than plain identifiers
to be declared using type inference, in C2y?  
(WG14 voted YES to this question at the Spring 2025 meeting)

Would WG14 like to change the definition of `auto` from a storage-class specifier to a
placeholder-type specifier, in C2y?

Would WG14 like to remove the implementation-defined behaviour associated with underspecified
declarations introducing more than one identifier, making this well-defined in C2y?  
(WG14 voted YES to this question at the Spring 2025 meeting)

## Acknowledgements

Many thanks to Joseph Myers for a detailed review of previous revisions.

## References

[C2y public draft (N3550)][1]  
[C++23 public draft (N4950)][2]  
[C23 CD ballot comments][3]  
[N3007 Type inference for object definitions][4]  
[N3427 Unspecified sizes in definitions of arrays][5]  
[N3580 `if` declarations, v5.1][6]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3550.pdf
[2]: https://open-std.org/jtc1/sc22/wg21/docs/papers/2023/n4950.pdf
[3]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3067.doc
[4]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3007.htm
[5]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3427.pdf
[6]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3580.htm
