    Proposal for C2y
    WG14 3586

    Title:               The `void`-_which-binds_, v3: typesafe parametric polymorphism
    Author, affiliation: Alex Celeste, Perforce
    Date:                2026-06-01
    Proposal category:   New feature
    Target audience:     Compiler/tooling developers, library developers, application developers

## Abstract

C has a built-in mechanism for handling generically-typed data: `void *`. Unfortunately, this has
a problem: it achieves genericity by simply discarding all type information!

We propose a mechanism to define native functions and data structures that are _both_ polymorphic
_and_ strongly-typed, as commonly seen in functional programming languages or those that have
adopted functional idioms.

The mechanism does not impose any runtime or ABI burden.

-------------------------------------------------------------------------------

# The `void`-_which-binds_, v3: typesafe parametric polymorphism

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3586
    Revises:      N3316
    Date:         2026-06-01

## Summary of Changes

### N3586
 - remove everything related to rejected `qual_var` and `qual_with` features
 - remove `bind_var` from return, overcomplicates specification and not used
 - change "the same" to "compatible" for types, as compatibility is the property needed
 - add example of collaborating with `_Type`
 - address review comments and add clarifying examples
 - rebase proposed changes against the latest draft

### N3316
 - add discussion of function pointer usability issues, proposal for conversion
 - add qualifier-generic binding in response to Committee request
 - add discussion of container types
 - small fixes, typo changes, completely reworked normative wording
 - rebase proposed changes against latest draft

### N2853
 - original proposal

## Introduction

C can define polymorphic functions by using `void *`, but this comes at the cost of complete type
erasure.

This means that while a function can be defined to operate on any type of data, it trades the
ability to do so for the lost ability to statically check that the operations it wants to perform
on that data are actually valid for its concrete type. It cannot guarantee that two operands to
be compared originate from the same kind of data, and it cannot even guarantee that the comparison
operator intends to work with them if they are!

In many other languages, expressing the necessary constraints is easy. In many languages influenced
by ML, "type variables" can be used in the function signature, and in C++, templates can be used
to parameterize a function over types, although templates are not parametric in quite the same way
(templates are not polymorphic functions, but instead instantiate _to_ monomorphic functions).

We propose a matched pair of two attributes which can appertain to a pointed-to `void`, which force
the pointer to either bind to a concrete type (for operator implementations and container type
definitions), or to bind _consistently_ with other `void` pointers within the same toplevel
function signature. Because these are attributes, they impose no runtime cost, and if they are
removed from a correct program, ensure it will remain binary-identical to the type-safe version.
This has the convenient impact that implementation conformance can be achieved by doing nothing.

### Proposal history

This proposal was previously discussed in Fall 2024, where the Committee indicated:

 - strong enthusiasm for "something along these lines", but concern about the exact details
 - a preference for constraints introduced by attributes to be expressed as real Constraints,
   not as Recommended Practice
 - strong aversion to pursuing qualifier-generic notation at this time
 - weak dislike for implicit "shim function" generation

Discussion did not proceed as far as generic data structures or changes to the Standard Library.
Generic data structures remain a future direction for this change, while qualifier-generic
functions (generalized using attributes) have been removed and will not be revisited in the near
future.

Function pointer casts are separated into their own question for WG14 as this was not fully
addressed.

## Rationale

Other languages, especially those descending from or influenced by ML, have the notion of "type
variables", by which function (and other) types can be parameterized: at the point of use, an
operand type is substituted into the type of the function, and if the same type variable appears in
two places in the function type, it has to have the same binding to a type deduced from the
operands, enforcing type safety.

So in an ML-like or Haskell-like language, a function can have a type like this:

    map :: forall a. ([a], (a -> a)) -> [a]

...which we can understand to take some sequence of elements of type `a`, a callback which
transforms objects of type `a` to other objects also of type `a`, and returns another sequence
which (we will assume) contains the results of each transformation. We can't call this function
with a callback that operates on objects of a type not compatible with the objects in the input
buffer.

Informatively, a similar C++ signature might be:

    template <typename A>
    auto map (Seq <A> const &, Function <A (A)> const &) -> Seq <A>;

Though it is important to remember that C++ templates are not functions; they merely instantiate
into _monomorphic_ functions, so this is not the same. Polymorphic functions in Haskell and ML
have a truly polymorphic type for a single identity; the same function can be referenced, passed
around, and still used polymorphically at a distance. This property is also true of C functions
that use `void *` - they can be referenced by pointer and still applied to different argument
types - and is the one we wish to preserve.

To express this idiomatically in C, we could use `void *`, and the function signature would
probably look something like:

    void map (void * in
            , void * out
            , void (* op) (void const *, void *)
            , void * (* step) (void *));

(either the `step` function, or a step size and some crafty casts to `char *`, is required to
actually navigate the lists in the absence of size information about the elements)

Nothing prevents us from providing a mistyped `op` - or even a mistyped `step`:
not even _navigating_ a polymorphic sequence is statically assured to make sense!

Surprisingly, this is actually possible to enforce (with a lot of supporting verbosity) in pure
C11. However, the technique is too verbose and fragile for real-world use and does not scale at
all. Examples are provided at the end of an untyped program which can miscompile, a typechecked
C11 program, and an equivalent typechecked program using the `void`-_which-binds_, showing the
relative verbosity and the improvements in readabilty of the full feature.

## Proposal

We propose to add two new attributes which can appertain to the type `void`, when it appears as the
_referenced type_ of a pointer type used as either a function parameter or a function return. These
are:

    [[bind_var (id)]]
    [[bind_type (type)]]

where `id` is an identifier that introduces a _type variable_, and `type` is a concrete _type name_
other than `void` (it does not otherwise need to be complete).

The names of these attributes are tentative.

Syntactically, because `bind_var` and `bind_type` appertain to the type `void` itself, they appear
after it in a pointer type specifier. They may also appertain to a typedef of `void`.

