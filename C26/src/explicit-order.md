    Proposal for C2y
    WG14 3203

    Title:               Strict order of expression evaluation
    Author, affiliation: Alex Celeste, Perforce
    Date:                2023-12-14
    Proposal category:   Clarification/simplification
    Target audience:     All

## Abstract

Only certain operators and whole statements describe a sequence point in C; for
historical reasons, within an expression, the order of evaluation of operands
is assumed to be unspecified unless stated otherwise by the Standard.
We suggest that between language, hardware, and compiler advancements since the
original development of the language, this allowance is no longer needed and 
expression evaluation should be sequenced by default.

-------------------------------------------------------------------------------

# Strict order of expression evaluation

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3203
    Revises:      (n/a)
    Date:         2023-12-14

## Summary of Changes

### xxxx
 - ; other languages; typo/phrasing fixes

### N3203
 - original proposal
 
## Introduction

As of C23, the order of evaluation of expression operands is unsequenced by
default, and sequence points only apply at the statement or declaration break
or on a small number of specifically called-out operators. In particular, the
presence of a side effect _does not_ imply a sequence point. Therefore, in the
following example:

    int f1 (void) { printf ("f1\n"); return 1; }
    int f2 (void) { printf ("f2\n"); return 2; }
    
    int z = f1 () + f2 ();

although the final value of `z` is always 3, the side effects to reach that
state can occur in either order: the program can print either "f1", "f2" or
"f2", "f1"; and both are conforming.

In the following classic:

    int x = 1;
    int y = ++ x + ++ x;

...the value of `y` is a trick question, as the behaviour is undefined.
We posit that this fact serves nobody except smug 101-level professors.

Although only one of these constructs is undefined, MISRA and related
Guidelines (such as [CERT][3]) forbid both, because the unspecified behaviour
is almost as bad for actual users. Therefore, real-world code that cares at all
about side effects will always split an expression into multiple separate
expression statements assigning temporary values, unless it really is trivially
obvious that there is no overlap at all in the _domain_ of side effects of
separate operands.

To fix this, we propose that _every_ operator in the expression language should
be defined to introduce a sequence point, not just `&&` and friends.

### Background

The order of expression evaluation has been unspecified since K&R C
(7 "Expressions": "Otherwise the order of evaluation of expresisons is
undefined ... The order in which side effects take place is unspecified";
7.1 "Primary expressions": "take note that the various compilers differ").
The historically accepted reason for this is that since C was designed with
some concessions to portability, different calling and evaluation conventions
on the various target systems required operands or arguments to be pushed to
stack positions in forward or reverse order, and since the language always
aimed to allow for efficient output code, it could not require some arbitrary
subset of targets to perform "redundant" copies just to be able to push all
arguments in the platform correct order, wasting both space and operations.

We believe that this is an irrelevant concern as of 2023, when a compiler's
working memory no longer needs to be measured in kilobytes.

Firstly, the main case for reordering concerns expressions where the operands
are calls to functions that are not being used for side effects:

    int gen1 (int x, int y) { return x + y; }
    int gen2 (int x, int y) { return x - y; }
    
    z = gen1 (x, y) * gen2 (x, y);

In the above example, the ordering of the calls is unspecified and the compiler
can emit them in either order. But in this case, there is no side effect anyway
and any reordering doesn't matter. If the function definitions are visible, a
modern optimizing compiler can apply the as-if rule and call `gen2` first _even
if the Standard says_ that `gen1` is always ordered first, as there is no way
for the user to observe this fact.

If the definitions are not visible, as of C23 we are equipped with `[[unsequenced]]`
and can now annotate the function declarations:

    [[unsequenced]] int gen1 (int x, int y);
    [[unsequenced]] int gen2 (int x, int y);
    
    z = gen1 (x, y) * gen2 (x, y);

The compiler is still free to apply the as-if rule and the order of the calls
in the translated code still doesn't matter.

