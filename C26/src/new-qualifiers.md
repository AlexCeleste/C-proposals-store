    Proposal for C2y
    WG14 3321

    Title:               User-defined qualifiers
    Author, affiliation: Alex Celeste, Perforce
    Date:                2024-08-20
    Proposal category:   New feature
    Target audience:     Compiler implementers, users

## Abstract

C's qualifier mechanism provides an elegant way to mark objects as having
specific properties, and of specifying types that take these properties into
account for object access; however, it is limited by the fact that the set of
qualifiers available is fixed to only those with some meaning directly
defined in the core language. Adding the ability for the user to define new
qualifiers that behave according to the same rules as the access-qualifiers
would allow users to encode domain logic directly into the type system in a
more effective way, by describing the accesses their _program_ can perform on
objects in the types of their APIs.

-------------------------------------------------------------------------------

# User-defined qualifiers

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3321
    Revises:      N/A
    Date:         2024-08-20

## Summary of Changes

### N3321
 - original proposal

## Introduction

C provides a small set of _access qualifiers_ (to use the term from [Embedded C][2])
that work to add constraints on the way an object is accessed through an lvalue.
These encode information about access into the type system so that only APIs that
conform to particular requirement can access these classes of objects.

It would be immensely useful for users and implementations to also be able to make
similar promises and enforce similar restrictions about what can be done with a
value, within the domain-specific logic of a given program.

Therefore, we propose to extend the existing core set of qualifiers with an
arbitrarily-extensible group of added qualifiers that behave along similar
lines in the type system, but that have no predefined meaning to the compiler.
Instead, their meaning is entirely determined by the APIs the user creates to
handle them.

### Related work

In [N3320][3] we propose a related mechanism to allow the user to define type
aliases that specify distinct types from their declaring type.

A qualifier creates a distinct type from the unqualified type, so adding new
qualifiers is also a way to allow the user to arbitrarily create new types.

However, the mechanisms are quite distinct: `[[strong]]` creates types that
describe _values_, whereas qualifiers are always a property of the _objects_
that hold values.
The goal of this proposal is to allow the definition of objects marked as
having distinct properties within the domain logic of the program, rather
than to create distinct classes of value expression that persist through
nested expression evaluation.
Therefore, while related, the proposals serve distinct purposes.

## Proposal

We propose to add a new keyword, `_Qualifier`, which declares an identifier as
a new qualifier for use on any object type. The newly-added qualifier is not
one of the builtin access qualifiers and does not imply any of their properties,
but within the type system, behaves in a similar way.

    _Qualifier QA;  // new qualifiers
    _Qualifier QB;
    
    int i1;
    int const i2;
    int QA i3;
    
    int * p1 = &i1;
    p1 = &i2;  // error: loses const from the pointed-to type
    p1 = &i3;  // error: loses QA from the pointed-to type
    
    int const * p2 = &i1;  // OK, implicitly adds qualification to pointed-to
    p2 = &i2;  // OK
    p2 = &i3;  // error: loses QA from the pointed-to type
    
    int const QA * p3 = &i1; // OK, implicitly adds const and QA
    p3 = &i2;  // OK, implicitly adds QA
    p3 = &i3;  // OK, implicitly adds const

The existing rules for the three _access qualifiers_ are elegant and easy to
remember. It would be possible to propose more complicated declarations for more
powerful and/or convoluted forms of qualifier (starting with `_Atomic` with the
rule that it cannot be added implicitly), but would be overly complicated.

## Prior Art

The [Embedded C TR][2] adds named address spaces as a feature to the addressing
model of C. Pointers into a given address space are represented by a qualified
pointer, and objects are defined into an address space by qualifying their
definition with the same qualifier.

The qualifiers themselves are not specified and are left implementation-defined.
The TR does not introduce implicit conversion of a pointer to an address-qualified
pointer, instead requiring a cast to change address space regardless of nesting.
In this proposal, since the implicit conversions can be defined in the declaration
of the qualifier, this can be replicated or a system-declared address-space
qualifier could choose to be as restrictive as in the TR.

Our proposal does not include the concept of nesting because nesting can be
achieved by simply only using one qualifier in conjunction with another
qualifier at all times, e.g.:

    // QB defined to nest inside QA
    
    QA void * p1;     // QA-qualified
    QA QB void * p2;  // OK: QB-qualified within the QA-space
    QB void * p3;     // Design-level error: QB is meaningless without QA

