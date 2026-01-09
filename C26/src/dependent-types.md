    Proposal for C2y
    WG14 3318

    Title:               Statically-dependent array types
    Author, affiliation: Alex Celeste, Perforce
    Date:                2024-08-19
    Proposal category:   Feature enhancement
    Target audience:     Compiler implementers

## Abstract

The C type system includes the notion of a _variably-modified type_, but does
not incorporate it as a first-class citizen. Variably-modified types cannot be
type-checked by the type system as described in C23, and are largely left up to
analysis tools and secondary passes to either verify with dataflow analysis or,
more commonly, simply ignore and trust that the code the user wrote was valid,
because variably-modified type errors are left in the realm of undefined
behaviour rather than constraints.

We propose a refinement of the type system that would allow a large class of
variably-modified types to be checked at compile-time as proper citizens of
C's type system. The range of valid conversions is reduced by this change,
which aims to restrict them to "intentionally correct" operations.

-------------------------------------------------------------------------------

# Statically-dependent array types

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3318
    Revises:      N/A
    Date:         2024-08-19

## Summary of Changes

### N3318
 - original proposal

## Introduction

The core of the issue is described succinctly in 6.7.7.3 "Array declarators",
paragraph 6:

> For two array types to be compatible, both shall have compatible element types,
> and if both size specifiers are present, and are integer constant expressions,
> then both size specifiers shall have the same constant value. If the two array
> types are used in a context which requires them to be compatible,
> **it is undefined behavior if the two size specifiers evaluate to unequal values**.

C is clear that the size of an array derivation doesn't have to be a constant
expression, in places where a non-constant is expressible. However, it doesn't do
anything with this information in the otherwise-static type system. The language
is slightly ambivalent about whether such array types even _are_ distinct types -
they cannot be matched by `_Generic` for obvious reasons (`_Generic` must select at
translation-time), but in other contexts the language is happy to just assume that
they are compatible with ...whatever the user wants them to be compatible with.
`sizeof` is able to produce some value, mostly as an artifact of this information
needing to exist "somewhere" for array indexing to function, but the value is not
bound in any traceable way to the type it describes.

Within prototypes the size is even explicitly discarded if the user has provided it
(paragraph 5). Compatibility is left _entirely_ within the realm of runtime UB.

Effectively, in C99 C added the _concept_ of [dependent types][3] to the type system,
without adding any definitions to the type system to enable them to be checked,
outside of simply leaving this entire responsibility to UB-sanitization. Dependent
types at their most general are not necessarily decidable, so this is arguably the
only way to incorporate them into the language without restricting the values in
some way. C99 VLAs are a "bare minimum" solution that makes the feature work, but
only if it is already being used correctly and only for a specific use case.

New features proposed for C2y, such as [N3212 Polymorphic types][6], exacerbate the
problem by adding new kinds of dependent types; the `_Var` derivation and the `_Type`
specifier proposed by N3212 are equivalent to a variably-modified array type and a
non-constant expression respectively, creating another way to potentially introduce
inconsistencies as there is no way to enforce that two `_Var(T)` are the same.
Fixing the definition of array types will enable new features like N3212 to be built
on a more solid type system foundation and defined in "as-if" terms that will inherit
a stronger typing model.

Other features like the forward declarations proposed in [N3207][7] remain toothless
as long as the type system simply discards the size information rather than attempt
to check it for consistency. The ability to express a wider range of dependent types
in interfaces would be significantly more useful if the language also required those
types to mean anything.

Unfortunately full dependent type checking is not decidable and any approach that
relies on the advanced folding capabilities of optimizing compilers is not adequate
as a reliable solution; and relying on UB sanitization to catch egregious mistakes
after they happen is not an acceptable approach as it fails to guarantee _any_ class
of mistake is not made and compiled.

We believe that adding some restrictions on the expressions that can be used to
derive an array type can have a significant effect on analyzability without necessarily
becoming onerous for correctly-written code. For instance:

    void foo (size_t len, int (*buf)[len]);
    
    void use (size_t x, size_t y) {
        int array[y];
        foo (x, &array);
    }