On the other hand, say the functions are defined elsewhere and the attribute
is not used to tell the compiler that their order doesn't matter:

    int gen1 (int x, int y);
    int gen2 (int x, int y);
    
    z = gen1 (x, y) * gen2 (x, y);

We do not believe it is actually correct in this case for the compiler to take
the liberty of assuming these calls can be emitted in either possible order.
If the compiler doesn't have enough information to _know_ whether there are any
observable side effects that would be affected by evaluating `gen2` first, it
also does not have enough information to validly say that reordering the calls
is a genuine optimization (either logically correct, or a local improvement).
The few cycles potentially gained locally are outweighed by an unknown number
on the other side of the call boundary; this "optimization" is likely 
irrelevant.

For function calls, the following was a concern for the compilers and machines
of the 1970s and 1980s:

    int foo (int x, int y, int z);
    
    int f1 (void);
    int f2 (void);
    int f3 (void);
    
    foo (f1 (), f2 (), f3 ());

The target machine might not have many resources for storing temporary results
(this part is still true depending on the target), and the compiling machine
had very limited resources available to rewrite an expression to assign to
storage according to register discipline, so stack discipline was used.

This is an implementation performance concern that has evaporated with the
passage of time - if a user cares _at all_ about the performance of the resulting
code, they are _not using_ a compiler that is not able to rewrite a whole call
expression in one go. If they are using a compiler so simple it can only handle
one subexpression at a time without knowledge of the rest of the expression, they
realistically do not care about the resulting performance. The permissiveness
that once allowed compilers to do the best job they can is simply no longer
relevant as any compiler advanced _enough_ to use for a performance case is now
so advanced it doesn't need this permission in order to emit the best possible
machine instructions.

## Impact

Words are cheap. What about timings?

### Measurements

Dos Reis/Sutter/Caves have kindly done this work already as part of a proposal
for C++17, which was partially adopted, in [p0145r3][2]. The problem is worse
in C++ because of operator overloading and member call syntax, which can appear
to give guarantees or implications of order that do not exist.

In Section 7, the authors describe an implementation experiment where Visual C++
was modified to create a worst-case combination:

- the compiler was forced to respect strict evaluation order for functions

- the optimizer was _not allowed_ to exploit this new guarantee

> We successfully built, installed, and booted the NT kernel. Then we built a
> large application code base, and ran “build, validation, test suites.” That
> uncovered sources of potential bugs due to non-portability assumptions: one
> real-world-code test failed, out of 26. Then, we compiled and ran Spec benchmarks.
> We found that some entries in the benchmark suite ran slower, others ran
> faster compared to the scenario where the evaluation of the argument list is
> left unspecified. **The variation is between -4% and +4%.** It is worth noting
> that these results are for the worst case scenario where the optimizers have
> not yet been updated to be aware of, and take advantage of the new evaluation
> rules and they are blindly forced to evaluate function calls from left to right.

(emphasis mine, differs from original)

Dos Reis _et al_ bring the receipts for the impact being essentially noise as
large changes balance out across the whole of a very large, very complex system.
Although the optimization characteristics change, they also balance out; and
if the compiler was allowed to use complete information there is a reasonable
case that it might actually do better here.

### Semantics

For all that, we do not consider the impact on performance to be instrumental.
As noted above, for C code, the considerations are different and in practice,
code that would be affected by evaluation order is generally required by 
Guidelines to be rewritten so that it is not, e.g. [CERT EXP30-C][3].

Therefore we consider the value of this proposed change to rest on the removal
of some undefined and unspecified behaviours. More code becomes conforming and
more expressions have clear effects. Where the effects don't matter, we have
new tools to take advantage of that explicitly.

The most important consideration is:

- no observable behaviour of a portable, strictly conforming program is affected
  by this proposal.
  
  - a strictly conforming program in C89 through C23 _remains_ conforming if this
    new rule is enforced.

