    Proposal for C2y
    WG14 3317

    Title:               Essential Effects for C
    Author, affiliation: Alex Celeste, Perforce
    Date:                2024-08-19
    Proposal category:   New feature, discussion
    Target audience:     Compiler implementers, users

## Abstract

In this discussion proposal we suggest that C would be improved by extending the
core type system into a _type and effect_ system, that forces functions and
blocks to declare which classes of effects will result from their evaluation.
We show how this can be used to improve composability in metaprogramming, and
can be used to define MISRA-style restrictions more easily through signatures
expressed in the core language. The extended notion of type allows us to model
C's primitive language constructs as operators with a checkable signature.

We only propose _effect tracking_ and _effect checking_, and not to introduce
dynamic _effect handling_ at this time.

-------------------------------------------------------------------------------

# Essential Effects for C

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3317
    Revises:      N/A
    Date:         2024-08-19

## Summary of Changes

### N3317
 - original proposal to WG14, but a previous version of this document was submitted
   to the MISRA C WG's document log.

## Introduction

A formalized _effect system_ keeps track of the side effects of expressions and
statements in a program at a finer level of granularity than usually expressible
in C.

An _effect system_ is usually combined with a type system, so that the side effects
potentially involved in the production of a value are tracked, alongside the type
of the result value. Effect systems may be either fully static, or partially
value-dependent.

A conventional notation for the _type-and-effect_ of a value-producing expression
is:

    expr: type ! { effect } region

where `expr` is the object or expression being typed, `type` is its in-language
or dialect type (such as the C Standard language type or the MISRA essential type),
`{ effect }` is a union of all effects triggered by the evaluation of `expr`, and
`region` is the name of a category of objects or memory to which those effects
may apply.

By encoding the actions available to a function into the type system as effects,
it is possible to statically ensure that specific effects do or do not occur in
certain places, that callbacks or library functions are safe to use for sensitive
tasks, and that effects that do occur are _handled_ so that they do not escape
to the wider calling context (which can even imply static verification of resource
management).

Effect systems generalize the C23 features of `[[reproducible]]` and `[[unsequenced]]`
attributes for functions, which guarantee that the annotated functions are free
of certain effect classes.

## Proposed application

The proposal is that every statement, expression, subexpression and complete
function body in a TU has a _type-and-effect_. For the purposes of the
combined _type-and-effect_, the type component of the _type-and-effect_ of a
non-expression statement is defined as `void`.

The effect of an expression or statement is the (counted) union of the effects
of *all* subexpressions syntactically included in the construct. Therefore, this
proposal is for a complete static, non-value-dependent effect system that does
not consider reachability or symmetric operations.

Objects do not have an effect, only a type, though their initialization
expression will have a (possibly-empty) effect.

Some operations introduce a new explicit effect by themselves. For instance, the
assignment operator `=` introduces a write effect.

When the effects of an operation and its operands are combined, the result effect
counts the number of sub-effects as well as their classes. For instance, a binary
operation with a write effect in both the lhs and rhs operands contains *two*
write effects in addition to any others that may be introduced.

Explicit regions are not currently proposed. Instead, the model proposes that
all effects have an implicitly global region. Finer granularity is achieved by
splitting the effect classes themselves (i.e. separation of local and non-local
writes into different effects).

Note that an expression with no traced effects is a distinct concept from a
constant expression, as it may still involve "pure" reads from non-constant
objects, which are not recorded. This is captured by the `[[reproducible]]`
attribute added to C23.

### List of proposed effects

The proposed "essential" effects are broken into three major groups:

 - Local:
   - `mut`: local write or assignment, countable
   - `vol`: volatile access (should this be persistent?), countable

 - Persistent:
   - `write`: non-local write or assignment, countable, implies `mut`
   - `mem`: memory manipulation (allocation or free)
   - `lock`: exclusive lock acquire or release
   - `file`: external resource manipulation
   - `errno`: sets `errno` (or other named out-of-band flags?), implies `write`
   - `system`: hosted os state manipulation (? out of scope? usefully distinct from `file`?)

 - Control:
   - `jump`: non-local control transfer (signal, `noreturn` call, `longjmp`, panic)

 - Any:
   - the `wild` effect to explicitly indicate unknown / any action
   - the `none` effect is a name for the empty set of effects