Whether this program is correct or not depends entirely on the runtime values of `x` and
`y` and cannot be decided in general. At the moment, C provides no means at all to say
that there is _any connection_ between an array type and an expression: our proposal
is to provide a means of connecting array derivations and a subset runtime expressions
that are known to be "the same", so that the above program can be statically rejected
and its equivalent (passing `y` as the first argument to `foo`) can be statically
accepted, without relying on undecidable flow analyses or checks that do not integrate
into the core language's understanding of types and constants.

It isn't dependent types, but it is a "[paper money][3]" promise of something with a
similar effect to dependent types.

## Proposal

We propose achieving this by changing the definition, but _not the user-facing syntax_,
of the array type derivation, to add a secondary expression part to the derivation, the
"expression tag". The tag is:

- implicit (never represented in the language syntax);

- always derived from the array length expression;

- always a constant expression, regardless of the value of the length expression;

- not always the same for the same length expression value.

This allows array type derivations to always be treated as distinct when they are not
statically known to be the same.

To accompany this, we need a way to unify the types of two arrays that are only determined
to be the same at runtime (including cases where the program logic allows them to either
be compatible or incompatible). We therefore introduce a way for array tags to be attached
to other expressions by a restricted number of syntax constructs, enabling code like:

    void foo (size_t s1, size_t s2, int (*l)[s1], int (*r)[s2]) {
        if (s1 == s2) {
            l = r;  // correct within the extent of the `if`
        }
        
        l = r; // rejected by type checking
    }

This corresponds to a very restricted and limited form of [typestate analysis][8].

### Array expression tags

An array derivation becomes a three-part rather than two-part derived type. In
"virtual syntax", the derivation might look like:

    element [size ; tag]

However this syntax is not exposed - there is no use case for the user being able
to control the `tag` (if they could, the problem would reset).

The first part, the element type, is not changed.

The second and third parts, the size and the tag, are both derived from the length
expression. For two array types to be compatible, they must have the same size
_and_ the same tag components.

For an array with a constant length expression, the derivation is almost exactly
the same as it is in C23:

    T[ice]  ->  T[ice ; *]

The size component is the constant length and the tag is a nil value. Therefore,
arrays with constant extents function identically to their definition in C23 and
the tag can be ignored when only manipulating these types.

For an array with a variable length expression, the derivation is almost the reverse:

    T[var]  ->  T[* ; $(var)]

where the size is a nil value and the tag is some constant produced by a syntactic
hash over the length expression. This type is no longer compatible _by default_ with
any constant-sized array type. The use of a syntactic hash also means that the type
is not compatible with any other variably-modified type, even if the _value_ of the
size is the same - arrays will not be assumed to be compatible and instead need to
have an equivalent length expression.

The starting point for compatibility therefore becomes that variably-modified arrays
are _non_-compatible by default, which is a stronger position: runtime values that
happen to produce the same result without clearly being related expressions, are no
longer considered compatible.

Arrays may also acquire the same tag through secondary means, explored below,
making it possible for array types with equivalent runtime values to be identified
as compatible dynamically.

### Syntactic length hash

Variably-modified array types are derived based on the _form_ of the expression.

The syntactic hash is produced according to a rule that should allow some flexibility
in permitting slightly different expressions, while also ruling out expressions that
are not definitely the same in a way that is simple for all implementers and users to
understand (simplicity is more important than generality overall). The rule is:

- an expression is first rewritten to expand all constant subexpressions to their
  abstract value
  (e.g. `v + (1 + 2)` unconditionally becomes `v + 3`;
  `3U`, `3L`, `3.0f` are all treated as the same value)

- statically-unused subexpression branches are discarded (e.g. the inapplicable
  branches of a `_Generic` selection, or the other branch of a constant `?:`)

- each variable identifier is converted to a unique identifier referring to the
  scoped variable entity (i.e. two expressions referring to variables with the same
  name in different scopes, do not produce compatible tags), by some implementation
  internal means

- each expression containing a (non-eliminated) modification operator (`++` etc.)
  unconditionally produces a unique tag