(note that the syntax requires them to appear at the end of the specifier list, so if the type is
qualified-`void` e.g. `void const`, they have to appear `void const [[here]]` and not
`void [[here]] const`, which forces the use of "west const" `const void [[here]]` if the `void`
and the attribute originate from the same macro)

Both of the `bind` attributes add _constraints_ which restrict which pointer types the given
parameter or return value may be converted to without an explicit cast. Effectively both of these
attributes add virtual qualifiers that associate with the pointed-to type and therefore cannot be
discarded except by explicit cast-off.

Only `bind_type` can be used with a return type. While it is possible to specify a system that also
deduces signatures from return types, in practice users do not seem to need this and it reduces the
number of "surprising" errors.

### `bind_type`

The simpler attribute is `bind_type`. This adds the _constraints_ that the pointer with the
annotated referenced type can only implicitly convert to or from a (suitably qualified) pointer to
`type`, or be assigned to another (suitably qualified) pointer to `void` with the same annotation.

For example:

    void const [[bind_type (float)]] *
    float_op_impl (void const [[bind_type (float)]] * p1, void [[bind_type (float)]] * p2)
    {
      float const * fp1 = p1; // OK, bound type
      float * fp2 = p2; // OK
      
      int  const * ip1 = p1; // new constraint violation
      void const * vp1 = p1; // new constraint violation
      int  const * ip2 = p2; // new constraint violation
      void const * vp2 = p2; // new constraint violation
      
      fp2 = p1; // (existing constraint violation)
      
      ip2 = (int  *)p2; // OK, explicit cast
      vp2 = (void *)p2; // OK, explicit cast-off of bind_type
      
      return fp1; // OK, bound type
      return vp1; // new constraint violation
      return (float const *)ip1; // ...OK at least by these rules
      
      return p1; // OK, same bind_type
    }

This attribute is not really intended to be used in polymorphic functions themselves, but rather in
the operators passed to the polymorphic function, which are monomorphic in themselves but usually
will require a signature that explicitly works with `void *` (because `int * (*) (int *)` and
`void * (*) (void *)` are not convertible or guaranteed to have similar representations), in order
to be passed to the actual polymorphic function (but see below for more on both of these points).

### `bind_var`

This attribute is used on the parameters of the actual polymorphic function. It is also appertained
to the parameters of any callback operators accepted by the polymorphic function, within its own
signature.

`bind_var` adds the _constraint_ that whatever the referenced types of the pointers that are
converted from (possibly-qualified) `void *` in the argument types, must be consistent between all
`void` pointers annotated with the same `id`, within each instance of a call to that function.

In other words it introduces a _type variable_ and all `void *` annotated with the same _type
variable_ must convert to or from the same (ignoring qualifiers) pointer type.

For example:

    void [[bind_type (A)]] *   // (don't deduce from return)
    select_first (void [[bind_var (A)]] * a, void [[bind_var (A)]] * b, void [[bind_var (B)]] * c)
    {
      return a; // OK, a must have bound to an A*
      return b; // OK, b must have bound to an A*
      return c; // new constraint violation, c may have bound to a different type
    }
    
    int * ip;
    float * fp;
    
    ip = select_first (ip, ip, fp); // OK, a and b are both A
    fp = select_first (fp, fp, ip); // same
    
    ip = select_first (ip, ip, ip); // OK, A and B can both be the same, it just doesn't know that
    
    ip = select_first (fp, fp, ip); // new constraint violation, mismatched A between return/param
    select_first (ip, fp, fp);      // new constraint violation, mismatched A between params 1 & 2

The two attributes combine: `bind_type` can bind its _type variable_ to the type introduced by a
`bind_var` annotation, to allow operator callbacks to be type-checked against data arguments to a
higher-order polymorphic function:

    void map (
        void [[bind_var (A)]] * in
      , void [[bind_var (A)]] * out
      , void (* op) (void const [[bind_type (A)]] *, void [[bind_type (A)]] *)
      , void [[bind_type (A)]] * (* step) (void [[bind_type (A)]] *)
    );

The distinction is that `bind_var` deduces a value for the type variable from context, while
`bind_type` enforces that the conversion is consistent when there is no input context (it is for
conversions that will occur within the function body, from typed-`void` to typed-`void`, and can
therefore be used in declarations local to the definition and in the return type).

`bind_type` therefore works with untyped callbacks and enforces a type for their arguments _within_
the function's scope only. If the callback had typed-`void` parameters already, `bind_var` could
also appear here and _deduce_ a type from the callback's existing `bind_type` attributes (which
would still have to be consistent with the type deduced from `in` and `out` in this particular
example).

In the above declaration of a polymorphic map (implementation is in the Examples section), all of
the `void *` type components have been annotated as having to bind to the same concrete object
type.
Therefore:

    int i1[10], i2[10];
    float f1[10], f2[10];
    
    void int_op (void const [[bind_type (int)]] *, void [[bind_type (int)]] *);
    void [[bind_type (int)]] * int_step (void [[bind_type (int)]] *);
    
    void float_op (void const [[bind_type (float)]] *, void [[bind_type (float)]] *);
    void [[bind_type (float)]] * float_step (void [[bind_type (float)]] *);
    
    map (i1, i2, int_op, int_step);     // OK, all void * bind to int *
    map (f1, f2, float_op, float_step); // OK, all void * bind to float *
    
    map (i1, i2, float_op, float_step); // new constraint violation - operator types do not match data
    map (i1, f2, int_op, float_step);   // new constraint violation - operator and data types mismatch

