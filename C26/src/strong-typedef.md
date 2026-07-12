    Proposal for C2y
    WG14 3587

    Title:               Strong typedefs
    Author, affiliation: Alex Celeste, Perforce
    Date:                2026-06-01
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
    Document No:  N3587
    Revises:      N3320
    Date:         2026-06-01

## Summary of Changes

### N3587
 - changes in response to Committee feedback
   (keeping the attribute _for now_)
 - accepting Committee feedback on VWB here too, to prefer Constraints over RP
 - make the text significantly more normative
 - rebase on working draft N3886

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

### Proposal history

This proposal was previously discussed in Fall 2024, where the Committee indicated:

 - strong enthusiasm for "something along these lines" of the core `[[strong]]`
   feature
 - tentative support, pending further investigation, for `_Newtype`
 - no real direction on whether `[[strong]]` should become a keyword

In response to this feedback, we have simplified the proposal by removing `_Newtype`
for the time being; it will return as an independent feature proposal.

### Related work

In [N3321][6] we proposed a related mechanism to allow the user to define new
type qualifiers.

This proposal was strongly rejected by the Committee. As a result, the parts of these
features that would impact type qualification have been removed.

## Proposal

We propose one new features to add to the language, currently in attribute
form but with scope to be promoted to a keyword later.

The spelling of the feature can be altered and is not critical.

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
although it might not expose any general way of invoking it to the user.

## Impact

This proposal describes an original feature which only adds functionality
and therefore there is no impact against existing code.

Implementation is relatively straightforward and considerably simpler than
compiler support for the overloading/class-based solution that is idiomatic
in C++. Any tool that can currently enforce the MISRA C Essential Type system
can enforce the `[[strong]]` attribute, while tools that do not do this can
also simply choose to ignore the attribute and leave enforcement to others.

(Implementation of the currently-withdrawn `_Newtype` feature is even simpler
and requires no new compiler machinery at all.)

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3886][1]. Bolded text is new text when inlined into an existing sentence.

These changes assume direction to adopt as an attribute.
Adding a keyword would integrate similar wording elsewhere in the document.

### Constraints and conformance

Modify 5.2.1.3 "Diagnostics", paragraph 1:

> A conforming implementation shall produce at least one diagnostic message (identified in an
> implementation-defined manner) if a preprocessing translation unit or translation unit contains
> a violation of any syntax rule or constraint, even if the behavior is also explicitly specified
> as undefined or implementation-defined **, unless the rule or constraint is introduced by an**
> **optionally-supported attribute that the implementation does not support.** Diagnostic messages
> are not required to be produced in other circumstances.

This allows _implementations_ to remain conforming without actively enforcing the constraints
introduced by `bind_var` and `bind_type`.

