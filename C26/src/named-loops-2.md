    Proposal for C2y
    WG14 3658

    Title:               Simplified lexical scope for labels
    Author, affiliation: Alex Celeste, Perforce
                         Jan Schultke
                         Sarah Quiñones
    Date:                2025-07-24
    Proposal category:   Feature refinement
    Target audience:     Compiler implementers, users

## Abstract

Named loops were accepted into C2Y with consensus from WG14 at the Fall 2024
meeting. However, the feature as adopted remains contentious because of some
problems caused by reusing the `label:` feature in this way. This paper aims
to show how the major scoping problems of labels can be resolved without
needing to move away from label syntax, which would deviate from existing
practice in the field. This paper aims to maintain consistency between C and
a corresponding C++ feature that is being discussed by WG21.

-------------------------------------------------------------------------------

# Simplified lexical scope for labels

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3658
    Revises:      N3474
    Date:         2025-07-24

## Summary of Changes

### N3658
 - apply feedback from Reflector to tidy up wording
 - modify Prior Art to talk about label scopes (not named loops in general)
 - rebase against latest draft N3550

### N3474
 - original proposal, but refer back to [N3355][2] and [N2859][3]

## Introduction

After adoption of N3355 into C2Y, which provides the core "named loop" functionality,
various potential issues were identified and laid out by Keane _et al._ in
[N3377 "Named Loops Should Name Their Loops"][4].

For a discussion of the motivations for adopting the functionality itself, which has already been
accepted into C2Y, please refer to [N3355][2].

### Scoping Issues

The most important issue identified in that document is that N3355's proposed syntax reuses
labels, which are currently defined in C to have "function scope". This is a single, non-nesting,
lexically-unordered scope that spans the entire body of a function definition, in order to allow
code like this:

    void f (int x, int y)
    {
      if (x == y) {
        goto mylabel;  // looks "into" two braced scopes on subsequent lines
      }                // to find the declaration of the label to jump to
      
      for (int z = 0; z < y; ++ z) {
        if (z > x) {
          mylabel:
          g ();
        }
      }
    }

...as well as forward jumps within the same level of bracing (used for early-exits), jumps backward
and potentially into nested braces, and so on. This is useful functionality and core to how `goto`
works in C.

A consequence of this is that, without extensions such as GCC's
[locally declared labels][5], it is not possible to reuse the same label name twice within the same
function body:

    #define SomeWork(X) ({          \
      if (bad (X)) goto failblock;  \
      ...work                       \
    failblock:                      \
      result;                       \
    })
    
    // cannot be used this way:
    void f (int x, int y) {
      y = SomeWork(x);
      y = SomeWork(x);  // error - duplicate 'failblock'
    }

GCC adds locally declared labels (just) to make this possible:

    #define MoreWork(X) ({          \
      __label__ failblock;          \
      if (bad (X)) goto failblock;  \
      ...work                       \
    failblock:                      \
      result;                       \
    })
    
    void g (int x, int y) {
      y = MoreWork(x);
      y = MoreWork(x);  // OK
    }

The `__label__` feature declares an identifier has having block scope, which can later be used to
define a `goto` target. The advantage of this for GCC is that it does not interfere with common
use of labels in non-GNU parts of the code.

For named loops, this manifests as every loop in a function having to have a distinct name, even
if the loop was introduced by a macro. Kean _et al._ provide an example of an update macro which
expands to a named loop, but the problem generalizes to:

    void h (int x, int y) {
      loop: for (int z = 0; z < x; ++ z) {
        if (z == 10) break loop;
      }
      
      loop: for (int z = 0; z < y; ++ z) {
        if (z == 10) break loop;
      }
    }

It is clear enough to a human reader what this means - the name is associated with the block, and
there is only one reasonable target for the `break` in each loop - so reusing the name `loop`
_ought_ to make exactly as much sense as reusing the name `z` for the loop counter.
But C's label scoping rules do not allow this.

