    Proposal for C2y
    WG14 3320

    Title:               Strong typedefs
    Author, affiliation: Alex Celeste, Perforce
    Date:                2024-08-20
    Proposal category:   New feature
    Target audience:     Compiler implementers, safety-critical users

## Abstract

C's `typedef` mechanism to create type aliases is useful for conveying intent
to the reader, but has no impact on the type system itself as the aliases it
creates are completely transparent. We propose enhancing the mechanism with a
new keyword and a new attribute in order to allow users to express more
constraints over how the newly created alias can be used, by constraining type
compatibility and implicit convertibility respectively.

-------------------------------------------------------------------------------

# Strong typedefs

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3320
    Revises:      N/A
    Date:         2024-08-20

## Summary of Changes

### N3320
 - original proposal

## Introduction

The `typedef` storage class enables users to declare new names for types, in
a fashion syntactically identical to declaration of objects or functions:

    typedef int Integer;
    // Integer can be used anywhere int can

This is a wonderful tool for enhancing readability and for conveying developer
intent. In addition to aliases provided by the Standard Library to name specific
sizes of builtin types, aliases can also communicate when a value is intended
to mean a quantity, a difference, or some real-world unit like a length or
duration:

    hours trip = 6;  // duration of an excursion
    
    kilometres length = 10;  // of a road, but readably not
                             // of a movie or a shelf

Unfortunately, because these are _only_ aliases, there is no type system
enforcement that they are used consistently, or at all:

    length = trip;  // in isolation, the names do not indicate a
                    // problem - length could be a duration, trip
                    // could be a distance

There are many other possible examples, such as unit conflicts within the same
physical dimension
(e.g. [mixing SI and Imperial measurements of the same value][2] to catastrophic
effect), or even mixing up dimensions entirely, such as confusing speed with
acceleration, or time with temperature. It is also reasonable not to want values
with the same meaning but different local domains within a design to be able to
mix (e.g. length of different non-interacting components).

The only straightforward way to express this in C23 is to use a tag type, which
is the only mechanism for creating a completely new type that is not compatible
with any existing type in the program. (Type derivations are designed to work in
the opposite way: the same sequence of derivations always produces _the same_
type.) This is potentially useful:

    typedef struct Metres { double val; } Metres;
    typedef struct Yards  { double val; } Yards;
    
    Metres height1 = { 6.5 };
    Yards  height2 = { 8.4 };
    
    height1 = height2; // Error: types are not compatible (great!)
    
    height1 = imperialToSi (height2); // Correct!

...but limited by the need to "unbox" when the values are used concretely:

    Metres addDistance (Metres m1, Metres m2) {
        return { m1.val + m2.val };
    }
    
    // which means it is still completely possible to write:
    Metres addDistance (Metres m1, Yards m2) {
        return { m1.val + m2.val };
    }

As demonstrated, this means the type-safety can only really be enforced by this
means at an abstract interface level, and evaporates at the expression level. The
resulting expressions are also inelegant, and not all ABIs support this idiom as
efficiently as if the builtin types were used (mostly for historical reasons).

C would therefore be dramatically improved by a feature which allowed a similar
degree of type-checking that integrates seamlessly with the expression language.

### Related work

In [N3321][6] we propose a related mechanism to allow the user to define new
type qualifiers. A qualifier creates a distinct type from the unqualified type,
so creating new qualifiers is another way to allow the user to arbitrarily
create new types.

However, the mechanisms are quite distinct: `[[strong]]` creates types that
describe _values_, whereas qualifiers are always a property of the _objects_
that hold values. A qualifier on a type is always discarded during value
conversion, whereas the goal of this proposal is to allow the definition of
useful value types that thread through evaluation of an expression. Therefore,
while related, the proposals serve distinct purposes.

## Proposal

We propose two new features are added to the language, with semantics that
directly complement each other: one new attribute, and one new keyword.

The spelling of either feature can be altered and is not critical.

### `[[strong]]`

The `[[strong]]` attribute appertains to a type definition. When a type is
named using the declared identifier, the attribute requires it to be used
_consistently_ such that it is never required to be compatible with, or
implicitly convertible to, a type not named by that identifier:

    [[strong]] typedef double Metres;
    [[strong]] typedef double Yards;
    
    Metres d1 = 6.5;
    Yards  d2 = 8.4;
    d1 = d2;  // Error: attempting to convert from a type named by
              // [[strong]] Yards to another type
    
    double abstract;
    abstract = d1;  // Error: attempting to convert from a type named
                    // by [[strong]] Metres to a non-strong type