Finally, if a polymorphic function is called from within another polymorphic function, the _type
variables_ bind to the _type variables_ of the `void *` pointers passed directly to it, rather than
to `void`:

    // yikes!
    void map2 (
        void [[bind_var (A)]] * in1
      , void [[bind_var (A)]] * out1
      , void (* op1) (void const [[bind_type (A)]] *, void [[bind_type (A)]] *)
      , void [[bind_type (A)]] * (* step1) (void [[bind_type (A)]] *)
    
      , void [[bind_var (B)]] * in2
      , void [[bind_var (A)]] * out2
      , void (* op2) (void const [[bind_type (B)]] *, void [[bind_type (B)]] *)
      , void [[bind_type (B)]] * (* step2) (void [[bind_type (B)]] *)
    ) {
      map (in1, out1, op1, step1); // OK, all void * bind to A *
      map (in2, out2, op2, step2); // OK, all void * bind to B *
    
      map (in1, out1, op2, step2); // new constraint violation - operator types do not match data
      map (in1, out2, op1, step2); // new constraint violation - operator and data types mismatch
    }

Within the function body, `void`-pointers annotated by `bind_var` may only implicitly convert to
other `void`-pointers with the same type variable. Conversions to other types, including pointers
to `void` without the same type variable (inc. no type variable), require an explicit cast.

A `void`-pointer annotated by `bind_type` with a type variable that was introduced by `bind_var`
has the same constraint (consistent with the constraint if it had named a concrete type).

### `qual_var` and `qual_with`

Previous versions of this proposal introduced two further attributes, `qual_var` and `qual_with`,
which introduced variables for _qualification_ of a pointed-to object type.

These attributes implemented a feature which is already in C23 but currently uses ad-hoc signatures,
namely the qualifier-generic search functions. These provide the best examples:

    char [[qual_with (Q)]] * strchr (const char [[qual_var (Q)]] * s, int c);

This part of the proposal was separately rejected by WG14 during the Fall 2024 meeting as being
too complicated to introduce in the first version of the change. It may be revisited in a future
proposal, building atop `bind_var` and `bind_type` if they have been accepted into the document.
For the time being, we already have the _QVoid_ notation.

### Casting to generic function types

The "operator" functions that make use of the `bind_type` attribute suffer from an unfortunate
usability problem, which is that although the concrete type of the pointed-to object is known by
the function definition, the pointers cannot be dereferenced because they still have an incomplete
C language type: the attribute does not change the meaning of the annotated C code, so there is no
ability added to use the pointers "conveniently", without converting the pointer type.

This is hardly insurmountable but makes the code much less elegant to read:

    // compar functions for qsort:
    
    int cmp_float_void (void const [[bind_type(float)]] * lv
                      , void const [[bind_type(float)]] * rv) {
        float const * lhs = lv;
        float const * rhs = rv; // this verbosity isn't great
        return *lhs < *rhs ? -1
             : *lhs > *rhs ? +1
             :                0;
    }
    
    int cmp_float_simple (float const * lhs, float const * rhs) {
        return *lhs < *rhs ? -1
             : *lhs > *rhs ? +1
             :                0;
    }

Consequently we propose that it should be permitted to cast a pointer to any given function type,
to a pointer to a function type with the same signature but with some or all of the argument types
or the return type substituted for (equivalently-qualified) pointers to `void`, annotated by
`bind_type` with the substituted type name, if they were pointers to complete object types in the
original signature. This would allow:

    qsort (base, n, sz, (Compar)cmp_float_simple);

even though `cmp_float_simple` doesn't have the same signature as the `compar` for `qsort` and is
not a compatible function type.

We propose that this permission should only be made available to (possibly parenthesized/generically
selected) _identifiers_ that declare a function, and not to generalized function pointers. The 
reason for the restriction is that the conversion is not guaranteed to be a no-op on targets where
data pointers can have different representations, and therefore function types that differ only in
parameter and return pointer types can potentially have different ABIs. This primarily affects the
embedded space. A cast of an identifier which designates a specific function can be implemented by
a compiler-generated shim function wrapping that specific function in the new signature, but this
cannot implement generalized (non-constant) function pointers.

    (Compar)cmp_float_simple
    
    // compiler inserts the static shim:
    static int @shim$cmp_float_simple (void const [[bind_type(float)]] * lv
                                     , void const [[bind_type(float)]] * rv) {
        return cmp_float_simple (lv, rv);
    }
    
    (Compar)some_dynamic_pointer
    
    // static shim is not possible

There is an existing requirement in 6.3.2.3 that when a pointer to a function is converted to
a pointer to a function of another type and back again, the result compares equal to the original
pointer. This is achievable with compiler-inserted shims, since the compiler (or at least, the
linker) would be able to construct a list of the generated shim functions and their originals
to check for in a back-conversion. This is not free, but on platforms where the conversion is not
trivial anyway, is potentially an acceptable cost as users are less likely to be freely converting
between function types in that situation. (on common desktop platforms where the shim is not
needed anyway, the conversion back will always be a no-op too)

### Library changes

We would propose that the `bind_type` and `bind_var` attributes apply immediately to the functions
`qsort` and `bsearch` in `<stdlib.h>` and to `memchr` in `<string.h>`:

    void [[bind_type (T)]] * bsearch (void const * key
        , void const [[bind_var (T)]] * base
        , size_t nmemb
        , size_t size
        , int (*compar)(void const [[bind_type (T)]] *, void const [[bind_type (T)]] *));
    
    void qsort (void [[bind_var (T)]] * base
        , size_t nmemb
        , size_t size
        , int (*compar)(void const [[bind_type (T)]] *, void const [[bind_type (T)]] *));
    
    void [[bind_type (T)]] * memchr  (void const [[bind_var (T)]] * s,  int c, size_t n);

i.e. the functions do not deduce object type from the return conversion, and
neither `bsearch` nor `qsort` deduce object type from `compar`, only from `base`.

(The bulk of the originally-proposed library changes concerned attributes which are no longer part
of this proposal.)

## Alternatives

The names and exact placement of the attributes can be subject to further discussion. In practice
we expect that users would wrap them inside macros anyway, for instance:

    #define Auto(T) void [[bind_var (T)]]
    #define Void(T) void [[bind_type (T)]]
    
    void map (
        Auto (A) * in
      , Auto (A) * out
      , void (* op) (const Void (A) *, Void (A) *)
      , Void (A) * (* step) (Void (A) *)
    );
    
    void int_op (const Void (int) *, Void (int) *);