Regions _could_ also be used to express parts of this hierarchy: for example,
`mem` and `file` could be viewed as applications of a single `resource` effect
over two regions indicating internal-to-program and external-to-program resources.
Similarly, `mut` and `write` could be viewed as two separate applications of a
single assignment effect to local and non-local object regions. However, it is
proposed that splitting the effects as a whole is simpler than using explicit
regions.

**Question:** which other explicit effect classes would be useful?
e.g. thread creation?

**Question:** are there other useful major groups?

**Question:** are there any effect classes other than `write` for which we might
prefer regions to allow combined counting? (in the current proposal we need
these classes to be separate so that `mut` can be discarded, see below)

Segfaults, floating point exceptions, non-volatile reads, etc. may have
observable effects within the program but are not proposed as useful to track
on this level.

A concrete notation for annotating effects is not part of this proposal.
A proposal for notation in attributes will follow if the core concept is adopted.
Comment-based pseudocode is used in this document.

### Propagation

Within a scope block, _all_ effects of subexpressions and nested statements
_unconditionally_ propagate to their containing expressions and statements.
For example:

    extern int x;
    void f1 (int k) { // TE: void ! write
    
      if (0) {   // TE: void ! write
        x = k;   // TE: int ! { mut | write }
      }
    
      volatile j = 0;
      int y =      // TE: int ! { mut | vol }
        (x == 1    // TE: int ! none
        ? k = 1    // TE: int ! mut
        : j > 0);  // TE: int ! vol
    }

At the function boundary, effects in the Local group are discarded, as this group
signifies effects that are primarily of interest w.r.t correct ordering.

As shown above, some operations can introduce two effects at once: a write to a
non-automatic _lvalue_ introduces both the `mut` effect (to count assignments)
and the `write` effect to the containing function. Similarly, a write to a
`volatile`-qualified _lvalue_ would introduce both the `mut` and `vol` effects
to the result expression. A write to a non-local `volatile`-qualified _lvalue_
would introduce all three in a single operation.

When two effects are combined that contain the same class, if the effect class
is "countable", the number of occurrences in each sub-effect are summed:

    void f2 (int k, int j) {
      int x = 0;   // TE: int ! none
    
      x = k;       // TE: int ! mut
    
      x = (        // TE: int ! mut(2)
        j = k      // TE: int ! mut
      );
    
      x = (        // TE: int ! mut(2)
        k = 1,     // TE: int ! mut
        j = 2      // TE: int ! mut
      );
    
      x = (             // TE: int ! mut(2)
        k == 1 ? j = 0  // TE: int ! mut
               : j = 2  // TE: int ! mut
      );
    
      (x = 1) && (j = 1);  // TE: int ! mut(2)
      (x = 0) && (j = 1);  // TE: int ! mut(2) // (unconditionally)
    
      volatile a = 0;
      x = false    // TE: int ! vol
        ? a > 1    // TE: int ! vol
        : j > 1;   // TE: int ! none
    
      x = false    // TE: int ! vol(2) // (unconditionally)
        ? a > 1    // TE: int ! vol
        : a < 1;   // TE: int ! vol
    }

This illustrates some asymmetries between effect and type, for instance, we
introduce a concept of _effect magnitude_, and the effect of a subexpression is
**not lost** even when its value (and therefore its type) is discarded or converted
in such a way that it does not contribute to the expression's result value.

The summing is one-directional - this mechanism would not allow resource-class
effects to "cancel out" (which is why they are not marked as countable above):

    void f3 (int sz) {         // TE: void ! mem
      void * p = malloc (sz);  // TE: void * ! mem
      free (p);                // TE: void ! mem
    }

The above function is not "pure", and by not allowing effects to subtract, this
is communicated. A memory operation happened, and then another memory operation
happened - correct resource handling is a separate concern from immediate effect
tracking, i.e. even though the function correctly disposed of the resource, the
effect communicates that it relies on a working `malloc` implementation.