- each expression containing a (non-eliminated) reference to a global variable that
  is not `const` (or is `volatile`) unconditionally produces a unique tag

- each expression containing an object pointer dereference, or a call to a function
  not declared `[[unsequenced]]`, unconditionally produces a unique tag

These last two conditions mean that to produce two variably-modified array types
that depend on non-local state and are compatible, a local temporary should be used.

Additionally, the unique identifier for any variable entity with block scope is
incremented by some implementation-internal means after every expression that
modifies the named object (i.e. `=`, `++`, etc. as well as taking the non-constant
address). Sequencing of subexpressions is implementation-defined. Therefore,

    int x = ...
    int arr1[x];  // int[*; $(x@0)]
    ++ x;         // x internal id bump
    int arr2[x];  // int[*; $(x@1)]
    -- x;         // x internal id bump
    int arr3[x];  // int[*; $(x@2)] (NOT compatible with first type)

This does not depend on whether the intervening modification was evaluated.

If a variable modification is jumped over by a backwards jump, the location
of the jump _target_ also acts as an identifier increment point for the affected
variable(s):

    int x = ...
    int arr4[x];           // int[*; $(x@0)]
    do {                   // jump target, bump x id
        int arr5[x];       // NOT compatible, int[*; $(x@1)]
    } while (++x < K)      // bump x id as well
    ;                      // jump from here
    
    int arr6[x];           // int[*; $(x@2)] - not compatible with arr4
    do {                   // no bump because no modification
        int arr7[x];       // int[*; $(x@2)] - compatible with arr6
    } while (not_using_x); // no bump
    
    
    for (int x = 0; x < K; ++ x) {
        int arr8[x]; // unrelated x, int[*; $(x@BLK1@0)]
    }

Finally, for any expression that has not reduced to a constant or been defined as
producing a completely unique tag, an implementation-defined hash is applied to
the form of the converted expression, treating commutative operations as the same
and ignoring redundant parenthesization, casts, or identity expressions:

    int x = ...
    
    int arr7[x + 2];    // all of these arrays have compatible type:
    int arr8[2L + x];        // commutative
    int arr9[(long)x + (2)]; // value preserving cast, equivalent constant
    int arrA[(x + 0U) + 2];  // identity op, redundant parentheses

In practice, this should mean the majority of "correct" uses of variably-modified
types should still work as expected. Array types mostly become incompatible when
they rely on expressions that are not locally equivalent.

(We could demand that expressions are also normalized for logical equivalence,
(e.g. `!(a >= b)` to `a < b`) but this over-complicates the ruleset and doesn't
really capture the idea of consistency in user intent.)

### Value expression tag overrides

However, a large number of uses of variably-modified types _do_ depend on either
non-local values or expressions that are not trivially the same:

    extern size_t outsize;
    
    void f (size_t x, size_t y) {
        int arr1[outsize]; // these three types are
        int arr2[x];       // unconditionally incompatible ...
        int arr3[y];
        
        if (x == y) {
            // (completely valid work that needs arr2 and arr3 to be the same)
        }
        if (sizeof (arr1) == sizeof (arr2)) {
            // (completely valid work that needs arr1 and arr2 to be the same)
        }
    }

Therefore, we add two rules that enable _local_ compatibility between types that
are not generally compatible (which introduces a weak notion of [typestate][8]):

- the result of a non-constant `sizeof` expression produces a tag that is _always_
  compatible with the tag for the operand type. Therefore it is _always_ possible
  to create a variably-modified array type `A2` compatible with an existing array
  type `A1` by using `T[sizeof(A1)]`. The tag result of `sizeof` is just whatever
  the tag result of the operand type was (so for constant sizeof, the tag is zero).

- the `if` statement and `?:` expression each introduce a special scope for
  variably-modified types, when their controlling expression is a `==` expression.
  So long as the operands to the `==` operator are not defined to produce
  unconditionally unique tags, within the _guarded extent_ of the controlling
  expression, the left and right operand expressions are defined to produce the
  _same_ tag, including when used as a subexpression within a larger length
  expression.