### Other issues

N3377 also raises a stylistic concern about whether the loop label properly implies the jump target.
We believe this is ultimately a subjective matter and do not address it here.

Any syntax will become intuitive if a user makes use of it enough (see also:
declaration-reflects-use, ternary operators, `[static]`, ...).

N3377 raises a concern that because labels can be freely positioned within a block in C23 (and in
many implementations, already could), an implementation is not required to associate a label with
the entity it "names". We do not believe this is a real barrier to implementability; from experience
(Helix QAC already internally treats labels as separate statements, "associating" them with the next
block item), this is easy enough to work with.

## Proposed solution

Whereas Keane _et al._ propose a new kind of syntax for label names, which adds new scoping rules
for the new form, we propose simply _relaxing_ the scoping rules around labels, for the scenarios
where the code makes intuitive sense.

Instead of contorting the concept of "scope" to use it to define labels in a way that `goto` can
then use, we propose that the special lookup property becomes associated _with the_ `goto`
_statement_ specifically, rather than with labels.

Labels are not objects and do not have a value (outside of [extensions][6] which we do not propose
to adopt); they have no storage, and no storage duration, and cannot be used in arbitrary expressions
(again outside of extensions). Therefore, instead of treating them as having a single consistent
scope, or kind of scope, we suggest that their lookup should be made _entirely context dependent_:

- an identifier that appears as the operand to `break` or `continue` _shall_ name a containing
  control structure.
- an identifier that appears as the operand to `goto` _shall not_ be the identifier of more than
  one label within the containing function.

The effect of this is implicit: labelled `break` and `continue` "just work", because the jump
target is already defined in terms of the containing named control structure ("...the jump exits
the `switch` or iteration statement named by the label ..."; "... the jump is to the
loop-continuation of the iteration statement named by the label ..."). Specifying that lookup
refers to an "innermost enclosing" control structure is more direct than using scope as a
tool of abstraction here.

The `goto` statement also defines its action in terms of "a label located somewhere in the
enclosing function" and does not need to make reference to any kind of scope for this to work.

All existing valid C code making use of `goto` continues to be valid with these relaxed
constraints. Additionally, the desirable code above, where the same identifier can name more
than one loop or `switch` in the same function body, works in the intuitive and expected way
without needing additional text.

## Impact

The GCC [Labels as Values][6] extension allows label names to appear within arbitrary expressions,
and therefore uses a more traditional form of lookup.

Since this extension is used with `goto *`, we suggest that the label-address operator `&&` should
impose the same constraint as `goto` itself on any identifier that appears as its operand.

This should mean all existing code using `&&` continues to work.

Currently, the same label (not just the same identifier) can be used to both name a control
structure, and as the target of a `goto`. This should continue to work without change so long as
only one control structure has the given name. If two control structures share the same name,
the user should not expect `goto` to work because the target is ambiguous. There is no special
overlap between constraints here.

## Alternatives

### N3377

A number of potential issues with the alternative syntax proposed in N3377
[were also identified by Schultke _et al._ in P3568](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3568r0.html#opposition-to-n3377).

N3377 introduces a new kind of label, increasing complexity of the language for users and for
tooling. It occupies a new position in the syntax, which rules out certain extensions that aim
to put context-sensitive keywords in that place. Subjectively, it may be more difficult to read
as it adopts a "middle-endian" convention, and does not fix the subjective issues with label
placement at the beginning of a loop. It is not symmetric with the rest of the language syntax
and would not permit extensibility such as block-breaking (not currently proposed for C, but 
logically consistent and present in other languages).

Further discussion in greater detail is provided in [P3568][1].

### GNU locally declared labels

Locally declared labels can be used effectively to solve the specific problem of designing macros
that are meant to be self-contained and not clash with the user scope or other macros within it.
The feature is already widely-used in order to provide this capability, especially in combination
with statement expressions. However, outside of macros, writing it out is subjectively ugly, and
raises complexity because it requires an explicit opt-in to the "right" semantics by way of an
extra declaration. It requires a new keyword and introduces more complicated scoping concepts
instead of simplifying them.

The interactions between locally-declared labels, regular labels, and other GNU C extensions, is
also not necessarily entirely clear: by changing the scope discipline, the power of `goto` is
also changed to _implicitly_ allow emergent functionalities like jumping out of nested function
invocations.

### gensym

A [`gensym`][15] macro could be provided by the implementation or built atop other features like
[`__COUNTER__`][16]. This could be used to create new identifiers at the point of expansion, which
would allow every expansion of a macro relying on label names to be unique.

This also requires the user to opt-in to the correct or intuitive behaviour, and is similarly
subjectively ugly and potentially quite difficult to use correctly, depending on how the expansion
rules are defined. This seems unnecessarily complicated compared to the other available options,
and cannot be used for the non-macro case at all.

## C++ Compatibility

[A similar proposal][1] has been presented to WG21 as P3568.

[WG21 voted to strongly support the feature itself][14], but did not express a strong preference
between the N3355 syntax and the N3377 syntax.

WG21 expects to adopt whichever syntax WG14 prefers.

## Prior Art

See [N3355][2] for discussion of Prior Art for the named loop feature itself.

Loop names may shadow outer loop names in [Rust][7]:

    // Loop label shadowing example.
    'a: for outer in 0..5 {
        'a: for inner in 0..5 {
            // This terminates the inner loop, but the outer loop continues to run.
            break 'a;
        }
    }