- any program whose behaviour is affected is quietly relying on either undefined
  or unspecified behaviour.
  
  - because Guidelines and educators have spent so long telling users not to do
    this, most programs are designed not to rely on implicit ordering already.

- when the behaviour is unspecified, the compiler's freedom to "optimize" by
  reordering doesn't need to be protected. When the reordering is valid, the new
  stricter specification will not prevent it because of the as-if rule.
  
  - the majority of expressions will therefore optimize in exactly the same
    way, regardless of the abstract machine semantics.

### Future directions

A major motivating case for this change is the likely future standardization of
[statement expressions][4], which has been discussed and requested since ANSI C:

    int z = ({
      int x = 1;
      int y = 2;
      x + y;
    });

These have the same problem that already manifests with function calls by 
providing another way to inline side effects directly into an expression:

    int z = ({
      int x = ++ *p1;
      x;
    }) + ({
      int y = ++ *p2;
      y;
    });

The above behaviour isn't undefined, because there are sequence points _inside_
each statement expression (as they contain statement separators), so even if
`p1` and `p2` do point at the same object, there is no overlap in the effect 
itself, exactly like how the body of a function is "guarded" by sequence points
even if entry into two different functions is not sequenced. But as with the
function case, we also do not know which part of the expression is evaluated
first.

Statement expressions can make this worse:

    ({
      here: // (this is not usually allowed)
      x;
    }) + ({
      goto there; // sudden break out
      y;
    });
    
    goto here;
    ...
    there:
    ...

It is possible to bail out of the middle of an expression evaluation, and in
some implementations, also potentially possible to dive _into_ the middle of
one (though we hope this ends up becoming a constraint violation).

It would be good to know which side effects are triggered by providing a clear
sequence of events. It would also make the expression as a whole much more
consistent to have sequence points throughout, rather than... some sequenced
operations and some not.

Exactly the same considerations apply to the evaluations within function calls
(which can of course even bail out using `longjmp`), but statement expressions
inherit the problem and make it more obvious.

We also believe strict evaluation would resolve some outstanding issues with
the evaluation of designated initializers:

    struct S {
      int x;
      int y;
      int z;
    };
    
    struct S s = {
      .x = f1 (),
      .y = f3 (),
      .x = f2 (),
    };

Per C23 6.7.10 paragraph 20, the call to `f2` "overrides" the call to `f1`, but
it is unspecified whether `f1` is called first and the value overwritten, or if
`f1` is never called at all. Per paragraph 24, it is unspecified whether `f3` is
called before or after `f2` (both of which are at least definitely called), and
where `f1` is called relative to either _if_ it is called.

Effectively this makes side effects unusable within initializer lists. This also
cuts off an interesting folk usage of designators to provide "default" values
if the default value expression involves a side effect:

    #define MakeS(...) {   \
      .x = getDefaultX (), \
      .y = getDefaultY (), \
      .z = getDefaultZ (), \
      __VA_ARGS__          \
    }
    
    struct S s = MakeS (
        .x = nonDefaultX ()
      , .z = 6
    );

In the above example, it is well-defined for `.x` and `.z` to appear twice, and
the values `nonDefaultX ()` and `6` _will_ definitely be used, but it is not known
whether the default initializers will run and if so, when.

This significantly complicates some potential future extensions to array designators
to admit range expressions ([N3194][5]), where the repetition of effects in the
case of "empty" ranges or overlapping ranges has unclear consequences.

We again believe that the permissive specification here is not necessary: if the
user doesn't mean for a side effect in a comma-separated list of expressions to
run, they will probably not write that effect. If they do, they probably want it.
The "optimization" significantly changes the meaning of the program.

Some possible proposed future extensions also involve initializer expressions
being able to make reference to previously-initialized members by designator;
this allows member arrays to make use of other members to define a variably
modified type, or simply to establish temporary names. For this to work as expected,
the initializers must emit their effects and stores to member objects in the
syntactically-specified order and without deleting intermediate initializer
expressions.

## Prior Art