Therefore:

    void f (size_t x, size_t y) {
        int a1[x];
        int a2[y];    // not compatible with a1, $(x) != $(y)
        
        if (x == y) {
            // $(x) is overridden by $(@if0)
            // $(y) is overridden by $(@if0)
            
            // types of a1 and a2 are compatible here
            typeof (a1) a3;  // type is int[*; $(@if0)]
            
            int a5[x + 1]; // compatible with a6:
            int a6[1 + x]; // types are both int[*; $(@if0 + 1)]
        }
    }

This property does not backpropagate, so checking `sizeof (L) == sizeof (R)`
_does not_ make arrays declared with the original length expression of `L`
compatible with the type of `R`; but checking the expressions directly (`L == R`)
would do so.

(i.e. in the example above the compiler is not required to link `x == y` to
`sizeof (a1) == sizeof (a2)` as these are not consistent expression spellings)

The _guarded extent_ of the `if` statement extends from the start of the
_secondary-block_ associated with the `if`, until either the end of the
_secondary-block_ or the first identifier bump affecting either operand expression
to the `==`. The _guarded extent_ of the `?:` operator is its second operand,
unless the second operand contains any identifier bumping operation affecting
either operand to the `==`.

    int g (size_t x, size_t y) {
        int a1[x];
        int a2[y];    // not compatible with a1, $(x) != $(y)
        
        if (x == y) {
            // $(x) is overridden by $(@if0)
            // $(y) is overridden by $(@if0)
            
            // types of a1 and a2 are compatible here
            
            x += 1; // end of guarded extent, @if0 falls out of scope
            
            // types of a1 and a2 are not compatible here
        }
        
        return x == y
             ? expr1   // types of a1 and a2 are compatible
             : expr2;  // types of a1 and a2 are not compatible
             
        return x == y
             ? (expr3, ++ x)  // types of a1 and a2 are not compatible in expr3
             : expr4;         // because there is NO guarded extent
    }

If one of the operands to the `==` is a constant expression, within the _guarded_
_extent_, the other operand is replaced by the constant value when evaluating
array types. If this results in the whole array length expression becoming an
integer constant expression, the resulting type is a VMT compatible with the
equivalent non-variably-modified array type:

    void fc (size_t x) {
        int a1[x];
        int a2[10]; // not compatible with a1, $(x) != *
        
        if (x == 10) {
            // $(x) is overridden by $(10)
            
            // types of a1 and a2 are compatible here
            typeof (a1) a3;  // type is a VMT compatible with int[10; *]
                             // (still flagged as VMT for syntax purposes)
        }
    }

Although the type is compatible with a static array type, keeping it flagged as
a VMT internally lowers the implementation burden and makes it consistently
impossible to use in places like generic selections.

### Tag inference

In order to use variably modified types as the arguments to functions, the type
in the caller needs to be translated into a type for the callee:

    void user (size_t sz, int (*p1)[sz], int (*p2)[sz]) {
        // types of p1 and p2 are compatible
    }
    
    void caller (size_t x, size_t y) {
        int a1[x];
        int a2[y];
        
        // we want this to be a type error:
        user (x, &a1, &a2);
        
        // and for these to be OK:
        user (x, &a1, &a1);
        user (y, &a2, &a2);
        
        // this is also OK
        if (x == y) {
            user (x, &a1, &a2);
        }
    }

A similar tag-override mechanism is used at the point of application, with an
_extent_ that applies only within the application of argument list, to the argument
subexpressions themselves:

    void user (size_t sz   // variable the types of p1 and p2 depend on
        , int (*p1)[sz]    // type becomes int(*)[*; @user0]
        , int (*p2)[sz]);  // type becomes int(*)[*; @user0]
    
    // @user0 is available within the function call argument list:
    user (x      // x is overridden to have the tag @user0
        , &a1    // the type int[*; x] is compatible with int[*; @user0]
        , &a2);  // the type int[*; y] is not compatible, error here

`sz` is identified independently of the caller context as being a dependency of the
types of `p1` and `p2`, so we can generate an override for the expression passed to
`sz` and then directly check that the types of the dependent arguments have tags
compatible with the tags of their corresponding parameters.