The general qualifier system can therefore easily model the address-space
qualification system from the TR.

We have implementation experience with generalized qualifiers in QAC.
QAC makes heavy use of additional qualifiers with all variants of the different
implicit conversion rules, to mean several different things. These are used for
type inference, mutability analysis, and more. Our experience is that it is
very easy for a type system capable of representing the basic CVA-qualifiers to
add more qualifiers for other purposes.

## Impact

The proposal requires a new keyword and a new form of declaration very similar to
`typedef` to be added to the language. This is a simple addition.

Supporting an _arbitrary_ number of qualifiers would probably require significant
changes to some C type systems that have been implemented with the assumption that
type qualification can be represented with a very small number of flag bits.
In order to avoid disrupting these implementations and forcing heavy rewrites, we
suggest that a reasonably low translation limit for the number of qualifiers an
implementation is required to be support is appropriate, based on the likely
number of bits available for a flag on low-end systems.

## Future directions

A typedef-like mechanism to define qualifiers in terms of other qualifiers, without
fixing the value type, could be useful but is not necessary:

    _Qualifier QA;
    _Qualifier QB;
    
    // want to define a combined qualifier QA and QB?
    <new keyword> QA QB QAB;
    
    #define QAB QA QB  // works almost as well
    typedef QA QB int QABint;  // good enough in practice

We could add this at a later time if it turned out to be helpful in defining more
qualifier-generic interfaces along the lines of `memchr`.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3220][1]. Bolded text is new text when inlined into an existing sentence.

(We are not bound to the particular keyword `_Qualifier` or the use of `_Ugly`
keyword spelling in general.)

Modify 6.2.3 "Name spaces of identifiers", paragraph 1, last bullet:

> - all other identifiers, called _ordinary identifiers_ (declared in ordinary
>   declarators, qualifier declarators, or as enumeration constants).

(This explicitly puts qualifiers in the ordinary namespace, avoiding ambiguity
with typedef names.)

Add a forward reference:

> **Forward references:** enumeration specifiers (6.7.3.3), labeled statements
> (6.8.2), structure and union specifiers (6.7.3.2), structure and union members
>(6.5.3.4), tags (6.7.3.4), **user qualifiers (6.7.4.3),** the goto statement (6.8.7.2).

Modify 6.2.5 "Types", paragraph 31:

> Any type so far mentioned is an unqualified type. Each unqualified type has
> several qualified versions of its type,40) corresponding to the combinations
> of one, two, or all three of the `const`, `volatile`, and `restrict` **_access_**
> **_qualifiers_, or any number of identifiers declared with the `_Qualifer`**
> **keyword to represent _user qualifiers_.** The qualified or unqualified
> versions of a type are distinct types that belong to the same type category ...

Add a forward reference at the end of 6.2.5:

> **Forward references:** compatible type and composite type (6.2.7), declarations
> (6.7)**, user qualifiers (6.7.4.3)**.

NOTE that no change is needed to 6.3.2.3, 6.5.16, etc.

Add the new keyword to 6.4.1 Keywords:

> `_Qualifier`

Modify 6.7 "Declarations", adding a production to _declaration_:

> _declaration:_  
>   _declaration-specifiers_ _init-declarator-list <sub>opt</sub>_ `;`  
>   _attribute-specifier-sequence_ _declaration-specifiers_ _init-declarator-list_ `;`  
>   _static_assert-declaration_  
>   _attribute-declaration_  
>   **_qualifier-declaration_**  

Modify paragraph 6, last bullet:

> - a typedef **or qualifier** name, is the first (or only) declaration of the
>   identifier.

Modify 6.7.4 "Type Qualifiers":

Add to the syntax, in 6.7.4.1 "General":

> _type-qualifier:_  
>   `const`  
>   `restrict`  
>   `volatile`  
>   `_Atomic`  
>   **_user-qualifier_**  
>
> **_user-qualifier:_**  
>   **_identifier_**  

Add a new paragraph after paragraph 4:

> The identifier in a _user-qualifier_ shall have a visible declaration as
> an ordinary identifier that declares a qualifier.

Add a new paragraph after paragraph 9:

> An object qualified by a _user qualifier_ shall be accessed through an lvalue
> with a type qualified by the same _user qualifier_.

(ED: "it doesn't do anything but we didn't forget")

Add a new subclause to 6.7.4 after 6.7.4.2:

> ### 6.7.4.3 User qualifiers
> 
> **Syntax**
> 
> _qualifier-declaration:_  
>   `_Qualifier` _identifier_ `;`  
> 
> **Semantics**
> 
> In a declaration with the `_Qualifier` keyword, the identifier is defined as the
> name of a _user qualifier_. The qualifier is distinct from any other qualifier
> with a different name. If a qualifier with the same name is already visible in
> the scope, no new qualifier is added and the name continues to refer to the same
> qualifier.
> 
> A _user qualifier_ can be used to specify a _qualified type_, in the same fashion
> as the _access qualifiers_ `const`, `volatile` and `restrict`. Unlike the access
> qualifiers, a user qualifier does not give any special properties to an object
> that is defined with the qualifier or any special behavior upon accesses through
> an lvalue with the qualified type.
>
> A type qualified by a _user qualifier_ is a _user-qualified_ type.
> 
> **Recommended practice** A user qualifier is intended to communicate properties
> defined within the program's logic.
> 
> EXAMPLE  A _user qualifier_ follows the same rules as an _access qualifier_ in
> terms of implicit conversions:
> 
>     _Qualifier QA;  // define user qualifiers QA and QB
>     _Qualifier QB;
>     
>     int i1;
>     int const i2;
>     int QA i3;
>     
>     int * p1 = &i1;
>     p1 = &i2;  // invalid: conversion would lose const from the pointed-to type
>     p1 = &i3;  // invalid: conversion would lose QA from the pointed-to type
>     
>     int const * p2 = &i1;  // implicitly adds qualification to pointed-to
>     p2 = &i2;  // OK
>     p2 = &i3;  // invalid: conversion would lose QA from the pointed-to type
>     
>     int const QA * p3 = &i1; // OK, implicitly adds const and QA
>     p3 = &i2;  // OK, implicitly adds QA
>     p3 = &i3;  // OK, implicitly adds const

Modify 7.24.5.1 "The `bsearch` generic function", paragraph 5:

> The `bsearch` function is generic in the qualification of the type pointed to
> by the argument `base`. If this argument is a pointer to a `const`-qualified
> **or user-qualified** object type, the returned pointer will be a pointer to
> **`void` with the same qualification**. Otherwise, the argument shall be a
> pointer to an unqualified object type or a null pointer constant,<sup>354)</sup>
> and the returned pointer will be a pointer to unqualified `void`.

Modify 7.26.5 "Search functions", paragraph 1:

> The stateless search functions in this section (`memchr`, `strchr`, `strpbrk`,
> `strrchr`, `strstr`) are _generic functions_. These functions are generic in the
> qualification of the array to be searched and will return a result pointer to
> an element with the same qualification as the passed array. If the array to be
> searched is `const`-qualified **or user-qualified**, the result pointer will
> be to **an element with the same qualification**. If the array to be searched is
> not `const`-qualified **or user-qualified**,<sup>364)</sup> the result pointer
> will be to an unqualified element.

Modify 7.32.4.5 "Wide string search functions", paragraph 1:

> The stateless search functions in this section (`wcschr`, `wcspbrk`, `wcsrchr`,
> `wmemchr`, `wcsstr`) are _generic functions_. These functions are generic in
> the qualification of the array to be searched and will return a result pointer
> to an element with the same qualification as the passed array. If the array to
> be searched is `const`-qualified **or user-qualified**, the result pointer will
> be to **an element with the same qualification**. If the array to be searched
> is not `const`-qualified **or user-qualified**,<sup>412)</sup> the result pointer
> will be to an unqualified element.

(ED: unsure about whether the "supports all correct uses" wording should be
removed or not, since user qualifiers don't "do" anything and can be safely
removed by a cast)

## Questions for WG14

Would WG14 like to add a mechanism for declaring new type qualifiers to C2y,
along the lines of N3321?

## References

[C2y latest public draft][1]  
[Embedded C TR 18037][2]  
[Strong typedefs][3]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf
[2]: https://www.iso.org/standard/51126.html
[3]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3320.html