This is also allowed in [Perl][13].

In [Java][10] (see also [JLS][11] 14.7, 14.15, 14.16), a label's scope is the statement
it labels, and it cannot be reused within that scope; but it can be reused elsewhere in
the same method (so two loops can have the same name, but only if they are not nested
inside one another).

On the other hand, [Javascript][8] does not allow duplicate labels at all, even though
it supports the same `break` and `continue` statement syntax. See [ECMA-262][9] 8.3.1.

### Teachability

(i.e. consistency with the Prior Art of C itself)

The advantage of the existing structured jump operators is that the user does not
need to think about where they jump _to_, whereas with `goto`, knowing where the
target label is, is core to understanding what the jump itself will do.

We consider that the same principle continues to hold with the named jump variants:

- a `break` statement always jumps _forwards and outwards_, regardless of what it
  may or may not name;
- a `continue` statement always jumps _to the end_ of a containing block (some users
  think of it as jumping "back" to the start, but this isn't quite true and is it
  specified as a forward jump)

Even with names changing _which_ block the jump targets the end of, the direction of
the jump and its local effect on execution of the currently-enclosing block is
completely consistent, no matter what containing control structure is targeted.
Therefore, statement-local reasoning doesn't change at all, and zooming slightly
outwards, the reasoning is consistent (_more_ consistent when considering the ability
to eliminate the asymmetric behaviour of unlabeled jumps w.r.t `switch`).

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3550][0]. Bolded text is new text when inlined into an existing sentence.

Modify 6.2.1 "Scopes of identifiers, type names, and compound literals":

Modify paragraph 2 to remove the concept of "function scope":

> For each different entity **other than a label** that an identifier designates,...
>
> ...name spaces. There are **three** kinds of scopes: file, block, and function prototype.
> (A _function prototype_ is a declaration of a function.)

Delete paragraph 3, and replace by:

> A label name is the only kind of identifier that is not associated with any scope.
> The same identifier can appear as the name of more than one label within the same function.
> An identifier that is the name of a label can be used by a `goto` statement from anywhere
> in the function in which the label appears, including any point before its definition.<sup>footnote)</sup>
>
> <small>footnote) if there is more than one label with the same identifier in a function, the identifier
> cannot be used with `goto` as the `goto` statement will not be able to determine the intended target.</small>