This check examines the declaration visible at the point of call and does not imply
a runtime check against the parameter declarations in the definition, if it exists
in a different TU.

We also propose that the composite type rules are altered so that function declarations
are checked for compatibility in the same way as for argument application, rather than
allowing variably-modified types to form a composite type that discards inconsistent
size information.

When the size in the callee is unspecified (`[*]`) or is an expression with an
unconditionally-unique tag according to the rules above, the type of the argument
cannot effectively be checked against the type of the parameter because the
information is not available. The only thing we can propose here is that use of
the unspecified-length syntax in parameter lists should always cause a warning
to be emitted, and that the use of expressions producing unconditionally-unique
tags for array length expressions in function parameter lists should be deprecated,
also causing a warning to be emitted. Of these two, the former feature is probably
more disruptive.

## Related work

Much more advanced work, including the ability to properly typecheck calls to
functions like `printf`, has [been explored by Kiselyov][4]; a number of
strategies that take advantage of the tools already provided in more powerful
typesystems than C's can be used to fake dependent types at a local level,
closely bound enough to what the user is trying to do that their non-general
nature doesn't matter.

This approach is significantly more primitive but aims to capture the same
most fundamental idea, that true dependent types aren't really needed for most
code that most users write. As a result, [a "fiat" approach][3] to type checking
can be practical for most programs and still provides a huge improvement over
the status quo of simply not checking variably-modified types at all.

Another approach is to use staged-programming (or staged-metaprogramming) in
order to take advantage of a richer type system in a host language to prove that
an output program in a lower-level language is correct. This is demonstrated by
[Low∗][5], which embeds a C subset into F∗. However, the resulting C programs
are stripped of the dependent type information after it has been checked by the
F∗ type system ([Clight does not export VMTs at all][10], instead converting them
to element pointers), so this is of limited help to users who want to write in C
directly. However, staged programming with C as a second stage is a generally
interesting avenue and the Committee should aim to support this as a first-class
application of the language, as offloading correctness checking onto richer
tools is always helpful.

A simpler proposal also being considered for C2Y would reuse the `[*]` syntax
in block-scoped declarations, where it would deduce an array size. We think
this feature would compose very well with the proposed typechecking rules and
the deduced array size could easily simply inherit the tag information from
the initializing expression, after that working exactly like a VMT declared
with existing syntax.

## Impact

The goal of this proposal is to produce a compromise position between full type
checking for runtime-dependent types (practically impossible given the nature of
the surrounding C language, and undecidable at best) that is at least decidable
and implementable. The change would have some impact on implementations, but less
than a more complete solution.

As always, a smaller implementation can choose to be more permissive with type
checking and leave most of this up to more advanced tools, so long as it admits
correct programs. Since no change is proposed except for _checking_, no new
functionality is actually introduced here over what is provided in C23.

The actual type compatibility checks within a compiler would not be affected by
this change, as it takes effect at the point of establishing a type for a
declared object or function, and of determining the type of an identifier used
in an expression. Once these types have been generated they would feed into an
existing implementation of type compatibility in the same way as any other
complete type.

Generating the expression types has a little complexity because the type of
any expression within the _extent_ of an override would need to regenerate
its syntactic-hash to produce a new tag that took overrides into account.
At the implementation level this means that the same expression might have
distinct type identities depending on context, but this fact wouldn't be
observable to the end user.
This is not overwhelming to implement, however.

Arguably the ability to check compatibility of variably-modified types would
introduce the ability to list such types as generic associations, but we do not
propose adding that feature specifically because it would mandate stronger
implementation support. One possible option that may be useful is for the
Standard to allow smaller implementations to define that they will not check
dependent types at all, in which case they should be required to warn against
all assigning-uses of such types when in conforming mode. This may be
practical for implementations where the core underlying feature of variable
length array objects is not supported, especially if dynamic memory is also
in short supply and actual VMTs are unlikely to see much use.

## Future directions

As above, some changes required for this to complete coverage over array types
where the information is outright missing would generate warnings on their
use. In practice this would work to deprecate such types.