This adds a Constraint. The Constraint is suitable to be introduced and
enforced by an attribute because any program that correctly respects the
introduced Constraint would remain correct and continue to have the same
meaning if the attributes were removed.

Additional typedefs "over" a strong typedef could add new constraints:

    [[strong]] typedef Metres HullLength;
    
    HullLength hl = 10.8;
    d1 = hl;  // Error: converting domain-specific length to metres

A typedef of a strong type that does not include the attribute behaves as
a completely transparent alias for that type, just as it does in C23:

    typedef Metres M2;
    
    M2 d3 = 12.34;
    d3 += d1;  // fine, fully interchangeable
    d1 -= d3;  // likewise
    
    d2 = d3;        // Error, unrelated strong types
    abstract = d3;  // Error, implicit conversion to non-strong type

...but with this permissiveness only existing "above" the typedef.

Implicit conversions from non-strong to strong types should be permitted:

    d1 = abstract;  // double to meaningful Metres
    d2 = abstract;  // double to meaningful Yards
    
    hl = d1;  // Metres to HullLength (in metres)
    hl = d2;  // Error, cannot convert Yards to HullLength

This allows the user to define hierarchies of meaning and also makes
initialization significantly simpler, as values have to originate from
somewhere. (It would be possible to restrict this permission only to
initialization, but allowing it generally seems more expressive.)

Expressions requiring the _usual arithmetic conversions_ are permitted
if one operand is assignable to the other according to the above rules.
The result expression inherits the strong type of the operand and
propagates it to any containing expression, even if the common real type
of the result is not the same as the strong operand type:

    [[strong]] typedef short T1;
    
    T1 x, y, z;
    (x + y) + z;  // OK, (T1 + T1) still has strong type T1
                  // despite having representation type int
    
    [[strong]] typedef short T2;
    T2 s, u, v;
    (x + y) + (s + u);  // Error: (T1 + T1) + (T2 + T2)
                        // even though the toplevel is int + int
    
    (x + y) + (double)v;  // OK, double is convertible to the
                          // underlying type of T1

An explicit cast always overrides strong typing (but should be used
sparingly).

### `_Newtype`

The `_Newtype` keyword behaves _exactly_ like the `typedef` keyword for
syntactic purposes, acting as a storage class and creating identifiers with
no linkage in exactly the same way.

The keyword differs from `typedef` in one respect only, which is that the
declaration creates a _new_ type with the exact same representation and
other properties as the type used in the declaration, but which is _not_
_compatible_ with that declaring type (i.e. it is [generative][5]):

    typedef int Int;
    _Generic (x
        , int: 0
        , Int: 1); // Constraint violation: int appears twice
    
    _Newtype float Real;
    _Generic (x
        , float: 0
        , Real:  1); // OK, two distinct, incompatible types

This functionality requires a keyword rather than an attribute as the
behaviour would change if `_Newtype` was replaced by `typedef`.

By default, a type created by `_Newtype` is implicitly convertible to the
underlying type, which is always a NOP:

    typedef int Int1;
    typedef int Int2;
    
    Int1 i1 = 0;
    Int2 i2 = 1;
    int i3 = 2;
    
    i1 = i2;         // OK
    i3 = i1;         // OK
    (i2 + i3) * i1;  // OK

In addition, if a tag type is aliased with `_Newtype`, the resulting new
tag type is also implicitly convertible to the underlying type, and vice
versa, even though normally tag types have no conversions.

`_Newtype` creates a type without restrictions against implicit conversion
because if such restrictions are required, they can be added with `[[strong]]`
at the same time:

    [[strong]] int Int1;
    [[strong]] int Int2;
    
    Int1 x;
    Int2 y;
    x = y;  // Error

## Prior Art

The `[[strong]]` attribute draws heavily on ideas found in [MISRA C][2]'s 
Essential Type system, which defines a secondary layer on top of the C core
type system, requiring a form of strong typing that persists across the
implicit conversions of arithmetic and initialization. Tools that implement
the Essential Type system are required to provide the functionality to type
check expressions that perform a C conversion while preserving the operand
type, so for instance:

    typedef unsigned short US;
    
    US x, y, z;
    z = x + y;  // OK even though the type of (x + y) is int
    
    signed short u, v;
    z = u + v;  // Not OK - (u + v) is Essentially Signed

