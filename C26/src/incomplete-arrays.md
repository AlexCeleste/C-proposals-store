    Proposal for C2y
    WG14 3777

    Title:               Allow arrays of incomplete element type
    Author, affiliation: Alex Celeste, Perforce
    Date:                2026-...
    Proposal category:   Undefined behaviour removal, Constraint refinement, Demon slaying
    Target audience:     Compiler implementers

## Abstract

The Constraint against specifying an array type with an element that is not a complete
object type is overly strict.

This proposal argues that removing the Constraint entirely
and replacing it with words defining that such arrays are themselves incomplete types,
results in the same level of allowed behaviour, by deferring all of the restrictions against
the uses of such types in value expressions to the definition of completeness.

-------------------------------------------------------------------------------

# AT ITS CENTRE, A HOLLOWNESS

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  Nxxxx
    Revises:      N3777
    Date:         2026-...

## Summary of Changes

### Nxxxx
 - wording for flexible array members to use "incomplete size" instead of general incompleteness
 - wording for array initialization to use "incomplete size" instead of general incompleteness
   and specify that the size, not the type in general, is completed at the end
 - include `alignof` as well as `_Countof`
 - do not imply functions are objects or have a notion of completeness by themselves
 - comments to the impact section (extern, parameters)

### N3777
 - original proposal

## Introduction

### The language restriction

Section 6.7.7.3 "Array declarators" specifies among the Constraints that when declaring an
identifier with array type

> The element type shall not be an incomplete or function type.

In practice this is a Constraint against specifying any array type with an incomplete or
function element type, because this section defines the syntax used to specify array types
generally. The Constraint was added in C99 but is not described by the C99 Rationale.

The Constraint against the declaration is supported by a UB against the type itself, in
Section 6.2.5 "Types" specifies in the definition of array types that

> The element type shall be complete whenever the array type is specified.

This creates an opening for undefined behaviour when an array type is "somehow" specified
that uses `void` or another incomplete type as its element type. The undefined behaviour
itself is a potential language extension point, almost impossible to access without also
encountering the Constraint, and is not generally a runtime risk, making it a demon-class UB
(it is not a ghost-class UB if the compiler provides builtin means of specifying an array
type without using declarator syntax).

The undefined behaviour is not listed explicitly in Annex J and was made explicit for
C11. Previously, the same UB existed in C99 and C90 but was fully implicit (with a footnote
that simply claimed "an array of incomplete type cannot be constructed", because incomplete
types were not considered object types prior to C11).

### The problem in practice

The Constraint has the effect of preventing code with apparently-reasonable meanings from
being accepted, and can appear arbitrarily-contradictory. For instance, it is possible to
declare an identifier with external linkage if the identifier has an incomplete, _non_-array
object type; but not if the identifier has array type. The meaning of the declarations is
not substantially different since the non-array identifier can already only be used in very
limited ways (i.e. it has an address, but unknown footprint):

    struct Incomplete;
    
    extern struct Incomplete EI;        // OK even though we can't do much with it
    
    extern struct Incomplete EA[];      // Not OK, for some reason
    extern struct Incomplete EA10[10];  // Not OK
    
    extern void EV[];                   // Not OK even though we can do exactly as much
    
    void f0 (void * p) {
      p = &EI;                          // all of these operations are effectively the same
      p = &EA;
      p = &EA10;
      p = &EV;
    }

Declaring an external identifier with "array of `void`" type is a common extension that
indicates it is intended to provide some amount of byte storage. If the compiler allows
the code to translate, taking its address produces a value of `void *` that can potentially
be used like allocated memory.

Related to this, there is a _completely_ artificial Constraint imposed against function
parameters that do not even have array type, if they are specified using array _syntax_,
because the order in which the compiler interprets the syntax requires it to first encounter
and reject the array _declarator_ that would, if it was accepted, otherwise be adjusted to
a non-array type:

    void f1 (struct Incomplete * pi) { }   // OK
    void f2 (struct Incomplete pi[1]) { }  // Not OK despite same apparent meaning
    
    void f3 (void * pv) { }   // OK
    void f4 (void pv[1]) { }  // Not OK despite same apparent meaning

The language also contains a pre-emptive Constraint against specifying arrays of incomplete
array types or of function types. There are potential use cases for such types under discussion
(especially for arrays, potentially in the extension of `_Generic` by [N3348][4], [N3441][5], [N3428][6])
and in any case, also no mechanism in the language to "misuse" such types, so the Constraint
is probably overly-aggressive here too.

## Proposed improvement

The Constraint should be removed from 6.7.7.3 "Array declarators" and the corresponding
_shall_-clause should be removed from 6.2.5 "Types".

To replace this, it should be made explicit a few lines down in 6.2.5 that an array derivation
of an incomplete or non-object element type is itself an incomplete type.

This should be the only normative wording required - everything else should be covered by the
general restrictions that already exist on the use of any expression with an incomplete type.
For instance, `sizeof` already requires its operand to have or be a complete object type.

The only missing space should be the definition of the subscript operator, and only because the
operator is no longer (as of C2Y) defined in terms of an intervening decay-to-pointer - in all
prior versions of C this would also have been an implicit change.

6.7.1 in combination with 6.7.7.4 contains an apparent exception to allow function parameters
to only be complete _after_ adjustment, and therefore does not appear to require modification by
itself; however, this exception does not appear to be usable by parameters declared with
syntactic array type because it directly conflicts with the Constraint against using the declarator
syntax for them. This may warrant non-normative clarification.

The `_Countof` operator does not realistically need to consider the element type and could
also be refined to only require the element count to be provided, enabling additional uses.
This might have useful applications if the extension to combine `_Countof` with array-parameter
syntax is adopted later.