(**NOTE** this is not intended to imply that _user code_ stops violating the Constraints simply by
moving it to a tool that doesn't check for them.)

EDITOR note this change is also present in/required by N3586.

### `[[strong]]`

This change should be introduced in a single place, in the clauses describing
attributes:

Modify 6.7.12.2, paragraph 2, to add the new attribute name:

> `deprecated`  
> `fallthrough`  
> `maybe_unused`  
> `nodiscard`  
> `noreturn`  
> `_Noreturn`  
> `unsequenced`  
> `reproducible`  
> **`strong`**  

(EDITORIAL: not in alphabetical order?)

Add a new section after 6.7.12.8:

> #### 6.7.12.9 The `strong` attribute
>
> **Constraints**
>
> The `strong` attribute shall be applied to the declaration of a typedef name.
>
> A value with a _strongly-typed_ type shall not be converted to a value, or assigned
> to an lvalue, that is not strongly typed at the same position within the type
> derivation, other than by means of an explicit cast or default argument promotion.
>
> **Semantics**
>
> The `strong` attribute marks a typedef name as _strongly-typed_.

> Strongly-typed typedefs behave as though a unique, virtual qualifier is added
> onto a type at the position within the type derivation that the typedef name was
> used. Unlike a concrete qualifier, the virtual qualification is not removed by
> value conversion, and may appear at any level of derivation.
>
> The resulting type is _strongly typed_ at the position where this virtual qualification
> would appear (i.e. at the position where the typedef name was used).
>
> This strong typing applies to the result of value conversion and to the result of
> any expression that uses a strongly-typed expression as a subexpression, except:
> 
> - strong types in the type of the left operand of a comma operator do not affect the
>   type of its result, only the type of the right operand;
> - strong types in the type of an integer value added to or subtracted from a pointer
>   do not affect the type of the result, only the type of the pointer operand;
> - strong types in the controlling operand and in unselected associations of a generic
>   selection do not affect the type of the result, only the type of the selected
>   association;
> - an explicit cast discards the strong typing of the operand expression and replaces
>   it with the strong typing associated with the type named by the cast (if any);
> - strong types are silently discarded by the default argument promotions.
>
> The composite type of a strongly-typed typedef name with another type inherits
> the strong typing associated with that typedef name. The composite type of two
> different strongly-typed typedef names inherits the strong typing associated with
> both names.
>
> Two distinct typedef names describe two distinct strong types, even if they
> have the same name. When the same typedef name is redefined in the same scope,
> its strong typing is the composite strong typing of the previous definition
> and the current definition.
>
> A typedef name that specifies a strongly-typed typedef name in its definition inherits
> the strong typing associated with the specified name, in addition to any strong typing
> it might introduce itself by also using the `strong` attribute.
>
> Strong types do not affect compatibility.<sup>footnote)</sup>
>
> <small>footnote) it is therefore not an error to declare a name with external linkage
> using a strong type in one translation unit, an unrelated strong type in another
> translation unit, and define it using the underlying type in a third translation unit.
>
> The `__has_c_attribute` conditional inclusion expression (6.10.2) shall return
> the value _XXXXXXL_ when given `strong` as the pp-tokens operand, if the
> implementation supports the attribute.
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
>     [[strong]] typedef int  Int1;
>                typedef Int1 Int2;
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
>
> EXAMPLE 6  Composite types and the type of composite operators can both be used to
> combine strong types, creating a value or type with the union of both original strongly
> typed attributes:
>
>     [[strong]] typedef int S1;
>     [[strong]] typedef int S2; // unrelated to each other
>     
>     extern int f (int);
>     extern S1  f (int);   // composite type of f is now S1(int)
>     extern int f (S2);    // composite type of f is now S1(S2)
>     
>     S1 s1 = ...;
>     S2 s2 = ...;
>     
>     s1 = s1 + s2;         // Diagnose an attempt to convert a value with S1 and S2 to S1
>     typeof (s1 + s2) s3 = s1 + s2;  // OK
>     
>     typedef typeof (s1 + s2) S4;    // Inherits the strong typing as previously
>     S4 s4 = s1 + s2;                // OK
>
> Usual arithmetic conversions might change the underlying type, making it possible to
> specify distinct underlying types with a common strong type, forming a "universe":
>
>     [[strong]] typedef long Steps;
>                typedef typeof ((Steps){0} + 0.0) PartSteps;
>     [[strong]] typedef typeof ((Steps){0} + 0.0) StrongPartSteps;
>     
>     Steps sl = ...;
>     PartSteps sd1 = sl;        // OK - converted to double, but still strongly Steps
>     StrongPartSteps sd2 = sl;  // OK - converted to a refinement of Steps
>     
>     sd2 = sd1;   // OK - normal assignment to a refined/stronger type
>     sd1 = sd2;   // Diagnose an attempt to convert to an underlying type
>     sl  = sd1;   // OK - converts back to long but within Steps
>     sl  = sd2;   // Diagnose an attempt to convert to an underlying strong type
>                  // (even though the double-to-long conversion is OK)

## Questions for WG14

Would WG14 like to add something along the lines of the `[[strong]]`
attribute into C2y?

Would WG14 like to make use of `[[strong]]` in the specification of
any library features?

## Acknowledgements

Thanks to Joseph Myers for detailed review comments!

## References

[C2y latest public draft][1]  
[Mars Climate Orbiter][2]  
[MISRA C 2023][3]  
[Definition of Standard ML (opaque signature matching)][4]  
[User-defined types in ML][5]  
[User-defined qualifiers][6]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3886.pdf
[2]: https://en.wikipedia.org/wiki/Mars_Climate_Orbiter
[3]: https://misra.org.uk/product/misra-c2023/
[4]: https://smlfamily.github.io/sml97-defn.pdf#page=114
[5]: https://courses.cs.washington.edu/courses/cse341/04wi/lectures/07-ml-user-types.html
[6]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3321.html