The effect of an external function whose body is not visible, and whose effects
are not otherwise specified to the system, is assumed to be a union of all effects
listed in the Persistent and Control groups:

    extern int action (int);
    void f4 (int k) {
      int x = action (k);  // TE: int ! { write | mem | lock | file | errno | jump } AKA int ! wild
    }

The `wild` effect signifies that an action is a total unknown.

Note: resource matching *could* be communicated by an effect system, but it would
need to be both dependent and heavily region-annotated (as found in [Deputy for C][5]).
A simplified version that simply tried to "decrement `mem` on `free`" would
not be able to statically prove that the right resources are released, only
that _some_ resource was released.

Therefore this method of effect combination/propagation is completely flow-
independent, including in the case of completely linear control flow.

An effect with more classes or a higher count for a given class is _stronger_
than an effect with fewer classes or a lower count. Analogous to qualification,
every effect class within the effect must be stronger for the whole effect to
be considered stronger. An effect can be implicitly converted to a strictly
stronger effect (see below), so for instance `mut` can convert to `{ mut | vol }`;
like with qualifiers, if the target effect is not _strictly_ stronger the
effect cannot be converted (`mut` cannot convert to `vol` in the same way that
`T const *` cannot convert to `T volatile *` - implicitly adding the `vol` or
`volatile` qualifier is fine, but losing the `mut` or `const` is not).

### Effect signatures vs type signatures

An effect signature is similar in concept to a type signature but the resulting
effect does not describe quite the same thing.

Roughly, a function signature describes a mapping from _input types_ to an
_output type_:

    add :: (num, num) -> num