The MISRA type system is slightly different and is roughly similar to what
would be the result of every type having `[[strong]]` splitting the signed
and unsigned types, the real floating and the complex floating types, etc.
Unlike in this proposal, _every_ C expression has a MISRA Essential Type,
although there are some relaxations for constant expressions and conversions
that do not lose value range.

Overall the Essential Type system is mostly focused on preservation of value
ranges and precisions and is not a type system focused on domain or on
expressiveness, so the exact set of permitted conversions and operations
is not the same and it is not possible to define new restrictions. However,
from an implementation perspective, support is essentially identical as it
requires the tool to process types in a way that is distinct from representation.

This is especially evident with the Essentially Boolean types, which are
required to be supported even in C90 mode; this means a tool must be able to
be configured _in some way_ to understand typedefs separately from their
representations:

    // configuration: -intrinsictype "bool=Boolean"
    
    typedef int Boolean;
    extern int x, y;
    
    int r1 = (x == y); // MISRA error:
                       // assigning Essentially Boolean value to int
    Boolean r2 = (x == y);  // MISRA is OK with this
                            // Tool needs to be able to distinguish the type

In practice this means a tool capable of enforcing the Essential Type system
must already have implicit support for _the same mechanism as_ `[[strong]]`,
although it may not expose any general way of invoking it to the user.

`_Newtype` has equivalents in [Standard ML and related languages][4] where
it is possible to define a module with opaque types. `datatype` with a single
constructor (and its equivalents in other languages) can also often be used
in a similar practical fashion, although what it does is closer to the "box"
idiom (and it can unfortunately be misused in the same way).

C++ of course allows for any type to be boxed within a class:

    StrongType <int, struct MY_TAG_1> x;
    StrongType <int, struct MY_TAG_2> y;
    
    x = y;  // type error

`StrongType` can be defined to allow implicit conversions only one way or neither.
However, this requires significant language machinery not available in C.
It is also possible to define any number of drop-in replacements for the basic
arithmetic types thanks to operator overloading, which is also not available in C
and is also very verbose even in C++. This also has similar _de-facto_ ABI
limitations to the "box" idiom, for the same reasons.

## Impact

This proposal describes an original feature which only adds functionality
and therefore there is no impact against existing code.

Implementation is relatively straightforward and considerably simpler than
compiler support for the overloading/class-based solution that is idiomatic
in C++. Any tool that can currently enforce the MISRA C Essential Type system
can enforce the `[[strong]]` attribute, while tools that do not do this can
also simply choose to ignore the attribute and leave enforcement to others.
`_Newtype` can be treated as equivalent to `typedef` in most contexts, while
_all_ compilers already have the ability to introduce new types through the
`struct` keyword; all this feature requires is that they can be used like the
underlying type.

Implementations already generally have the ability to understand that two
types can be distinct but identical, as (for instance) on most targets `long`
will be identical to either `int` or to `long long` and only be distinguished
at compile-time by the type system. `char` is similarly already defined by the
Standard to be identical to either `signed char` or `unsigned char` and again
cannot be distinguished from either one at runtime.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3220][1]. Bolded text is new text when inlined into an existing sentence.

These changes assume the spellings described above. The features can be
renamed without substantially affecting the proposed wording changes.

### `[[strong]]`

This change should be introduced in a single place, in the clauses describing
attributes:

Modify 6.7.13.2, paragraph 2, to add the new attribute name:

> `deprecated`  
> `fallthrough`  
> `maybe_unused`  
> `nodiscard`  
> `noreturn`  
> `_Noreturn`  
> `unsequenced`  
> `reproducible`  
> **`strong`**  

Add a new section after 6.7.13.8:

> #### 6.7.13.9 The `strong` attribute
>
> **Constraints** The `strong` attribute shall be applied to the declaration of
> a typedef name.
>
> **Semantics** The **strong** attribute marks a typedef name as _strongly-typed_.
>
> The `__has_c_attribute` conditional inclusion expression (6.10.2) shall return
> the value _XXXXXXL_ when given `strong` as the pp-tokens operand if the
> implementation supports the attribute.
>
> **Recommended Practice** Implementations are encouraged to issue a diagnostic
> when a value originating from a declaration using a _strongly-typed_ typedef name
> is converted to a value, or assigned to an lvalue, originating from a declaration
> that did not use the same typedef name (possibly via a second typedef name that
> was itself declared using the first typedef), other than by means of an explicit
> cast; except when the conversion is part of the usual arithmetic conversions,
> in which case the same check is moved to any use of the result of the subexpression.
>
> _Strongly-typed_ typedefs behave as though a unique, virtual qualifier is added
> onto a type at the position within the type derivation that the typedef name was
> used. Unlike a concrete qualifier, the virtual qualification is not removed by
> value conversion, and may be "added" at any level of derivation.
>
> EXAMPLE 1  No diagnostic is encouraged against conversions _to_ strong types,
> in order to permit initialization, but conversion between strong types should
> result in a diagnostic:
>
>     // define two distinct strong typedefs
>     [[strong]] typedef double Metres;
>     [[strong]] typedef double Yards;
>     
>     Metres d1 = 6.5;  // initialization is OK
>     Yards  d2 = 8.4;
>     
>     d1 = d2;  // Diagnose attempt to convert between Yards and Metres
>     d2 = d1;  // Diagnose attempt to convert between Metres and Yards
>     
>     double abstract;
>     abstract = d1;  // Diagnose attempt to convert to underlying type
>     d1 = abstract;  // OK - strengthening conversion
>
> On the assignment `d1 = d2`, the proper reason for the warning is the conversion
> from a value strongly-typed as `Yards` to _any_ type not strongly-typed as `Yards`.
> (The warning is on the loss of a strong type rather than on a conversion "between"
> strong types, but the effect for the user is the same.)
>
> EXAMPLE 2  A typedef of a strong type inherits its strong type, and also has the
> opportunity to introduce a new layer of typing:
>
>     [[strong]] typedef int Int1;
>                typedef int Int2;
>     Int1 i1;
>     Int2 i2;
>     
>     i1 = i2;  // OK - these are aliases for the same strong type
>     i2 = i1;  // just like a normal typedef
>     
>     [[strong]] typedef Int1 Int3; // more "refinement"
>     Int i3;
>     
>     i3 = i1;  // OK - strengthening implicit conversion
>     i1 = i3;  // Diagnose attempt to convert Int3 to an underlying type
>
> EXAMPLE 3  When a strongly-typed value is converted by arithmetic conversions,
> the result of the subexpression inherits the strong type, regardless of representation:
>
>     [[strong]] typedef short T1;
>     
>     T1 x, y, z;
>     z = (x + y);  // OK - the result is an int, but still strongly-typed
>                   // as T1 and can be converted on assignment to z
>     int w;
>     w = (x + y);  // Diagnose attempt to convert T1 to int, even though
>                   // the representation type is int
>     
>     [[strong]] typedef short T2;
>     T2 s, u, v;
>     z = (x + y) + (u + v);  // Diagnose attempt to convert a result to T1
>                             // which has inherited two strong types T1 and T2
>     s = (x + y) + (u + v);  // Same issue, converting (T1+T2) to just T2
>
> In the example above, the result expression inherits two strong types with distinct
> origins, so there is no valid assignment for the result as conversion would have to
> lose either `T1` or `T2`. In contrast, an expression that inherits two strong types
> with related origins can still be used as a value of the "stronger" type:
>
>     [[strong]] typedef T1 T3; // T3 strengthens T1
>     T3 s2, u2, v2;
>     
>     z = (x + y) + (u2 + v2);  // Diagnose attempt to convert a result to T1
>                               // which has inherited strong types T1 and T3
>     
>     s2 = (x + y) + (u2 + v2); // OK - inheriting T1 and T3 is just a T3
>
> EXAMPLE 4  A derived type can introduce a strong type at any level, unlike a
> qualifier which can only be added by implicit conversion in select circumstances:
>
>     [[strong]] typedef size_t Metres;
>     [[strong]] typedef size_t Yards;
>     
>     // weakly-typed function signature
>     size_t addLengths (size_t l, size_t r);
>     
>     // allow conversion to a strongly-typed function derivation
>     typeof (Metres (Metres, Metres)) * addMetres = addLengths;
>     typeof (Metres (Yards,  Yards))  * addYards  = addLengths;
>     
>     addMetres = addYards; // Diagnose an attempt to convert a derivation
>                           // from Yards to a derivation without Yards
>
> EXAMPLE 5  An explicit cast always succeeds (without the diagnostic emitted for
> an implicit conversion that would discard a strong type):
>
>     Metres m;
>     Yards y;
>     
>     m = y;  // Diagnose attempt to convert Yards to non-Yards
>     y = m;  // Diagnose attempt to convert Metres to non-Metres
>     
>     m = (Metres)y;  // Succeeds without the diagnostic for loss of strong type 
>     m = (double)y;  // Similarly successful
>
> An implementation might choose to emit a secondary diagnostic against the cast
> directly between unrelated strong types, but should not use the main warning.