C++
Java, C#
Rust
Go, D, Swift
Javascript, Python, etc.

## Proposed wording

The proposed changes are based on the latest public draft of C23, which is
[N3096][0]. Bolded text is new text when inlined into an existing sentence.

This change set is incomplete and we expect a revised document to be needed.

**Delete** paragraph 16 from 5.1.2.3 "Program execution" (EXAMPLE 7).

Leave 6.5 "Expressions", paragraph 2 in place, for the time being, in case some
other way to have unsequenced effects remains uncaught. This leaves the core
UB in place right now.

Modify paragraph 3:

> The grouping of operators and operands is indicated by the syntax.<super>96)</super>
> Except as specified later, side effects and value computations of subexpressions
> are **sequenced from left to right <super>footnote)</super>**.
>
> **footnote) therefore in a binary expression such as `E1 + E2 * E3`, evaluation**
> **of `E1`, including side effects, is sequenced before `E2`, and `E2` is similarly**
> **sequenced before `E3`. The same is true in `(E1 + E2) * E3` because order does**
> **not depend on precedence or binding.**

(This change provides the bulk of the intent.)

**Delete** current footnote 97.

Add a new paragraph after 6.5.1 "Primary expressions", paragraph 7:

> A primary expression does not introduce a sequence point.

Modify the second sentence of 6.5.2.1 "Array subscripting", paragraph 2:

> The definition of the subscript operator `[]` is that `E1[E2]` is identical
> to `(*((E1)+(E2)))` **(including in the sequence of subexpression evaluation)**.

Replace 6.5.2.2 "Function calls", paragraph 8:

> Evaluation of the function designator is sequenced before evaluation of any
> function arguments. Evaluation of each argument is fully sequenced before the
> evaluation of the subsequent argument in left-to-right order. All operations
> within the execution of the body of the called function are sequenced after
> evaluation of the designator and arguments, and sequenced before the value
> of the called function is returned to the calling expression. <super>106)</super>

Modify footnote 106 to delete "in other words":

> 106) Function executions do not interleave with each other.

Modify 6.5.2.4 "Postfix increment and decrement operators", paragraph 2:

> The result of the postfix `++` operator is the value of the operand. As a side
> effect, the value of the operand object is incremented (that is, the value 1
> of the appropriate type is added to it). See the discussions of additive
> operators and compound assignment for information on constraints, types, and
> conversions and the effects of operations on pointers. The value computation
> of the result is sequenced before the side effect of updating the stored value
> of the operand **, which is sequenced before any further evaluations or use**
> **of the result value in the enclosing expression.**. With respect to an
> indeterminately sequenced function call, the operation of postfix `++` is a
> single evaluation. Postfix `++` on an object with atomic type is a
> read-modify-write operation with `memory_order_seq_cst` memory order semantics.<super>110)</super>

**Delete** the first sentence of 6.5.13 "Logical AND operator", paragraph 4.

**Delete** the first sentence of 6.5.14 "Logical OR operator", paragraph 4.

**Delete** the last sentence of 6.5.16 "Assignment operators", paragraph 3.

Add a new paragraph here after paragraph 3:

> Evaluation of the right operand is sequenced before evaluation of the left
> operand.

(this is the major exception to the left-to-right ordering and matches C++)

6.5.17 "Comma operator" does not strictly need to change, although the wording
becomes redundant.

Modify 6.7.10 "Initialization", paragraph 20:

> The initialization shall occur in initializer list order, each initializer
> provided for a particular subobject overriding any previously listed
> initializer for the same subobject by **overwriting the previously-initialized**
> **value of that subobject**; all subobjects that are not initialized explicitly
> are subject to default initialization.

**Delete** footnote 184.

Modify paragraph 24:

> The evaluations of the initialization list expressions are **sequenced in the**
> **order they are specified in the list. The effect of assigning the result value**
> **to the _current object_ is sequenced before evaluation of the next initialization**
> **expression in the initializer list. If two expressions initialize the same**
> **subobject, they are both evaluated in the specified sequence and the initialized**
> **value of the subobject is updated with the second value.**