(again note that the choice to deduce from both `in` and `op`, but not `out` or `step`, is a design
decision and all four could be deduced from, or even just one of the four; wherever the type
information exists)

Overloads and `_Generic` do not provide a suitable alternative to this feature. The ad-hoc
polymorphism provided by `_Generic` operates on a fixed set of known types. The operator name that
it can be used to construct is a second-class language feature that does not have a single address.
To pass such an operator around, it must be deconstructed somehow and a non-generic overload
selected.

This functionality is completely orthogonal; where an ad-hoc polymorphic "overloaded" function
provides _different_ suitable implementations for a number of different types, a parametrically
polymorphic function is _completely unaware_ of the concrete type of its arguments and always
provides the exact same implementation regardless of it.

Templates are similarly orthogonal: they do not create a first-class polymorphic language feature
(remaining polymorphic when passed by value), but instantiate into separate monomorphic functions.
They can also specialize, which betrays the principle of always providing the _exact_ same body
implementation down to the instruction level.

### Dynamic types

[A different proposal targeting C2y][4] suggests adding runtime-polymorphic types to the language.
This feature can be used to implement essentially any algorithm that could be expressed using
parametric polymorphism, but does not perform any type-checking. Essentially it expands the
functionality of `void *` itself to allow whole object footprints to become untyped. In general
we feel like these proposals are best suited to target different use cases - N3212 is better
suited to use cases where the object type really is intended to be _dynamic_, such as relying
on a dynamic object footprint, rather than functions where it is simply abstracted and can be
checked for consistency against the other arguments. It also does not propose any way to improve
type safety, so does not help with any case where the existing `void *` feature is already enough
to implement an algorithm but is not enough to be sure it is correct. The `sort` examples from
N3212 are an area which we feel this proposal handles better, while the `atomic` examples are
a use case that this proposal cannot handle at all and benefit very well from its dynamic
approach.

## Impact

The implementation impact is lower than it seems.

Firstly, there is **no ABI impact** whatsoever from this feature. Because the constraints are
applied by attributes, they do not change the representation type of the pointers to generic data,
which remain `void *` at all times. There is no binary difference between a program that chooses
to make use of this typechecking feature and one that does not.

This is different from C++, where a template function instantiates into multiple implementations
with semantically separate identities and which all have different signature types. There remains
only one polymorphic function, with one type, and whose type remains polymorphic even after it is
referenced indirectly.

This follows directly from the principle that a correct program with standard attributes _remains_
correct and has the exact same observable meaning if the attributes are removed.

Secondly, if polymorphic type checking proves too difficult for a smaller implementation, since
these constraints are only introduced by attribute - the implementation can choose to be unable
to diagnose violations of the constraints and will still be conforming, because it will always
compile a correct program to exactly the same meaning as a higher-end compiler that does
understand the attributes. Therefore, implementations are eased-in to needing to support
polymorphism, and do not need to provide full checking in order to conform.

It is possible and entirely plausible that only analysis tools and not primary compilers would
ever bother to implement the constraints for these attributes.

Thirdly, implementation experience with Helix QAC suggests that ML-style type inference with
unification is "not too difficult" for a small team (one) to implement in a short amount of time.
We found that C could easily be extended with back-propagating inferred pointer types, which are
used to drive a number of different type-based (proprietary) analyses. (In QAC these properties
are mostly inferred from usage rather than specified explicitly by user attributes.)

ML subset compilers are frequently implemented as student projects, so this is a widely understood
feature in the broader compiler community.

## Future directions

### For container types

A request that immediately arose from the first version of this proposal was that `bind_var` should
also be available for use in structures, to make container types slightly more resilient.

If `bind_var` and friends are adopted for use with function types, it therefore follows that the
logical next line of development would be to define the attributes in order to allow syntax along
the lines of the following:

    [[bind_var (T)]] struct LinkedList {
        struct LinkedList [[bind_type (T)]] * next;
        void [[bind_type (T)]] * data;
    };

Such a structure definition _is the same type_ as a generic linked list that can hold elements of
any type, but enforces that the type held in `data` is consistent with the declaration of the pointer
to the head, and that the type of the node pointed to by `next` will similarly be consistent with the
type of this node. This would enforce that the list only contains elements of a single element type
without having to instantiate or metaprogram-up a specific `LinkedList` for the member type. This has
the advantage that containers of distinct (but internally-consistent) element types can be handled
by algorithm functions (`map` etc.) that are themselves fully polymorphic; if `LinkedList` needed to
be instantiated into multiple struct types, the handling functions would themselves also need to
have a distinct instance for each concrete type, ending up with something like C++ templates instead
of a proper parametrically-polymorphic system with only one generic function and one generic
container type.

Note that unlike polymorphic functions, which deduce bindings for the type variable from use,
polymorphic types would need to have the type explicitly declared:

    struct LinkedList [[bind_type (int)]] * head = malloc (sizeof (struct LinkedList));

Note it is not possible to write `auto [[bind_type (int)]] * head =` in the current version of the
language because of syntax limitations on `auto` (the star is an implementation-defined extension).
This is a separate change proposed by [N3579 "`auto` as a placeholder type specifier"][5] which
has not yet been integrated.

We do not propose wording for this feature at this time, though overall the development should be
simpler than the basis established for function types, and the extension seems reasonable.

### For dynamic types

Although the feature described by [N3212][4] generally fits better against other use cases, it has
room to compose well with `void`-_which-binds_ in situations where a function accepts more than one
dynamically-typed argument. For instance:

    void generic_from_to (_Type T
        , _Var(T) const * from
        , _Var(T)       * to) {
      T temporary;
      // work using the temporary, from, and to
    }

could avoid having, and needing to check, a _dependent type_ in the signature by using `void *`:

    void generic_from_to (_Type T
        , void const [[bind_type (T)]] * from
        , void       [[bind_type (T)]] * to) {
      T temporary;  // can still do this bit
      // same work because `_Var(X) *` is defined to be a `void *` in ABI
    }