We do **not** propose adding any definition or meaning to "`void` pointer arithmetic" as part of
this change.

## Impact

All existing conforming code remains valid.

Existing code making use of language extensions to allow linking to externally-defined objects
of unusual types (usually, `void[]`) become conforming.

Type derivations that make sense in the type system, such as "array of `void`" or "array of array
of unknown size", become possible to specify. The type system is not concerned with their use as
objects and does not depend on completeness in order to distinguish and match types, so this
simplifies the restrictions and allows more generality in what derivations can be applied; this
may allow more library-based traits to be defined using `_Generic` and similar interesting things.
All existing tools should be able to implement this at the type level simply by removing/silencing
the existing error raised by the Constraint violation.

Types such as `int[2][]` become available for use by additional refinements to `_Generic` in the
realm of array matching. This may be an attractive replacement for the currently-adopted star
syntax, or complement it with a new meaning. During adoption of [N3348][4] it was observed that
the existing Constraint was an unfortunate obstacle.

Array-of-`void` types become available for use as the type of function parameters. This can be used
in conjunction with `static` to make clear that a `void *` argument should not be null or that the
size of the referenced storage should depend on another parameter. This might also be clearer than
use of `unsigned char[]` for this purpose for some function signatures.

The meaning of the size of a `void` array does not need to be defined in general. It can be defined
by the specification of each individual library function that uses such syntax; the important thing
is just that `void[1]`, `void[2]`, `void[n]` etc. are distinct types that the type system instantly
becomes able to distinguish, understand, and use existing dependency-tracking or dataflow mechanisms
to enforce bounds checks against.

Array-of-function types are unusable as objects and therefore harmless. There may be potential uses
in type traits, since `_Generic` would support matching them, or as a starting point for further
development not proposed here. Allowing them to be specified makes the language more consistent.

With this change it becomes possible to _declare_ an array with incomplete element type and external
linkage, and then complete the element type with a flexible array member. It is not actually possible
to _define_ this array object (in any TU), so the effect is essentially similar to using dynamic
storage to allocate a buffer of such objects - currently no Constraint stops the user trying to
index such a buffer, which may be worth addressing in a separate change. Tools should warn about
completed types that would violate previous implied Constraints as a matter of QoI.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3685][0]. Bolded text is new text when inlined into an existing sentence.

Modify 6.2.5 "Types":

Modify the first sentence of paragraph 25; and delete the original second sentence with the
_shall_-clause that required an array type to have a complete object type as its element type:

> An array type describes an **indexable set of members with a particular type**, called the
> _element type_. **Member objects of an object of array type are allocated contiguously.**
> Array types are characterized by their element type and by the number of elements in the
> array. An array type is said to be ...

Modify the first two sentences of paragraph 28:

> An array type of unknown size **or with an element type that is not a complete object type** is an
> incomplete type. It **becomes complete when both the size is completed and the element type is**
> **a complete object type**. **The size is completed** for an identifier of that type by specifying
> the size in a later declaration (with internal or external linkage).

**NOTE** is "completed" the right word for the size? Would "provided" be better?

(EDITORIALLY) Suggest that the remainder of paragraph 28, beginning "A structure or union type", is
split to a new paragraph.

(It does not appear necessary to over-complicate the wording by specifying when the element type becomes
complete, as part of the specification of the array type - this is handled elsewhere/recursively.)

Modify 6.5.3.2 "Array subscripting", making the requirement for arrays and pointers symmetrical:

> One of the operands shall have type "pointer to complete object _type_" or "array of **complete object**
> _type_", the other operand, called the subscript, shall have ...

Modify 6.5.4.5 "The `sizeof`, `_Countof`, and `alignof` operators", second sentence of paragraph 1,
to partially relax the completeness requirement for the operand of `_Countof` or `alignof`:

> The `_Countof` operator shall only be applied to an expression that has **an array type with a**
> **completed size**, or to the parenthesized name of such a type. The `alignof` operator shall be
> applied to a complete object type or an array **type. If applied to an array type, removing all**
> **array derivations from the type shall yield a complete object type.**

(**NOTE** see above w.r.t "completed")

Modify 6.7.3.2, "Structure and union specifiers", paragraph 3:

> ..., except that the last member of a structure with more than one named member can have
> **an array type with complete object element type but incomplete size**; ...

Modify 6.7.7.3 "Array declarators", deleting the sentence providing the Constraint on the element
type from paragraph 1:

> ... it shall have a value greater than zero. The optional type qualifiers and ...

Modify 6.7.11 "Initialization", first sentence of paragraph 4:

> The type of the entity to be initialized shall be an array of **complete object element type and**
> unknown size**,** or a complete object type.

Modify 6.7.11, paragraph 26:

> If an array of unknown size is initialized, its size is determined by the largest indexed element with
> an explicit initializer. The **size of the** array type is completed at the end of its initializer list.

## Questions for WG14

Does WG14 want to accept the proposed change to relax the rule against specifying an array type with
an element type that is not a complete object type?

## Acknowledgements

Thanks to Jay Ghiron and Joseph Myers for detailed review comments!

## References

[C2y public draft N3685][0]  
[C11 public draft][1]  
[C99 public draft][2]  
[C90 public draft][3]  
[N3348 Matching of Multi-Dimensional Arrays in Generic Selection Expressions][4]  
[N3441 `_Generic` and VLA Realignment and Improvement][5]  
[N3428 Comments on Array Sizes][6]  

[0]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3685.pdf
[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf
[2]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf
[3]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n423.pdf
[4]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3348.pdf
[5]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3441.htm
[6]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3428.pdf