As a result, in future we propose that unspecified array sizes in function
prototypes should be made obsolescent; and that variably-modified types in
function parameters that make use of global or indirect references, assignment,
or calls to non-`[[unsequenced]]`` functions in their length expression,
should also be made obsolescent (and deprecated if a distinction exists).

## Proposed wording

This proposal is an early draft that is still likely to significantly change
before potential adoption. Therefore wording here is at most illustrative.
We **do not bother** with all necessary changes at this time as the proposal
would result in a significant number of knock-on edits that are better left
until rough consensus on the semantics is achieved.

The proposed changes are based on the latest public draft of C2y, which is
[N3220][1]. Bolded text is new text when inlined into an existing sentence.

Add a new subclause in 6.2.5 "Types", after all current paragraphs:

> ### 6.2.5.1 Variably modified types
> 
> When the number of elements specified for an array type is a constant
> expression, it is a fixed-length array type. Every specifier that derives
> an array type with the same element type and the same constant number of
> elements describes the same type.
> 
> When the number of elements specified for an array type is not a constant
> expression, the array type is a _variable length array_ type. A variable
> length array is a _variably modified type_, and any type derived from a
> _variably modified type_ is also a _variably modified type_.
> 
> A specifier for a variable length array type names a specific type that
> does not directly depend on the concrete value of the length expression.
> Instead, a key is generated from the syntactic structure of the length
> expression<sup>footnote1)</sup>, and two array specifiers that result
> in the same key describe the same type:
> 
> - First, any constant subexpressions are evaluated and replaced by their
>   value, and any generic selections are replaced by their matching
>   association expression.
> 
> - Then, any additive or multiplicative expressions that perform an identity
>   operation (adding or subtracting zero, multiplying or dividing by one)
>   are replaced by their non-constant operand.
> 
> - Then, any statically-unevaluated subexpressions are replaced by a constant
>   value equivalent to a default-initialized value of the same type.
> 
> If the remaining expression contains any of:
>   
> - an increment, decrement, or assignment operator;
> 
> - a reference to a volatile object, or non-constant object with non-automatic
>   storage duration, except as part of an address constant;
> 
> - a dereference of a pointer to object;
> 
> - a call to a function that was not declared with the `[[unsequenced]]`
>   attribute;
> 
> a new unique key is generated for this array specifier<sup>footnote2)</sup>.
> 
> An expression not containing such operands is a _stable key expression_.
> 
> Otherwise, each identifier within the remaining expression is converted to
> a unique representation distinguishing it from any other identifiers
> designating objects in other scopes. Additionally, this unique representation
> is updated after<sup>footnote3</sup> every occurrence of a _modifying-operator_
> applied to the identifier in the source, regardless of whether it is evaluated;
> and after any occurrence of a backward jump in the source that jumps over one or
> more occrrences of a _modifying operator_ applied to the designated object.
> Each such update shall produce a new unique representation.
> 
> A _modifying operator_ is any of: an increment or decrement operator; an
> assignment operator; or the unary `&` operator, unless its operand has
> `const`-qualified type or the resulting pointer is immediately converted to a
> pointer to `const`-qualified type.
> 
> An implementation-defined hash function is applied to the syntactic
> form of the resulting expression, as-if by proceeding over the remaining tokens
> (with discarded subexpressions replaced by generated tokens for their generated
> values) from left to right; except that redundant parentheses (that do not
> affect order of operations) and casts that do not alter their operand's
> arithmetic value are not considered, and commutative operators do not depend
> on order of operands; and only the arithmetic value and not the spelling of
> constant subexpressions is considered.
> 
> An occurrence of the `sizeof` operator with a variable length array type used
> as the length expression for an array specifier will always cause the resulting
> specifier to designate the same type as the operand of `sizeof`, even if it
> previously specified a unique type.
> 
> An `if` statement whose _expression_ is an _equality-expression_ using the `==`
> operator to compare two _stable key expressions_, introduces a _guarded extent_,
> which begins at the start of the `if` statement's first associated secondary block
> and terminates at either the end of the secondary block or at the first occurrence
> of any _modifying operator_ that would affect a key generated for either
> _stable key_ operand to the `==` operator. Within the _guarded extent_, both
> _stable key_ operands are shall produce the same key (or key part) when they
> appear within the length expression of an array specifier. This is a _key override_.
> 
> **NOTE** this implies that an implementation may need to re-evaluate the key
> for an array type specified outside a _guarded extent_ if the type is used within
> that extent.
> 
> A _conditional expression_ whose first operand is an _equality-expression_ using
> the `==` operator to compare two _stable key expressions_, introduces a
> _guarded extent_ which covers the second operand, so long as there are no
> occurrences of any _modifying operator_ within the second operand.
> 
> Within the arguments of a call to a function with variably modified parameter types,
> a _key override_ also occurs if the variably modified parameter types depend on
> the value of another parameter. In a subexpression that calls such a function,
> the _stable key_ associated with the parameters with non-variably-modified type
> replaces any _stable key_ that would be generated for the argument expression
> applied to that parameter, during the evaluation of the types for any subsequent
> argument expressions.
> 
> **Recommended Practice:** in the event that a parameter has a variably modified
> type that does not depend on the value of another parameter, an implementation
> is encouraged to emit a diagnostic that the argument applied cannot be type-checked.
> 
> EXAMPLE 1  The following declarations all specify the same type:
> 
>     int x = ... // block scope
>     
>     int arr7[x + 2];    
>     int arr8[2L + x];        // commutative, spelling of literal
>     int arr9[(long)x + (2)]; // value preserving cast, equivalent constant
>     int arrA[(x + 0U) + 2];  // identity op, redundant parentheses
> 
> Example 2  Modification of an identifier means it will produce a different key,
> and therefore the containing specifier will designate a different type:
> 
>     int x = ... // block scope
>     
>     int arr1[x];  // UID of x, state 0
>     ++ x;         // modifying operation
>     int arr2[x];  // UID of x, state 1
>     -- x;         // modifying operation
>     int arr3[x];  // UID of x, state 2
> 
> Each of the declared arrays has a distinct type.
> 
> EXAMPLE 3  If a _modifying operator_ is jumped over by a backwards jump,
> the location of the jump target also acts to update the unique representation
> of the identifier that was the operand of the jumped-over operator:
> 
>     int x = ... // block scope
>     
>     int arr4[x];           // UID of x, state 0
>     do {                   // jump target, updates unique representation
>         int arr5[x];       // UID of x, state 1
>     } while (++x < K)      // ++ updates unique representation too
>     ;                      // jump from here
>     
>     int arr6[x];           // UID of x, state 2
>     do {                   // no update because no modification
>         int arr7[x];       // UID of x, state 2 - same type as arr6
>     } while (not_using_x);
> 
> EXAMPLE 4  The `sizeof` guarantees the resulting type specified will be the
> same as its operand, if it is used as the length expression:
> 
>     extern int f (void);
>     
>     int arr8[f ()]; // always specifies a unique type
>     
>     int arr9[sizeof arr8]; // specifies the same type as arr8, whatever it is
> 
> EXAMPLE 5  A guarded extent establishes a scope where two incompatible array
> types can temporarily become compatible after meeting a runtime condition:
> 
>     void f (size_t x, size_t y) {
>         int a1[x];
>         int a2[y];    // not compatible with a1, $UID(x) != $UID(y)
>         
>         if (x == y) {
>             // $UID(x) is overridden by $UID(@if0)
>             // $UID(y) is overridden by $UID(@if0)
>             
>             // types of a1 and a2 are compatible here
>             typeof (a1) * p3 = &a1;  // type uses $UID(@if0)
>             p3 = &a2;  // OK - type of a2 also uses $UID(@if0) within
>                        // the extent, not $(y) within the extent
>             
>             x += 1; // end of guarded extent, @if0 falls out of scope
>         }
>     }
> 
> EXAMPLE 6  Within a call, the subexpression for an argument to a parameter that
> other parameters are _dependent_ upon, has its stable key locally overridden
> with an extent limited to the argument applications:
> 
>     void user (size_t sz, int (*p1)[sz], int (*p2)[sz]) {
>         // types of p1 and p2 are compatible
>     }
>     
>     void caller (size_t x, size_t y) {
>         int a1[x];
>         int a2[y];
>         
>         // error: a1 and a2 have unrelated types
>         user (x, &a1, &a2);
>         
>         // OK
>         user (x, &a1, &a1);
>         user (y, &a2, &a2);
>         
>         // this is OK because of the guarded extent
>         if (x == y) {
>             user (x, &a1, &a2);
>         }
>         
>         // within the argument application subexpressions only:
>         user (x     // stable key subexpression x defined to produce the sz key
>             , &a1   // therefore this has a type dependent on $(sz)
>             , &a1); // and these variably modified arguments type-check
>     }
> 
> Like example 5, the implementation may need to temporarily change the type of
> an expression within an extent; however, this is never observable from the code.
> 
> EXAMPLE 7  Implementations are encouraged to emit a diagnostic when an argument
> is applied to a parameter with a variably modified type that cannot be checked:
> 
>     void user (size_t, int (*)[*]); // variably modified type that doesn't
>                                     // seem to depend on anything specific
>     
>     void caller (size_t x) {
>         int arr[x];
>         user (x, &arr);  // diagnostic here: the type of the second parameter
>     }                    // cannot be checked, don't know what is expected
> 
> <small>**footnote1)** this key is internal to the implementation and never
> exposed into the program source. The corresponding "hash function" is also
> not observable or directly invocable.</small>
> 
> <small>**footnote2)** therefore, every array specifier whose expression contains
> one of these features in an evaluated subexpression designates a distinct array
> type, regardless of whether the lengths evaluate to the same value.</small>
> 
> <small>**footnote3)** the order of evaluations within an expression is
> implementation-defined.</small>
> 
> **Forward references:** compatible type, generic selection, multiplicative
> operators, additive operators, constant expressions, array declarators, 
> initialization

### Additional work

In 6.2.7 "Compatible type and composite type", additional work is still
needed to clarify what "the same" means for types, and this section should
probably be reworked before changing the relevant composite type rules, in
terms of derivations producing specific types.

The undefined behaviour associated with composite types for unevaluated
variably modified types is a problem for the key evaluation rule and needs
to be addressed.

Additional work is needed in 6.6 "Constant expressions" to clarify when a
subexpression is statically unevaluated, which is not currently a term of art.

Additional work is needed in 6.7.7.3 "Array declarators", which is currently
responsible for defining _variable length array_ and _variably modified type_
in terms of declarators. We think the core of these definitions should be
extracted to the new clause given above, or at least a subclause in 6.2.5, and
instead the definition of declarators should refer to what type derivations
they describe.

Adopting [consistent left-to-right evaluation][11] within expressions would
significantly simplify the definition of _guarded extent_ and what "after" means
for a _modifying operator_.

## Questions for WG14

Would WG14 like to add a mechanism for statically checking dependent (variably
modified) types to C2y, along the lines of the proposal in N3318?

Would WG14 like to allow implementations to define that they either check
dependent types, _or_ warn against all assigning-uses of dependent types?

## References

[C2y latest public draft][1]  
[Dependent type][2]  
[Lightweight guarantees and dependent types][3]  
[Genuine dependent types and faking them][4]  
[Verified Low-Level Programming Embedded in F∗][5]  
[N3212 Polymorphic types][6]  
[N3207 Forward Declaration of Parameters v3][7]  
[Typestate analysis][8]  
[Unspecified Sizes in Definitions of Arrays][9]  
[Mechanized semantics for the Clight subset of the C language][10]  
[Strict order of expression evaluation][11]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf
[2]: https://en.wikipedia.org/wiki/Dependent_type
[3]: https://okmij.org/ftp/Computation/lightweight-static-guarantees.html#deptype
[4]: https://okmij.org/ftp/typed-formatting/index.html#dep-type
[5]: https://dl.acm.org/doi/pdf/10.1145/3110261
[6]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3212.pdf
[7]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3207.pdf
[8]: https://en.wikipedia.org/wiki/Typestate_analysis
[9]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3295.pdf
[10]: https://xavierleroy.org/publi/Clight.pdf
[11]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3203.htm