... by using `bind_type` to say that the pointed-to type of `from` and `to` is _statically_ known
to be (compatible with) `T`, even though `T` itself is dynamic.

Wording to enable this integration between static and dynamic polymorphic types is not included
in the proposed wording below because dynamic polymorphic types have not been adopted, but should
certainly be investigated as a future direction if they are integrated into C2y. Such wording
would only need to allow the use of the _object_ identifier declared as a `_Type` as the operand to
`bind_type`, and should form a straightforward enhancement.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3886][0]. Bolded text is new text when inlined into an existing sentence.

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

### New attributes

Modify 6.7.12 "Attributes":

Modify 6.7.12.2 to add the four new standard attribute names:

> The identifier in a standard attribute shall be one of:  
>   **`bind_type`**  
>   **`bind_var`**  
>   `deprecated`  
>   `fallthrough`  
>   `maybe_unused`  
>   `nodiscard`  
>   `noreturn`  
>   `_Noreturn`  
>   `unsequenced`  
>   `reproducible`  

(EDITORIAL: not in alphabetical order?)

Note that 6.7.12.2 paragraph 3 and footnote 145 **remain true** for the new attributes without
changes.

Add a new section after 6.7.12.8:

> **6.7.12.9 The `bind_var` and `bind_type` attributes**
>
> **Constraints**
>
> The `bind_var` attribute shall be applied to the referenced type of a pointer in the parameter
> types of a function declarator.
>
> `bind_var` shall have an argument clause, which shall have the form  
>    `(` _identifier_ `)`
>
> The `bind_type` attribute shall be applied to the referenced type of a pointer in the return or
> parameter types of a function declarator, or to the referenced type of a pointer type specified
> within the definition of a function that uses `bind_var` in its signature.
>
> `bind_type` shall have an argument clause, which shall have the form  
>    `(` _type-name_ `)`  
> or  
>    `(` _identifier_ `)`  
> where _identifier_ is an identifier introduced by a `bind_var` attribute appearing in the
> function signature.
>
> The `bind_var` and `bind_type` attributes shall not be applied to any type other than
> a qualified or unqualified version of `void`. <sup>footnote)</sup>
>
> <small>footnote) they can therefore apply to aliases for `void` created by `typedef`.</small>
>
> Within the scope of its declaration, a _binding pointer_ shall only be converted to a pointer
> binding to the same identifier, or to a pointer constrained to the same identifier, except by
> means of an explicit cast.
>
> Within the scope of its declaration, a _constrained pointer_ shall only be converted to a pointer
> binding to the same identifier, or to a pointer constrained to the same identifier or type name,
> or to a pointer to (possibly-qualified) `void`, except by means of an explicit cast.
>
> For each binding pointer in the type of the called function that binds to the same identifier,
> the determined _actual types_ shall all be compatible.
>
> For each constrained pointer in the type of the called function that is constrained to an
> identifier, the actual type shall be compatible with the actual type of all binding pointers
> in the type of the called function that bind to that identifier.
>
> For each constrained pointer in the type of the called function that is constrained to a type
> name, the actual type shall be compatible with the constrained type, or `void`.
>
> **Semantics**
>
> The `bind_var` and `bind_type` attributes indicate that a pointer declared as a
> possibly-qualified pointer to `void` is intended to be used with statically-checked types.
>
> A pointer that specifies the `bind_var` attribute is a _binding pointer_. A _binding pointer_
> with an identifier _A_ as its attribute argument is _binding to A_. The identifier _A_ is
> introduced into a virtual scope beginning from the first token of the nearest containing function
> declarator and terminating at the end of that declarator's associated definition or declaration. <sup>footnote)</sup>
> Multiple uses of `bind_var` with the same identifier in the same declarator refer to the same
> binding.
>
> <small>footnote) this means a `bind_var` can introduce an identifier to be used by a `bind_type`
> that appears in an earlier parameter or in the return type, and also in the function body.</small>
>
> A pointer that specifies the `bind_type` attribute is a _constrained pointer_. A _constrained_
> _pointer_ with an identifier _A_, that is introduced into the virtual scope by `bind_var`,
> is _constrained to A_. A _constrained pointer_ with a type name _T_ as its attribute argument is
> _constrained to T_.
>
> If an identifier has been bound by `bind_var` in the current scope, and is also visible in the
> current scope as a typedef name, the `bind_type` attribute interprets it as the identifier
> bound by `bind_var`.
>
> For a function call expression, if a parameter of the called function is or derives from a
> _binding pointer_, then the corresponding type ignoring qualification in the type
> of the argument, or the type that the return type is converted or assigned to, determines the
> _actual type_<sup>footnote)</sup>. If the _actual type_ is a binding pointer or constrained pointer
> in the calling scope, the _actual type_ is the identifier or type name it is binding or constrained
> to; otherwise, the _actual type_ is the referenced type; or `void` if the corresponding type in the
> calling context is not a pointer.
>
> <small>footnote) this means that two instances of `bind_var` appearing in two different parameter
> types, either determine the same _actual type_, or violate a constraint.
>
> **NOTE** there is no restriction against two binding pointers with distinct identifiers
> binding to or from the compatible types outside the function, but internally the function
> treats them as unrelated.
>
> The constraints imposed by the binding pointers and constrained pointers within a type also
> apply to uses of that type when specified by a typedef name or by the result of `typeof`. Any
> uses of a composite type formed from two such types are subject to the union of both sets of
> constraints. Such constraints do not affect whether two types are compatible.
>
> Any constraints that apply to the type of an expression used to initialize an object with inferred
> type also apply to the type of the declared object.
>
> The `__has_c_attribute` conditional inclusion expression (6.10.2) shall return the value ######L
> when given `bind_var` or `bind_type` as the _pp-tokens_ operand, if the implementation supports
> the attribute.
>
> **Recommended Practice**
>
> The `bind_var` attribute should be used in the parameters of a polymorphic function to indicate
> that two separate generic pointers in its signature are intended to work
> on the same concrete type, in any given invocation.
>
> This attribute allows usage of the polymorphic function to be type-checked against the concrete
> types of its arguments and destination, as well as the expected types to be operated on by any
> callback operators passed to it.
>
> An implementation is encouraged to emit a diagnostic if the _identifier_ introduced by `bind_var`
> has also been declared in any name space in the current scope when the attribute is encountered,
> or is used to declare an ordinary identifier within the range of the virtual scope of the
> binding. An implementation is encouraged to emit a diagnostic if the same identifier is reused
> in a virtual scope nested within a surrounding virtual scope (i.e. within the parameters of a
> parameter).
>
> An implementation is encouraged to emit a diagnostic if two declarations of a function do not
> make consistent use of the `bind_var` and `bind_type` attributes.
>
> The `bind_type` attribute should be used with a type name in the parameters or return type of a
> callback operator that is declared with a signature that uses pointers to `void` because it is
> intended to work with polymorphic higher-order functions (like `qsort`), but that is itself
> intended to operate on values with a concrete type.
>
> The `bind_type` attribute used with an identifier should be used to indicate when the concrete
> type of the pointed-to object is expected to be compatible with the concrete type pointed to by a
> binding pointer. This can for instance communicate that a return type is the same as a parameter
> type; or that an output parameter points to the same type as an input parameter but that it
> doesn't contribute to "deduction".
>
> **EXAMPLE 1**  This polymorphic function returns a reference to an element within the first
> buffer. It cannot correctly return a reference to an element within the second buffer:
>
>     int i1[10];
>     float f1[10];
>     
>     void [[bind_type (A)]] * first (void [[bind_var (A)]] * p1
>                                   , void [[bind_var (B)]] * p2) {
>         return p1; // correct
>         // return p2; // constraint violation
>     }
>     
>     int   * ip = first (i1, f1); // OK
>     float * fp = first (f1, i1); // OK
>     ip = first (i1, i1); // also OK, the function just doesn't know A and B are compatible here
>     fp = first (i1, f1); // constraint violation, A and A do not match
>
> **EXAMPLE 2**  `qsort` is a polymorphic function that needs to accept a compare function that
> will work on its base array, so the following signature can communicate this:
>
>     void my_qsort (void [[bind_var (T)]] * base
>                  , size_t nmemb
>                  , size_t size
>                  , int (*compar)(void const [[bind_type (T)]] *
>                                , void const [[bind_type (T)]] *));)
>     
>     my_qsort (f1, 10, sizeof (float), float_compare); // OK
>     my_qsort (i1, 10, sizeof (int), float_compare); // constraint violation, A and A do not match
>
> The declaration of `compar` must use `bind_type`, not `bind_var`, because it needs to be consistent
> with the _actual type_ deduced for `base`; but it doesn't make sense for `compar` itself to be used
> to deduce the element type.
>
> **EXAMPLE 3**  This callback is intended to be passed to `qsort` to sort an array of `float`
> values. Its operating parameter types are float pointers, but the signature of `qsort` requires
> them to be declared as pointers to `void`:
>
>     int float_compare (const void [[bind_type (float)]] * l
>                      , const void [[bind_type (float)]] * r) {
>         const float * fl = l;
>         const float * fr = r;
>         
>         return *fl < *fr ? -1
>              : *fl > *fr ? +1
>              :             0;
>     }
>
> **EXAMPLE 4**  This polymorphic function only intends to deduce the type `A` from its `in`
> parameter, and requires `out` to match its type exactly:
>
>     void move_bytes (size_t sz
>         , void const [[bind_var (A)]]  * in
>         , void       [[bind_type (A)]] * out);
>     
>     int const a[10] = { ... };
>     int b[10];
>     move_bytes (sizeof a, &a, &b); // Note that the actual type of A is int,
>                                    // not int const, so these match
>
> This can be helpful for communicating intent more clearly. This function declares that the
> returned pointer type has the same concrete object type as the parameter, without deducing the
> binding from it, which (for instance) allows it to be cast immediately to another type or
> converted to `void *`:
>
>     void [[bind_type (A)]] * do_step (size_t sz, void [[bind_var (A)]] * in);
>     
>     int i[10];
>     void * p = do_step (sizeof (int), &i[0]);  // OK - the converted result
>                                                // doesn't contribute to deduction so
>                                                // the (nop) conversion to void* is OK
>                                                // even though the actual type is int*
>
> **EXAMPLE 5**  Calls to a polymorphic function from within the body of another polymorphic
> function consider the binding identifiers the caller introduced, binding to those identifiers
> as the _actual types_ when they are available (rather than using `void` as the _actual type_):
>
>     void callee (void const [[bind_var(T)]] *, void const [[bind_var(T)]] *);
>     
>     void caller (void const [[bind_var(A)]] * x
>                , void const [[bind_var(A)]] * y
>                , void const [[bind_var(B)]] * z
>                , void const [[bind_var(B)]] * w) {
>     
>       callee (x, y);  // OK - T binds to A, T binds to A, consistent
>       callee (z, w);  // OK - T binds to B, T binds to B, consistent
>     
>       callee (x, z);  // error - T binds to A, then T binds to B, wrong
>       callee (w, y);  // error - T binds to B, then T binds to A, wrong
>     
>     }
>
> This maintains the property that `callee` is only called with arguments that point to compatible
> types by using the guarantees established by `caller`.
>
> **EXAMPLE 6**  The composite type of two polymorphic function types combines the effect of both
> of their bindings and constraints. Given:
>
>     // first declaration: says x and w are compatible, y and z are compatible
>     void f1 (void const [[bind_var (A)]] * x
>            , void const [[bind_var (B)]] * y
>            , void const [[bind_var (B)]] * z
>            , void const [[bind_var (A)]] * w);
>     
>     // second declaration: says x and y are compatible, z and w unknown
>     void f1 (void const [[bind_var (A)]] * x
>            , void const [[bind_var (A)]] * y
>            , void const                  * z
>            , void const                  * w);
> 
> ...the resulting composite type is as though it had been written:
>
>     void f1 (void const [[bind_var (A1), bind_var (A2)]] * x
>            , void const [[bind_var (B1), bind_var (A2)]] * y
>            , void const [[bind_var (B1)]]                * z
>            , void const [[bind_var (A1)]]                * w);
>
> ...implying that all four parameters are to compatible types. However, the reuse of the lexeme
> introduces two separate bindings with distinct virtual scopes, so given:
>
>     // first declaration: says x and w are compatible, y and z are compatible
>     void f2 (void const [[bind_var (A)]] * x
>            , void const [[bind_var (B)]] * y
>            , void const [[bind_var (B)]] * z
>            , void const [[bind_var (A)]] * w);
>     
>     // second declaration: says x and w are compatible, y and z are compatible
>     void f2 (void const [[bind_var (B)]] * x
>            , void const [[bind_var (A)]] * y
>            , void const [[bind_var (A)]] * z
>            , void const [[bind_var (B)]] * w);
> 
> ...the resulting composite type has no overlapping binding effects, as though it had been written:
>
>     void f2 (void const [[bind_var (AB1)]] * x
>            , void const [[bind_var (BA1)]] * y
>            , void const [[bind_var (BA1)]] * z
>            , void const [[bind_var (AB1)]] * w);
>
> For both of these cases, the recommended practice encourages a diagnostic.
>
> **EXAMPLE 7**  When an object is declared with inferred type using a value whose type is or is
> derived from a _constrained pointer_, the type of the declared object is similarly-constrained:
>
>     void const [[bind_type (T)]] * get (void const [[bind_var (T)]] *);
>     
>     auto p1 = get (float_ptr);  // p1 is constrained as if it was declared
>                                 // void const [[bind_type (float)]] * p1 = ...
>     
>     auto p2 = get (int_ptr);    // p2 is constrained as if it was declared
>                                 // void const [[bind_type (int)]] * p2 = ...