It can be polymorphic or dependent, in which case when applied the output is
affected by the input:

    map :: (('a -> 'b) , ['a]) -> ['b]
    alloca :: (int 'A) -> int(*)['A]

Because effects describe _actions_ rather than _values_, an effect signature for
any given entity class is similar to the type signature for the entity class
one level "down":

 - `void`
 - objects and values have a value type, and an effect of `none` because they
   don't "do" anything
 - first-order functions have a type that maps an input value to an output
   value, and an atomic effect because they only have one "action" in the body;
   the arguments are _values_, and therefore don't contribute to the effect at all
 - higher-order functions have a type that maps a function type to value types;
   if they run the function, its effects form part of the result effect, so
   these have an effect that resembles a signature
 - built-in language constructs like the operators, `if`, etc. have an effect
   signature that resembles the type signature of a higher-order function,
   because they accept one or more _fragments_, with an effect, as their
   operands, and execute them somehow to produce a combined effect

Therefore, the effect of most functions is "value-like" in the sense of being
shaped like the _type_ of an object:

    extern int x;
    
    // foo :: (int) -> int ! write
    int foo (int y) { return (x += y); }  // type is (int) -> int
                                          // effect is !write - NO arrow

If `foo` is called like:

    foo (++x);      // TE: void ! write
    foo (--local);  // TE: void ! { write | mut }

the effects of the arguments don't contribute to the effect of `foo`. They
instead contribute to the effect of the `()` operator, which is polymorphic.
In the effect system, `foo` has a signature "like" a value and `()` and other
operators have signatures "like" functions.

All of this is important in two ways, one of which is that we need to distinguish
two different kinds of effect annotations:

 - descriptive signatures, which capture the entire polymorphic effect of an entity
 - effect checks, which do not describe the _resulting_ effect but instead place a
   constraint on the inputs and outputs

### Effect checking

> If we want to restrict which effects can _appear in_ the arguments to a function,
> or operands of an infix operator, this is a different annotation (at least in any
> readable format) from describing the full polymorphic signature of the operator;
> so we need a second kind of signature. In an _effect check_ signature, the
> effects specified in the parameter and result positions designate a "maximum"
> effect rather than a concrete effect:
> 
>     // op :> (int ! mut(1), int ! mut(1)) -> int ! mut(1)
> 
> This is the _effect check_ signature for an operator that specifies that at most
> one `mut` effect may occur as a result of evaluation: the effect of the left
> operand will match `mut(1)` if it is `none`, because of the rule that an effect
> is implicitly convertible to a stronger effect. This also specifies `mut(1)`
> for the result check: this is also a maximum strength rather than a specification
> of what the result _will_ be, that checks the actual result effect and specifies
> it _must be_ no stronger than `mut(1)`, not that it actually is `mut(1)`. (It
> relies on the separate definition of the polymorphic effect of the operator as
> composing its left and right effects.)

Therefore, an operand checks that its _type-and-effect_ has _no_ effects that are
_not_ specified for that operand, or if a magnitude is specified, not more than
that magnitude. If an expression contains an effect class that is unspecified, or
a specified effect class but with a magnitude higher than one explicitly-specified,
a _constraint_ is violated. (Generally we only care about a magnitude of 1, for
`mut`/`vol`.)

The result effect is the propagated effect of the operands plus any other effect
the operation is specified as introducing in its descriptive signature.

For instance, we might propose that arithmetic operations may only evaluate
pure values and sequence-independent volatile accesses:

    int a, b;
    volatile c, d;
    int e (int x, int y) { return x + y; }
    int f (void);
    
    // + :> (int ! vol(1), int ! vol(1)) -> int ! vol(1)
    
    a + b;        // OK, int ! none
    a + c;        // OK, int ! vol(1)
    c + d;        // Error: int ! vol(2)
    a + e(a, b);  // OK, int ! none
    a + e(c, b);  // OK, int ! vol(1)
    d + e(c, b);  // Error: int ! vol(2)
    a + f();      // Error: int ! Persistent

The effect annotation allows a _maximum_ `vol` of 1 in either operand of `+`, but
also allows a maximum `vol` of 1 for the entire expression, which means that
at most one operand can have the `vol` effect at all. (It doesn't specify that
the result _has_ `vol(1)` if neither operand does.)

(note: not actually proposing this restriction!)

We could also provide specified effect checks for known functions, such
as `malloc` - say we wanted to ensure that the size property was always pure
(again, not proposing we do so), as well as refining the effect on the (system,
opaque) `malloc` implementation to just `mem`:

    // malloc :> (int ! none) -> mem
    
    void * p = NULL;
    p = malloc (6);                      // OK, parameter is int ! none
    p = malloc (d);                      // Error: parameter is int ! vol
    p = malloc (sizeof (int) * e(a, b)); // OK
    p = malloc (sizeof (int) * (a = b)); // Error: int ! mut
    p = malloc (f ());                   // Error: int ! Persistent

### Proposed recommended practice

With the above, we can provide a table of recommended practices, analogous
to the operator table for MISRA's _essential types_.

**Question:** what are the permitted effects for each category of operator?

We can generalize effect rules across expression statements and controlling
expressions so that we do not need special-case rules for where certain operations
can appear.

Proposal: every expression statement, operand, and controlling expression should
be _recommended_ to the effect magnitudes 1 for the `mut` and `vol` classes
(i.e. the effect signature of every expression should either not include `mut`
or `vol`, or only include them as `mut(1)` or `vol(1)` respectively). This
removes the need to ensure that assignments, modifications and volatile
accesses are fully-sequenced (even if we fix evaluation order), making them
much easier to read and reason about.

Proposal: initializers, function calls, and arithmetic operators should not
include the `mut` effect class in the the _type-and-effect_ of their operands,
with a recommendation to diagnose:

    // +, *, - etc. :> (int ! vol(1)
    //                , int ! vol(1)) -> int ! vol(1)

Or alternatively, to ensure readability and understand-ability without needing
to rely on strict order of evaluation, allow only one `mut` so that its
order is less important:

    // +, *, - etc. :> (int ! { mut(1), vol(1) }
    //                , int ! { mut(1), vol(1) }) -> int ! { mut(1), vol(1) }

For control structures, specifying an effect check for the operand expressions
is analogous to specifying a type for a function's parameters and would indicate
a maximum recommended effect (not the same as the "operator signature" for the
control structure itself, see below):

    // for-clause-1 :> T ! { Local | Persistent }
    // for-clause-2 :> T ! vol(1)
    // for-clause-3 :> T ! { Persistent | Local(1) }
    
    // alternatively
    // for :> (!{ Local | Persistent }  // ignores type component
             , !vol(1)
             , !{ Persistent | Local(1) }
             , !wild) // effect of the secondary-block is part of the signature!
           -> void ! wild