A description of the mechanics of named `break` and `continue`, or even `goto`, is not needed here.
The Constraint is applied in the definition of `goto` and does not need to be repeated.

Modify 6.8.2 "Labeled statements":

Delete paragraph 3 ("shall be unique").

Modify paragraph 7 to add a number ("1") to the EXAMPLE, and add a second example after it:

> **EXAMPLE 2**  Labels with the same identifier may name more than one statement within the
> same function:
>
>     loop: for (int z = 0; z < x; ++ z) {
>       /* loop may be used from here, referring to the first instance */
>     }
>     
>     loop: for (int z = 0; z < y; ++ z) {
>       /* loop may be used from here, referring to the second instance */
>     }
>
> Both `for` loops are named by an identifier `loop`.

Modify 6.8.7.2 "The `goto` statement", adding a new paragraph to "Constraints" after paragraph 1:

> There shall be exactly one label in the enclosing function with the same identifier named by
> a `goto` statement.

Modify 6.8.7.3 "The `continue` statement", paragraph 4:

> If the `continue` statement has an identifier operand, the jump is to the loop-continuation of
> the **innermost enclosing** iteration statement named by the label with the corresponding identifier.
> **If the `continue` statement has no operand**, the jump is to the loop-continuation of the
> innermost enclosing iteration statement.

Modify 6.8.7.4 "The `break` statement", paragraph 4:

> If the `break` statement has an identifier operand, the jump exits the **innermost enclosing**
> `switch` or iteration statement named by the label with the corresponding identifier.
> **If the `break` statement has no operand**, the jump exits the innermost enclosing
> `switch` or iteration statement.

Optionally, editorially, the order of the above two sentences (in both 6.8.7.3 and 6.8.7.4)
could be switched to put the no-identifier form first, if this improves readability.

Modify Annex I, adding a bullet point to I.2:

> - The same label identifier is used with a `goto` statement and a `break` or `continue`
>   statement within a function body (6.8.7).

(notwithstanding the possible future removal of Annex I entirely)

## Questions for WG14

Does WG14 want to accept the proposed change to relax the scoping rules for labels,
retaining the existing syntax for named loops?

## Acknowledgements

Thanks to Jan Schulkte, Sarah Quiñones, and Herb Sutter for supporting this counter
proposal; and to Javier Múgica and Jens Gustedt for feedback on the change.

## References

[C2y public draft N3550][0]  
[P3568][1]  
[N3355][2]  
[N2859][3]  
[N3377][4]  
[Locally Declared Labels (GCC)][5]  
[Labels as Values][6]  
[Named loops in Rust][7]  
[Named loops in Javascript][8]  
[ECMAScript 2023][9]  
[Named loops in Java][10]  
[Java language specification][11]  
[Named loops in Cpp2][12]  
[Named loops in Perl][13]  
[WG21 votes on named loops][14]  
[gensym (Common Lisp Hyper Spec)][15]  
[The `__COUNTER__` predefined macro][16]  

[0]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3550.pdf
[1]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3568r0.html
[2]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3355.htm
[3]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2859.pdf
[4]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3377.pdf
[5]: https://gcc.gnu.org/onlinedocs/gcc/Local-Labels.html
[6]: https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html
[7]: https://doc.rust-lang.org/reference/names/scopes.html?highlight=label#loop-label-scopes
[8]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label
[9]: https://ecma-international.org/wp-content/uploads/ECMA-262_14th_edition_june_2023.pdf
[10]: https://docs.oracle.com/javase/tutorial/java/nutsandbolts/branch.html
[11]: https://docs.oracle.com/javase/specs/jls/se18/jls18.pdf
[12]: https://hsutter.github.io/cppfront/cpp2/functions/#loop-names-break-and-continue
[13]: https://perldoc.perl.org/perlsyn#Loop-Control
[14]: https://github.com/cplusplus/papers/issues/2212
[15]: http://clhs.lisp.se/Body/f_gensym.htm
[16]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3457.htm