### Permissive function casts

OPTIONALLY (this change is subject to its own question and poll):

Modify 6.3.3.3 "Pointers", paragraph 11:

> A pointer to a function of one type may be converted to a pointer to a function of another type
> and back again; the result shall compare equal to the original pointer. **If the return type or**
> **any parameter type of the converted type is a pointer to possibly-qualified void, and the **
> **corresponding return type or parameter type of the type before conversion is a pointer to an**
> **identically-qualified object type, and the expression being converted is an identifier declared**
> **as having function type <sup>footnote1)</sup>, using the converted pointer to call the underlying**
> **function has the same effect as a call through a pointer of the original type<sup>footnote2)</sup>.
> Otherwise, ** if a converted pointer is used to call a function whose type is not compatible with
> the referenced type, the behavior is undefined.
>
> <small>footnote1) as opposed to an identifier declared as having pointer to function type.</small>
>
> <small>footnote2) however, these pointers might have different address representations.</small>

### Library changes

Modify 7.25.6.2 "The `bsearch` generic function", changing the signature in the synopsis:

>     #include <stdlib.h>
>     QVoid [[bind_type (T)]] * bsearch (void const * key
>         , QVoid [[bind_var (T)]] * base
>         , size_t nmemb
>         , size_t size
>         , int (*compar)(void const [[bind_type (T)]] *, void const [[bind_type (T)]] *));

