    Proposal for C2y
    WG14 3204

    Title:               Semantic basis for overloading
    Author, affiliation: Alex Celeste, Perforce
    Date:                2023-12-14
    Proposal category:   Clarification, non-functional
    Target audience:     Future feature authors

## Abstract

The document aims to compare and contrast the two major design approaches to
overloading as found on the one hand in C++ and Java and related languages, 
and on the other hand in languages influenced by functional programming and
strong type disciplines.
We propose that C should look to the latter group of languages for guidance
because of the simpler and clearer logical basis, the simpler integration into
the C language, and the better characteristics of code likely to be developed
by users in this system.

We try to describe typeclasses in non-theoretic terms that are likely to be
approachable to C programmers.

-------------------------------------------------------------------------------

# Semantic basis for overloading

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3204
    Revises:      N3204
    Date:         2023-12-14

## Summary of Changes
### xxxx
 - typo fixes
### N3204
 - original document
 
## Introduction

"Overloading" refers to the ability for one name (or symbol) to refer to more
than one function declared visible in the source, to be resolved to a specific
callable entity by the use of some additional context information beyond the
identifier (or symbol) and its linkage. This is often known as _static polymorphism_,
because the resolution of the name usually involves type information and is
decided at compile-time (although it can be implemented as a runtime feature: 
[[8]], [[9]]).

Taking a very coarse-grained view, there seem to be two major approaches to
the definition and resolution of overloadable entities:

- "open-set" overloading, as featured in [C++][1], [Java (JLS 15.12.2)][4], and
  extensions to C like [Clang's attribute `[[clang::overloadable]]`][2] or
  LCC's unique implementation described by [N3051][10].

- "trait" or "typeclass" overloading, as featured in [Rust][5], [Haskell][6],
  and with a limited presence in C++ after C++20 through [C++ concepts][7].

The boundary between these two approaches is fairly blurry and mostly orients
around approaching the question starting from a generic/polymorphic perspective,
vs a static/monomorphic perspective. The open-set approach provides a simple
implementation for monomorphic code with a smaller set of user-facing features,
while the typeclass approach presents more tools but allows for better control
over the definition of generic client functions.

The downside of the open-set approach is that the apparently simple set of tools
seem in practice to lead to extremely complicated backing implementations and
also to multi-stage complex resolution rules for users to remember. Providing 
"only" a single syntactic feature disguises a high degree of both implementation
and specification complexity, that combines poorly with already-complex
promotion and implicit conversion rules in the existing base C language.

Our proposal is that C should choose a high-level approach to thinking about
overloading and overload resolution before committing to implementation details
that force a specific path.

We favour the typeclass approach and argue that even though it has precedent in
C++, the open-set approach is a flawed design whose mistakes C has the opportunity
to avoid repeating.

### On infix

We **do not** generally consider infix syntax in this discussion. We believe all
points fundamentally apply to both prefix/call syntax and infix-operator syntax
alike, and will not discuss the details of what declaring or naming an infix
operator in C might look like. "Operator" will refer to overloads using 
prefix/call syntax in the rest of this discussion.

## Open sets

In C++ (and Java, and LCC, and Clang, and...) overloading is "open" in the sense
that there is no restriction on two functions in the same scope being declared
with the same name. At each call point, overload resolution simply considers
everything visible at that point, and finds the best match based on the argument
types.

Each overload is a first-class, top-level entity with a completely distinct
scope and identity. The feature is "simple", because to add multiple overloads
the user simply adds another declaration in the scope where they want to use the
different functions and everything else happens implicitly. Unfortunately out
of this apparent simplicity arise several points of complexity:

- the declarations are completely independent, meaning nothing forces them to be
  provided together or in a coherent way. It is possible to define a random
  assortment of operators for a given type, and as long as the operators the
  code _actually invokes_ are declared, the compiler is happy, regardless of
  whether they form a sensible algebraic structure or API.

- because the declarations are totally independent and not linked into any kind
  of class, group or category, the compiler has minimal information about user
  intent and has to resolve calls based on some "best-match" algorithm or other
  against the actual arguments. (This then needs to have a system for resolving
  ambiguity when there are also implicit conversions in the language.)
  
- in combination, these two facts can result in surprises if the currently-visible
  set of candidate functions is either more or less specific than the user
  expected. A syntactically-identical piece of code can also have a different
  set of candidate functions resolve calls if it is moved to interleave with
  overload declarations - which are not encouraged to be held together because
  they have to be declared in an ad-hoc way "by" (i.e. "sorta near") each type.

- because the overloads are all completely independent declarations in a scope,
  generic use of overloads falls down to a "syntactic" model - "is there anything
  in scope with this name, and if so does it match"? C++ templates show their
  origins as a very-weakly typed macro system here, because a template (until
  C++20) lacks type information about its generic parameters and can only check
  at the time of definition that "something" with the name exists, but might fail
  to resolve it later on at time of use - essentially exactly the same weakness
  that a generic function implemented as a macro would exhibit.
  
  - C++20 does improve this with concepts, but they primarily work as a requirement
    check and do not really provide a _type_ to a generic parameter that can
    resolve overloads before instantiation of the tempalte at a later point.
    While they make it easier to express that some group of operations exists,
    they do not provide a container for the group which still has to be resolved
    at instantiation-time, and do not remove the requirement to _instantiate_ a
    template (i.e. template functions defined with concepts still exist in
    "macro space" as opposed to having a single identity in program space).

While overload resolution is not by itself especially complicated to _implement_
(based on our experiences implementing it in both the Helix QAC C and C++ compilers),
as there is really only implementation in two places - resolution at call-points,
and some kind of naming scheme for the internal representation of the definitions -
it has complex and confusing rules and interactions on the user-facing side.

The introduction of templates to the C++ language is itself arguably a design
choice forced by this model of overloading, because by not binding overloads to
some kind of type-based container, generic functions have _no choice_ but to
require separate instantiation so that they can create different body definitions
with the different calls resolved.

## Type classes

Type classes, traits, etc. originate from an inversion of the problem from a user
perspective, _starting_ with the generic function use case rather than treating it
as an effective afterthought.

Starting with the premise that functions can be truly polymorphic:

    val head : 'a list -> 'a

(SML syntax) `head` is some function that operates on a list of objects all of
the same type. It doesn't care what type that is, but is able to guarantee that
given a nonempty list of objects it will return an object with the same type
as the element type of the list.

Despite having a polymorphic type, this function can still be used as a value
because it doesn't depend on any further instantiation that would create more
function body definitions. The type system is able to safely guarantee that
what in C would probably be conversions to and from `void *` are correct.
This kind of checking is [also expressible in C][12] with some attribute-based
extensions to the type system that show how pointer identity is not affected.

Because the type of the elements is abstract, if we want to perform an operation
on any of the values _in_ the container, the only way to ensure it is correct
(and actually does something) is to pass it in the parameters as well:

    val fold : ('a * 'a -> 'a) * 'a list -> 'a

This signature guarantees that some operation exists which can combine values
in the list, without knowing what it is, by using the common type variable in
the signature for both the operation and the container.

In `void`-_which_binds_ extended C syntax, this might look like:

    void fold (size_t N
        , Void(A) const in[N], Void(A) * out
        , void (*op) (Void(A) const *, Void(A) *));

which illustrates that the strong types are erased to `void *` and do not produce
separate instances of the same function body.

If this first parameter is a structure (a "dict") containing a group of
operations, instead of just one, we have implemented [dictionary-passing style][3]:

    val sum : AddPack 'a * 'a list -> 'a

Now the signature allows for a group of operations to be passed in that provide
sum expressions. The operations in the pack are on some type and the container
is also of objects of that type, as is the result.

Typeclasses generalize this a bit further by not only being able to say "this is
some type, and this other type is the same as it", but grouping "some type" by 
supported operations. For instance, `int`, `float`, `_Decimal32` all support the 
arithmetic operators,  so we can say that all three of those types are _instances_
of a class `Arithmetic`. They're also all instances of `Addable`, which includes
complete pointers, but `Arithmetic` doesn't include pointers because they don't
support stuff like division. This can be communicated by shifting the notation
slightly:

    val sum : (Addable 'a) => 'a list -> 'a

(evil hybrid of SML and Haskell syntax) Because we know that the type `'a` is the
same in both the concrete `list` and the concrete `Addable`, the `Addable` can
be deduced. Since there is a rule that there is only one object-instance of each
complete dictionary type, we don't need to pass the value of `Addable 'a` at the
call site because the singleton can be looked up by the compiler (either
concretely, at the outermost call site, or within a polymorphic function, by
continuing to pass the same argument through).

Within the body of `sum` there will be some invocation of `addPack.+` on the
dictionary argument that provides the concrete instance of the class used for the
overloaded addition operators. The language syntax will allow the `addPack.`
part to be implicit because the `+` is associated with the class `Addable` and
_no other_ type class - if `+` is available, it is as part of an instance of
`Addable`, and also `-` will definitely be available with the same signature.
In constrast to the open set where a template doesn't know that `+` _is_
available until the point of instantiation, and even if augmented with a concept,
can only ask that `-` exists as well but not that it is specifically bound to `+`
in a definition block.

As a result, the generic case does not rely on pure
syntactic (macro-like) lookup and instead allows generic functions to have a
unique, addressable/value identity (i.e. "templates") without requiring separate
instantiation for each concrete type. Monomorphisation remains available as an
optimization pass but is not exposed to the user at all by language semantics -
instead it is just a form of inlining, not observable in the program itself.

The use of overloading in languages with typeclasses _outside_ generic functions
is therefore just a special case where the type variables are already all filled
in and lookup is immediate. To make a flexible overloaded `add` function that
works with multiple types, we could, for instance:

    val add : (Addable 'a) => ('a * 'a) -> 'a

Instead of calls to the `add` operator building a candidate set and working out
the best matches to work out "which" `add` is being called here, we define that
there is actually just one addition function. The types of its arguments are
first checked to be the same, and then used to find the singleton dictionary
with the appropriate `addpack.+` implementation.

At runtime this will monomorphise to the same static direct call to some
mangled-named operator implentation that the open-set resolution would have done,
simply by regular function inlining (recognising that the implicit dict argument
has a `constexpr` value).

## Contrasting the approaches

The key difference in _application_ of these two approaches is that there is an
inverted order of operations:

- in candidate resolution for the open-set approach, every identifier used as an
  operator position names an "overload set" (not an element, the set), which is
  the group of "all functions named `op` in scope". The whole set gets pulled in
  at a call-point, and then a best match is found based on the shortest sequence
  of valid argument conversions.
  
  In other words, _first_ the whole set is visible, and _then_ conversions are
  applied and ranked to determine a best match. Notably, there can be multiple
  "valid" matches with different conversion distances from the syntactic call,
  and arbitrary elements can be added to the set (i.e. it is possible to add
  overloads not only for `f(int, int)` and `f(float, float)`, but also for
  `f(int, float)`, `f(float, int)`, etc., unrestricted).
  
- in a typeclass system, the use of the overloaded operator identifier tells the
  compiler, "the arguments _shall be_ of a type instantiating `Class`". Then
  instead of looking at the arguments for a best-match conversion path, it goes
  to the list of instantiations of `Class` and finds the specific entry for `T`.
  
  So an expression `int + int` tells the compiler to go fetch `Arithmetic(int)`
  specifically, and then retrieve `+` from that, _not_ to do the best-match
  search over everything visible in scope named `+`.
  
This is of particular benefit to C because we already have the Usual Arithmetic
Conversions in the language, and open-set operators have to build the list of
viable candidates _before_ applying arithmetic conversions (or promotions); if
we use a typeclass system, we can define that promotions and UAC happen _before_
dictionary instance lookup, drastically simplifying the rules and completely
eliminating the process of finding a shortest/best conversion "path".

Given an expression

    int x = ...
    float y = ...
    
    add (x, y); // definitely calls add(float, float)

We can define that the UAC happens to the operands, which means that `T` is
always the converted type: it is not possible to define an overload for `(int, float)`
and we do not need to even consider logical paths that would produce it.
Additionally, if `Addable(float)` is not visible in scope, we do not fall back
to a second-best match like `Addable(double)` or `Addable(int)` - this is just
a type error: the class instance required _is_ `Addable(float)`, not a nebulous
"viable candidate".

This _drastically_ simplifies use of overloading as a feature from a user
perspective.

Class-instance selection effectively already exists in the language in the form
of the `_Generic` operator, which takes a type argument (as an expression, but
the value is irrelevant) and converts it via a chain of _concrete_ generic-associations
into an expression. This is well-suited to returning singleton values as the
operator is a syntactic construct and therefore already returns a singleton
(the product of the operator is the _expression_, which is then evaluated).

## Modelled in C23

Since it is, verbosely, possible to express limited parametric polymorphism in
C23 (it is possible to 
[emulate a limited subset of VWB](https://gist.github.com/AlexCeleste/ad2db35940a77d7dc8575cf9744402f9#file-poly2_const-c)
using `typeof`); and we
have the class-instance selection mechanism in the language in the form of
`_Generic`; we can wrap these tools up in macros and provide a demonstrative
example of typeclass-based overloading using existing C23 features.

Declaring a type for the typeclass is enough to enforce that definitions of the
opereators are provided together, because it wants to be initialized (we could
potentially demand a dedicated syntax, but it isn't necessary). The names of the
operator _definition_ functions don't even matter and can be anything so long
as they have the right type to initialize the right slot in the class instance:

    typedef void (*BinOp) (void * out, void const * lhs, void const * rhs);
    
    struct SumInst {
      BinOp add;
      BinOp sub;
    };
    
    void addInt (void * out, void const * lhs, void const * rhs) {
      *(int *)out = *(int const *)lhs + *(int const *) rhs;
    }
    void subInt (void * out, void const * lhs, void const * rhs) {
      *(int *)out = *(int const *)lhs + *(int const *) rhs;
    }
    
    struct SumInst intInst = {
      addInt, subInt
    };

[and again for `float`](https://gist.github.com/AlexCeleste/76c1f62321ae21d3c1304052fe6c4830#file-explicit-dict-c).

A `use` function can accept the dictionary instances explicitly:

    void useAdd (struct SumInst const sumInst, void * out, void const * lhs, void const * rhs)
    {
      sumInst.add (out, lhs, rhs);
    }

and the first evolution of "generic" addition according to [the okmij hierarchy][3]
starts to look like:

    void call (int x, float y)
    {
      int ri;
      float rf;
    
      useAdd (intInst, &ri, ToLvalue (x), ToLvalue (x));
      useAdd (fltInst, &rf, ToLvalue (y), ToLvalue (y));
    
      printf ("%d %f\n", ri, rf);  // 12 25.00
    }

Though this isn't really "generic" yet, because explicit passing means manually
providing the type resolutions in source anyway.

With `_Generic` is is [possible to define instance-selection](https://gist.github.com/AlexCeleste/76c1f62321ae21d3c1304052fe6c4830#file-selected-dict-c)
which forces the emulation to rely on singletons, and allows them to be identified
by operand type:

    #include "oper_counter.h"
    DefInstance (Sum, float, {
      addFloat, subFloat
    });
    
    ...
    
    Sum intInst = Instance (Sum, int);
    Sum fltInst = Instance (Sum, float);
    
    useAdd (intInst, &ri, ToLvalue (x), ToLvalue (x));
    useAdd (fltInst, &rf, ToLvalue (y), ToLvalue (y));

From here it is [an obvious step](https://gist.github.com/AlexCeleste/76c1f62321ae21d3c1304052fe6c4830#file-selected-dict-c)
that the `Instance` resolution can be inlined into the call, and also use 
`typeof (arg-expression)` instead of a fixed type name as its operand, which
unlocks implicit instance resolution:

    void useAdd (Sum const sum, void * out, void const * lhs, void const * rhs)
    {
      sum.add (out, lhs, rhs);
    }
    
    // clearer with statement expressions but not required (see link)
    #define gAdd2(L, R) ({                            \
        typeof (L) _ret;  /* L+R also an option */    \
        useAdd (Instance (Sum, typeof(_ret))          \
              , &(_ret), ToLvalue (L), ToLvalue (R)); \
        _ret;                                         \
      })

used as

    void call (int x, float y)
    {
      int ri = gAdd2 (x, x);
      float rf = gAdd2 (y, y);
    
      printf ("%d %f\n", ri, rf);  // 16 29.00
    }
    
    call (8, 14.5);

This unlocks overloaded, generic addition, with argument promotion according to
existing arithmetic conversion rules, and requiring coherent complete sets of
operators with related features. It also shows that the `add` function itself
(`useAdd`, _not_ `gAdd`) has an addressable identity despite being polymorphic.

[More types can be added:](https://gist.github.com/AlexCeleste/76c1f62321ae21d3c1304052fe6c4830#file-more-types-c)

    typedef struct Fixie {
      int rep;
    } Fixie;
    
    void addFix (void * out, void const * lhs, void const * rhs) {
      ((Fixie *)out)->rep = ((Fixie const *)lhs)->rep + ((Fixie const *)rhs)->rep;
    }
    void subFix (void * out, void const * lhs, void const * rhs) {
      ((Fixie *)out)->rep = ((Fixie const *)lhs)->rep - ((Fixie const *)rhs)->rep;
    }
    
    #include "type_counter.h"
    ADD_TYPEID (Fixie);
    #include "oper_counter.h"
    DefInstance (Sum, Fixie, {
      addFix, subFix
    });

and work with the existing definition of `useAdd`/`gAdd2` unaltered:

    void call (int x, float y, Fixie z)
    {
      ...
      Fixie rF = gAdd2 (z, z);
      printf ("%d\n", rF.rep);  // 22
    }
    
    call (6, 12.5, (Fixie){ 11 });

### Conclusion

This syntax is not suitable for long-term use and the point is not to propose
that this method for implementing generic functions be copied directly.
Instead, the point is to demonstrate that there is an immediate way to model
typeclass/trait-style overloading in C23, and therefore that this approach
to both generic function definitions and overloaded operators is a more natural
fit for the language, because all of the rules fundamentally already exist,
than open-set overloading, which requires new machinery in the compiler,
(potentially) new machinery in the ABI, and a slew of complicated rules for the
user to remember which are easy to break or use in much more confusing ways.

Open-set overloading also _forces_ the language down the template/"macro space"
path for definition of generic functions, and cannot be taken advantage of at
all by true polymorphic functions. We should avoid this design mis-step at the
earliest opportunity and adopt a model that seamlessly integrates with generic
function definitions so that C can have provide fully monomorphic implementations
which suit the character and spirit of the language, with no out-of-line
instantiations; and which enables significantly strong type checking and earlier
error messaging than the very delayed approach enforced by templates.

## References

[Overloading in C++][1]  
[Overloading in Clang C][2]  
[Understanding type classes][3]  
[Java Language Specification][4]  
[Rust Traits][5]  
[Haskell Typeclasses][6]  
[C++ Concepts][7]  
[CLOS in Common Lisp][8]  
[COS][9]  
[N3051 Operator overloading in C][10]  
[Typeclasses in C23][11]  
[`void`-_which-binds_][12]  

[1]: https://eel.is/c++draft/over.match
[2]: https://clang.llvm.org/docs/AttributeReference.html#overloadable
[3]: https://okmij.org/ftp/Computation/typeclass.html
[4]: https://docs.oracle.com/javase/specs/jls/se18/jls18.pdf
[5]: https://doc.rust-lang.org/book/ch10-02-traits.html
[6]: https://www.haskell.org/onlinereport/haskell2010/haskellch4.html#x10-750004.3
[7]: https://eel.is/c++draft/temp.concept
[8]: https://en.wikipedia.org/wiki/Common_Lisp_Object_System
[9]: https://ldeniau.web.cern.ch/ldeniau/html/cos-oopsla09-draft.pdf
[10]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3051.pdf
[11]: https://gist.github.com/AlexCeleste/76c1f62321ae21d3c1304052fe6c4830
[12]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2853.pdf