**Delete** footnote 185.

(The change in initialized value is not usually observable unless the object is
`volatile`, but this should be left implicit and may change if "designator expressions"
are edded later that would make previously-initialized members accesisble as values.)

Modify 6.8 "Statements and blocks", paragraph 4, **deleting** the second half of the
second sentence ("; within that full expression...").

In 7.1.4 "Use of library functions", paragraph 1, **delete** footnote 237.

(is paragraph 3 ever necessary?)

DO NOT modify 7.17.3 "Order and consistency" or 7.17.4 "Fences".

**NOTE** Is 7.24.5 "Searching and sorting utilities", paragraph 5 redundant?

(Leave 7.31.2 "Formatted wide character input/output functions" paragraph 1 as-is?)

Modify Annex C "Sequence points":

> The following are the sequence points described in 5.1.2.3:
>
> - **Between the evaluation of the postfix and index expressions in a subscript**
>   **expression (regardless of their types). (6.5.2.1).**
>
> - **Between the evaluation of the function designator in a function call, and the**
>   **evaluation of any arguments. (6.5.2.2).**
>
> - **Between the evaluation of an argument to a function call, and the evaluation**
>   **of any subsequent arguments. (6.5.2.2).**
>
> - Between the evaluations of the function designator and actual arguments in a
>   function call and the actual call. (6.5.2.2).
>
> - **Between the evaluations of the left and right operands of all binary operators**
>   **except for the assignment operators. (6.5.5; 6.5.6; 6.5.7; 6.5.8; 6.5.9; 6.5.10;**
>   **6.5.11; 6.5.12; 6.5.13; 6.5.14; 6.5.17).**
>
> - Between the evaluations of the first operand of the conditional `?:` operator
>   and whichever of the second and third operands is evaluated (6.5.15).
>
> - **Between the evaluations of the right and left operands of all assignment**
>   **operators (the right operator is sequenced before the left). (6.5.16).**
>
> - Between the evaluation of a full expression and the next full expression to be
>   evaluated. The following are full expressions: a full declarator for a variably
>   modified type; an initializer that is not part of a compound literal (6.7.10);
>   the expression in an expression statement (6.8.3); the controlling expression
>   of a selection statement (if or switch) (6.8.4); the controlling expression
>   of a while or do statement (6.8.5); each of the (optional) expressions of a
>   `for` statement (6.8.5.3); the (optional) expression in a `return` statement (6.8.6.4).
>
> - **(deleted: before a library function returns?)**
>
> - After the actions associated with each formatted input/output function
>   conversion specifier (7.23.6, 7.31.2).
>
> - Immediately before and immediately after each call to a comparison function,
>   and also between any call to a comparison function and any movement of the
>   objects passed as arguments to that call (7.24.5).

## Questions for WG14

Would WG14 like to see something along the lines of this proposal to enforce
strict evaluation order in expressions?

Would WG14 like to enforce strict evaluation order for initializers?

Would WG14 like to enforce strict evaluation for all subexpressions rather than
just those changed in C++17?

Would WG14 prefer Java-like left-to-right order in all situations, or C++-like
right to left order for assignment and other specified exception cases?

## References

[C23 public draft][0]  
[Kernighan and Ritchie][1]  
[Refining Expression Evaluation Order for Idiomatic C++][2]  
[CERT EXP30-C: Do not depend on the order of evaluation for side effects][3]  
[GNU C Statement expressions][4]  
[N3194 range expressions][5]  

[0]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3096.pdf
[1]: https://en.wikipedia.org/wiki/The_C_Programming_Language
[2]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0145r3.pdf
[3]: https://wiki.sei.cmu.edu/confluence/display/c/EXP30-C.+Do+not+depend+on+the+order+of+evaluation+for+side+effects
[4]: https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html
[5]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3194.htm