Modify 7.25.6.3 "The `qsort` function", changing the signature in the synopsis:

>     #include <stdlib.h>
>     void qsort (void [[bind_var (T)]] * base
>         , size_t nmemb
>         , size_t size
>         , int (*compar)(void const [[bind_type (T)]] *, void const [[bind_type (T)]] *));

Modify 7.28.5.2 "The `memchr`generic function", changing the signature in the synopsis:

>     #include <string.h>
>     QVoid [[bind_type (T)]] * memchr (QVoid [[bind_var (T)]] * s, int c, size_t n);

## Questions for WG14

Would WG14 like to add something along the lines of the `bind_type` and `bind_var` attributes
specified in N3586 to C2y?

Would WG14 like to allow function designator expressions to be cast to more-generic function
pointers, as specified in N3586, in C2y?

Would WG14 like to change the specification of the generic search and sort functions `bsearch`,
`qsort`, and `memchr` to include the `bind_var` and `bind_type` attributes?

Would WG14 like to see something along the lines of `bind_var` and `bind_type` for structure
or container types?

## Acknowledgements

Thanks to Joseph Myers, Jens Gustedt, and Martin Uecker, for detailed review comments!

## References

[C2y latest public draft][1]  
[Parametric polymorphism][2]  
[Towards type-checked polymorphism][3]  
[Polymorphic Types][4]  
[`auto` as a placeholder type specifer][5]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3886.pdf
[2]: https://en.wikipedia.org/wiki/Parametric_polymorphism
[3]: https://gist.github.com/Leushenko/ad2db35940a77d7dc8575cf9744402f9
[4]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3212.pdf
[5]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3579.htm

## Examples

### Example 1: untyped

    // basic array mapper
    // we can use it with mis-typed arguments
    
    typedef void (* Mutate) (void *, void *);
    typedef void * (* Step) (void *);
    
    void addOneInt   (void * in, void * out) {   *(int *)out =   *(int *)in + 1;    }
    void addOneFloat (void * in, void * out) { *(float *)out = *(float *)in + 1.0f; }
    
    void * step_float (void * p) { return (float *)p + 1; }
    void * step_int   (void * p) { return   (int *)p + 1; }
    
    int ia[10];
    float fa[10];
    
    void map (void * array_in, void * array_out, int size, Mutate mut, Step step) {
        void * in = array_in;
        void * out = array_out;
        for (int i = 0; i < size; ++ i, in = step (in), out = step (out)) {
            mut (in, out);
        }
    }
    
    void incrArrays (void) {
      map (ia, ia, 10, addOneInt, step_int);
      map (fa, fa, 10, addOneFloat, step_float);
    
      // oh no: this also compiles, because of void*
      map (fa, fa, 10, addOneInt, step_float);
      map (ia, fa, 10, addOneInt, step_float);
      map (ia, fa, 10, addOneInt, step_float);
      map (ia, ia, 10, addOneInt, step_float);
      map (fa, fa, 10, addOneInt, step_int);
    
      map (ia, ia, 10, addOneFloat, step_int);
      map (ia, fa, 10, addOneFloat, step_int);
      map (ia, ia, 10, addOneFloat, step_int);
      map (ia, ia, 10, addOneFloat, step_int);
      map (ia, ia, 10, addOneFloat, step_float);
    }