### `_Newtype`

We do not introduce a new term of art and consider the identifiers introduced by
`_Newtype` to be a second kind of _typedef-name_. In most of the document the
term is not used as a monospaced-keyword and does not need to change.

Add a new clause to 6.3.2 "Other operands", after 6.3.2.4:

> **6.3.2.5 `_Newtype`s**
>
> Any type created by means of the `_Newtype` specifier may be converted to the
> type used in its definition, and vice-versa. Such a conversion shall not change
> the representation of the value.

Add the new keyword to 6.4.1 Keywords:

> `_Newtype`

Modify 6.5.17.2 "Simple assignment", paragraph 1, adding a bullet point to
the end of the list:

> - the left operand has a type created by the `_Newtype` specifier from the type
> of the right operand; or the right operand has a type created by the `_Newtype`
> specifier from the type of the left operand.

Modify 6.7.9 "Type definitions":

Modify paragraph 3, breaking it into four new paragraphs:

> In a declaration whose storage-class specifier is `typedef` **or `_Newtype`**,
> each declarator defines an identifier to be a typedef name that denotes the
> type specified for the identifier in the way described in 6.7.7. Any array size
> expressions associated with variable length array declarators and typeof operators
> are evaluated each time the declaration of the typedef name is reached in the
> order of execution.
>
> A `typedef` declaration does not introduce a new type, only a synonym for the type
> so specified. That is, in the following declarations:
>
>     typedef T type_ident;
>     type_ident D;
>
> `type_ident` is defined as a typedef name with the type specified by the declaration
> specifiers in `T` (known as _T_), and the identifier in `D` has the type
> "_derived-declarator-type-list T_" where the derived-declarator-type-list is
> specified by the declarators of `D`.
>
> **A `_Newtype` declaration has the same effect as the `typedef` specifier except**
> **that a new type is also defined alongside each declarator in the declaration,**
> **with an identical representation to the type specified in the declaration, but**
> **not compatible with it. That is, in the following declarations:**
>
>     _Newtype T type_ident;
>     type_ident D1;
>     T D2;
>
> **the identifier in D1 has a distinct and incompatible type from the identifier in**
> **D2, but their representations are the same.**
>
> **The type defined by a `_Newtype` declaration is convertible to the type used in
> **its definition, and vice versa.**
>
> A typedef name shares the same name space as other identifiers declared in ordinary
> declarators. If the identifier is redeclared in an enclosed block, the type of the
> inner declaration shall not be inferred (6.7.10).

Add a new EXAMPLE after paragraph 8:

> EXAMPLE 6  The `typedef` and `_Newtype` storage classes are the same except that
> `typedef` creates a transparent alias for a type, while `_Newtype` creates a new
> and incompatible type:
>
>     typedef  int Int1;
>     _Newtype int Int2;
>     
>     int x;
>     _Generic (x
>         , int : 1
>         , Int1 : 2); // Constraint violation, int specified twice
>     
>     _Generic (x
>         , int : 1
>         , Int2 : 2); // two distinct types specified, result is 1
>     
>     Int2 y;
>     y = x; x = y;    // conversion both ways is a no-op

## Questions for WG14

Would WG14 like to add something along the lines of the `[[strong]]`
attribute into C2y?

Would WG14 like to add something along the lines of the `_Newtype`
keyword into C2y?

Would WG14 like to make use of either `[[strong]]` or `_Newtype` in the
specification of any library features?

## References

[C2y latest public draft][1]  
[Mars Climate Orbiter][2]  
[MISRA C 2023][3]  
[Definition of Standard ML (opaque signature matching)][4]  
[User-defined types in ML][5]  
[User-defined qualifiers][6]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf
[2]: https://en.wikipedia.org/wiki/Mars_Climate_Orbiter
[3]: https://misra.org.uk/product/misra-c2023/
[4]: https://smlfamily.github.io/sml97-defn.pdf#page=114
[5]: https://courses.cs.washington.edu/courses/cse341/04wi/lectures/07-ml-user-types.html
[6]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3321.html