Recommendations like these incorporates some notions about readability. Other
generalizations are possible.

Specifying _type-and-effect_ checks for various functions (library and
standard) allows a finer grain on restricting permitted use patterns and may be
useful to ensure arguments are consistent or results are not used improperly.

### Polymorphic effect signatures

In contrast to a check which imposes constraints on operands and result, a
signature describes what the effect of an entity _is_ directly. This is where
a distinction creeps in between C functions, which will mostly have an atomic
effect, and C operators, which are polymorphic because an operator is a higher
order entity:

    // add :: int ! none
    int add (int x, int y) { return x + y; }
    
    add ((ext = v), ++ errno); // add still has effect none
                               // it's operator() which has to deal with the nonsense

In contrast to `add`, which is a function from two _values_ to a result value,
an operator like `+` takes two _expressions_ to produce a combined _expression_
i.e. it works on program fragments. This is therefore more analogous to a
higher-order function:

    (ext = v) + (++ errno);
    
    // is more like
    add2([]{ ext = v; }, []{ ++ errno; });
    
    // add2 :: (int ! 'a, int ! 'b) -> int ! { 'a | 'b }
    // + :: (T ! 'lhs, T ! 'rhs) -> T ! { 'lhs | 'rhs }

(This shows that a useful syntax for generalized operators needs to be able to
express dependence on the effects of the operands through _effect variables_)

Note that all binary operators in C have a very similar effect signature (the
difference is _only_ in the type part):

    // && :: (T ! { 'lhs }, T ! { 'rhs }) -> int ! { 'lhs | 'rhs }

The effects of the right hand operand are part of the signature regardless of
whether it is evaluated or short-circuited.

The assignment and unary increment operators can be specified as introducing
the `mut` effect class:

    // =, +=, etc. :: (T ! 'lhs, T ! 'rhs) -> T ! { 'lhs | 'rhs | mut }
    // ++, -- :: (int ! 'a) -> int ! { 'a | mut }

Finally, this allows us to define type-and-effect signatures for the language's
primitive control operators and model them as operators as well:

    // if :: (int ! 'cond) -> (void ! 'blk) -> void ! { 'cond | 'blk }

Although not useful by itself, this then makes it easier to define control
operators in terms of the existing operators with  more refined effect types:

    // my_if :: (int ! Local) -> (void ! Local) -> void ! Local
    #define my_if(COND) if (COND) /* */

The above macro defines a new control structure `my_if` which is typed as only
permitting local (`mut` and `vol`, uncounted) effects within its condition and
its secondary-block. The signature defines the control structure a a curried
operator, which takes a single expression argument and then the result of
that operation is a second operator which takes a _secondary-block_ as its
operand:

    my_if (a == b) { something (); }
    
    my_if (a == b)           // <- application 1
          { something (); }  // <- application 2

This is distinct from the signature

    // my_if2 :: (int ! Local, void ! Local) -> void ! Local
    #define my_if2(COND, STMT) if (COND) { STMT }

which has different compositional properties; we can imagine a higher-order
control structure that would accept the result of the first application of
`my_if`:

    // my_control :: ((void ! Local) -> void ! Local  // <- signature of `my_if(a == b)`
    //              , int ! None
    //              , int ! None) -> void ! Local
    #define my_control(OP, X, Y) do { OP { work (X, Y); } } while (0)

`my_control` treats `OP` as an operator and gives it the "operand" of `{ work (X, Y); }`.

This allows us to make control fragments composable in (the beginnings of) a typed
fashion, with many potentially useful applications.

## Prior Art

Effect systems generalize a concept which is repeatedly reinvented, e.g. as
"[cone of evaluation][4]", or even `constexpr`, and which could benefit by being
integrated into the type system of the base language instead of existing
implicitly as an ad-hoc.