### Example 2: semiautomatic typechecking with brittle verbosity (C11+)

    // basic array mapper
    // enhanced with type checking despite accepting arrays of any type - checks
    // that the operand kind of the mapped function matches the array element type
    //  i.e. map :: ([T], T -> T) -> [T]
    
    typedef void (* Mutate) (void *, void *);
    typedef void * (* Step) (void *);
    
    void addOneInt   (void * in, void * out) {   *(int *)out =   *(int *)in + 1;    }
    void addOneFloat (void * in, void * out) { *(float *)out = *(float *)in + 1.0f; }
    
    void * step_float (void * p) { return (float *)p + 1; }
    void * step_int   (void * p) { return   (int *)p + 1; }
    
    int ia[10];
    float fa[10];
    
    void map_impl (void * array_in, void * array_out, int size, Mutate mut, Step step) {
      void * in = array_in;
      void * out = array_out;
      for (int i = 0; i < size; ++ i, in = step (in), out = step (out)) {
        mut (in, out);
      }
    }
    
    
    #define FunctionDescriptor(Type, Func) union { Func func; Type T; }
    
    #define same_type(A, B) _Generic(1 ? (A) : (B)  \
      , void *: 0                                   \
      , void const *: 0                             \
      , void volatile *: 0                          \
      , void const volatile *: 0                    \
      , default: 1)
    #define check_same_type(A, B) _Static_assert (same_type (A, B), "types must match");
    
    #define check_array_size
    
    #define map(in, out, size, mut, step) do {                 \
      check_same_type ((in), (out));                           \
                                                               \
      check_same_type ((in), &(mut).T);                        \
      check_same_type ((in), &(step).T);                       \
                                                               \
      map_impl ((in), (out), (size), (mut).func, (step).func); \
    } while (0)
    
    
    typedef FunctionDescriptor (int, Mutate) MutInt;
    typedef FunctionDescriptor (int, Step) StepInt;
    typedef FunctionDescriptor (float, Mutate) MutFloat;
    typedef FunctionDescriptor (float, Step) StepFloat;
    
    MutInt addOneInt_g;
    StepInt stepInt_g;
    
    MutFloat addOneFloat_g;
    StepFloat stepFloat_g;
    
    
    void incrArrays (void) {
      map (ia, ia, 10, addOneInt_g, stepInt_g);
      map (fa, fa, 10, addOneFloat_g, stepFloat_g);
    
      // no longer compile!
      // map (fa, fa, 10, addOneInt, step_float);
      // map (ia, fa, 10, addOneInt, step_float);
      // map (ia, fa, 10, addOneInt, step_float);
      // map (ia, ia, 10, addOneInt, step_float);
      // map (fa, fa, 10, addOneInt, step_int);
    
      // map (ia, ia, 10, addOneFloat, step_int);
      // map (ia, fa, 10, addOneFloat, step_int);
      // map (ia, ia, 10, addOneFloat, step_int);
      // map (ia, ia, 10, addOneFloat, step_int);
      // map (ia, ia, 10, addOneFloat, step_float);
    }

### Example 3: polymorphic typechecking with `void`-_which-binds_

    // basic array mapper
    // enhanced with type checking despite accepting arrays of any type - checks
    // that the operand kind of the mapped function matches the array element type
    //  i.e. map :: ([T], T -> T) -> [T]
    
    #define Auto(T) void [[bind_var (T)]]
    #define Void(T) void [[bind_type (T)]]
    
    #define Mutate(T, N) void (* N) (Void (T) *, Void (T) *)
    #define Step(T, N) Void (T) * (* N) (Void (T) *)
    
    void addOneInt   (Void (int)   * in, Void (int)   * out) {   *(int *)out =   *(int *)in + 1;    }
    void addOneFloat (Void (float) * in, Void (float) * out) { *(float *)out = *(float *)in + 1.0f; }
    
    Void (float) * step_float (Void (float) * p) { return (float *)p + 1; }
    Void (int)   * step_int   (Void (int)   * p) { return   (int *)p + 1; }
    
    int ia[10];
    float fa[10];
    
    void map (Auto (A) * array_in, Auto (A) * array_out, int size, Mutate(A, mut), Step (A, step)) {
        Void (A) * in = array_in;
        Void (A) * out = array_out;
        for (int i = 0; i < size; ++ i, in = step (in), out = step (out)) {
            mut (in, out);
        }
    }
    
    void incrArrays (void) {
      map (ia, ia, 10, addOneInt, step_int);
      map (fa, fa, 10, addOneFloat, step_float);
    
      // no longer compile!
      // map (fa, fa, 10, addOneInt, step_float);
      // map (ia, fa, 10, addOneInt, step_float);
      // map (ia, fa, 10, addOneInt, step_float);
      // map (ia, ia, 10, addOneInt, step_float);
      // map (fa, fa, 10, addOneInt, step_int);
    
      // map (ia, ia, 10, addOneFloat, step_int);
      // map (ia, fa, 10, addOneFloat, step_int);
      // map (ia, ia, 10, addOneFloat, step_int);
      // map (ia, ia, 10, addOneFloat, step_int);
      // map (ia, ia, 10, addOneFloat, step_float);
    }