Effect systems are deployed in a number of languages to great effect. The
guarantee from a function's type that it will or will not behave in a certain
way, as a binary flag, is well-attested in every language from Java (as checked
exceptions) through to systems based on algebraic handlers like Koka.
A number of projects have also added effect systems to C such as [Deputy][5],
though these are generally far more complex than the proposal here.
In languages with richer type systems like OCaml, it is possible to
[encode effects and their handlers][7] using existing language features as a
library feature, which is simpler overall that introducing a parallel type
system; the language infrastructure to do this in C simply isn't there, though.
An alternative approach would be to build towards having a type system rich
enough to encode effects in a similar way.

An effect system is implemented in QAC and is used in several analyses. The
system we use is mostly additive, as for resource tracking we prefer to rely
on full flow analysis; however, it is capable of the operations described
above to both count and cancel-out, and thus _handle_ effects "algebraically"
as well.

## Impact

At the moment we propose that this feature should be provided through attributes,
and therefore any implementation that does not wish to implement effect checking
is free to simply not do so.

Actually implementing effect tracking and checking is not onerous, but adding
the attribute notation to the parser may represent a meaningful amount of
effort for some tools.

## Future directions

A concrete notation for effect annotation based on attributes can be specified
either resembling the notation in the examples above, or with a different
appearance.

This proposal composes well with [N3316 The `void`-_which-binds_][6], in that
the same annotation system used to indicate that functions are polymorphic
over types can be reused to indicate polymorphism over effects. [Effect polymorphism][3]
is essential for giving higher-order functions useful effect signatures without
blowing up the signature or having every complex operation fall back to `wild`.

An advantage of using attributes to declare effect signatures is that if attribute
syntax is integrated into the preprocessor, it will become possible to give macros
effect signatures and, in particular, to give function-like macros effect signatures
on each parameter:

    // safe to repeat operands in the definition,
    // because they aren't allowed to be non-repeatable!
    #define MAX(X [[effect(none)]], Y [[effect(none)]]) ((X) > (Y) ? (X) : (Y))
    
    // macro that accepts a statement operand
    // which operates on files but not on memory
    #define FILE_OP(STMT [[effect(file)]]) do { STMT } while (0)

We can later generalize this into a description of the C control structures as
analogous to higher-order operators accepting _actions_ as their operands,
which has many useful properties for metaprogramming and program construction
and allows us to define "types" for what are currently language fragments like
`do` or `if`. This makes development of staged programming environments that
embed C a much more consistent and attractive proposition and has many
interesting future applications in offloading more advanced type checking to
a first-stage metalanguage.

Generalized _effect handlers_ [as seen in Koka][8] are a powerful tool for
encapsulating work, but for the time being we only propose tracking-and-checking
as being more compatible with the aims and spirit of C, i.e. a C program should
prefer to disallow effects from occurring in specified regions _at all_ rather
than encapsulating them and having to provide failure strategies. An effect
handler in Koka, Ocaml, etc. is allowed to _not_ invoke the continuation, which
means that it needs to be able to compile as a jump-out; this would risk
introducing more opportunities for poor structure and generally favours writing
new code rather than annotating existing code.

It may also be unnecessary if it becomes easier to use C as the embedded
language in a staged programming system, in which case the handlers can be
expressed as operators in the first-stage language and do not need to be included
as first-class C features.

## Proposed wording

No concrete wording is proposed at this time.

## Questions for WG14

Would WG14 like to formalize an effect system for C2y along the lines of the system
proposed by N3317?

Would WG14 like to codify the effect system in the core language instead of in the
attribute language?

## References

[C2y latest public draft][1]  
[Effect system][2]  
[Associated effects][3]  
[Contracts: Protecting The Protector][4]  
[Deputy: Dependent Types for Safe Systems Software][5]  
[The `void`-_which-binds_][6]  
[Effect handlers in OCaml][7]  
[The practice of effects: from exceptions to effect handlers][8]  

[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf
[2]: https://en.wikipedia.org/wiki/Effect_system
[3]: https://dl.acm.org/doi/pdf/10.1145/3656393
[4]: http://wg21.link/p3285
[5]: https://www.microsoft.com/en-us/research/video/deputy-dependent-types-for-safe-systems-software/
[6]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3316.html
[7]: https://ocaml.org/manual/5.2/effects.html
[8]: https://xavierleroy.org/CdF/2023-2024/5.pdf
