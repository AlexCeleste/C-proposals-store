    Proposal for C2y
    WG14 3580

    Title:               `if` declarations, v5.1
    Author, affiliation: Alex Celeste, Perforce
    Date:                2025-06-23
    Proposal category:   Feature  improvement
    Target audience:     Compiler implementers, users

## Abstract

The C `if` statement only admits an expression as its operand. In contrast,
the `for` statement admits either an expression or a declaration as its
first operand. We propose that C modifies the `if` statement to allow the
operand to be either a declaration or an expression, and to optionally allow
a second expression clause when the first clause is a declaration. This is
taken from existing practice in C++.

-------------------------------------------------------------------------------

# `if` declarations, v5.1

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3580
    Revises:      N3473
    Date:         2025-06-23

## Summary of Changes

### N3580
 - rebase on N3550
 - minor updates with Committee feedback to retain the Constraint for now
   and fix an error in the semantic description

### N3473
 - rebase on N3435 which integrates this feature as described in N3356,
   reducing the changes only to those differing from the Working Draft
 - remove the _simple-declaration_ production again after Reflector feedback

### N3388
 - prohibit the use of non-automatic storage classes in the simple-declaration form
 - add a rewrite rule for the simple-declaration form to clarify the semantics 

### N3356
 - rebase against N3301
 - fixes to proposed grammar based on feedback
 - move more semantics to the toplevel of 6.8.5

### N3267
 - rebase wording on C2y draft
 - rework the grammar to use original, simpler rules instead of copying C++
 - scope of the identifier persists into the `else`
 - remove changes to `for`

### N3196
 - original proposal
 
## Introduction

This paper proposes a clarifying revision to the feature adopted into C2Y
as N3356, allowing a variable to be declared in the controlling clause of
a selection statement.

This revision is in response to Reflector feedback which raised two issues:

Firstly, the semantics of the "third form" (no explicit expression) were not
clear enough in the case of certain storage-class specifiers. Because the
wording simply said "is the value of" it was unclear how a value might be
re-evaluated or re-initialized.

By providing explicit wording to guarantee that a declaration in the third form
is always rewritten:

    if (T x = y) { ...
    
    // is ALWAYS the same as
    if (T x = y; x) { ...

...we can clarify the intended number of evaluations and the nature of the
implicit controlling expression. We can also make it explicitly implementation
defined whether `x` is re-evaluated when `volatile` or `atomic`, rather than
impose a behaviour here. This opens up space for helpful warnings.

Implicitly, this now provides a well-defined behaviour for a statement like:

    if (static int once = 1) {
      once = 0;
      work (); // only called once per program
    }

For the time being we suggest a Constraint against doing this in response to
feedback on the Reflector. We should lift the Constraint if it becomes clear
that users would understand code written this way. An equivalent to this
Constraint exists in C++ as of C++23.

Secondly, it was judged to be unclear to place _simple-declaration_ alongside
the definition of declarations in general in 6.7.1, when the grammar rule
does not actually define any separate kind of declaration or semantics.
Therefore, the rule is moved to 6.8.5 and the wording clarifies that it is
immediately rewritten into a different form that uses existing kinds of
declaration semantics.

See previous revisions, and [p0305][2], for a discussion of motivations for
adding the feature, which is now part of C2Y.

## Alternatives

See previous revisions for a discussion of alternatives.
This feature has now been adopted into C2Y and is available in the language.

## Prior Art

The feature was standardized in [C++17][5] and is now widely used.

### Compatibility

The feature standardized here differs slightly from the C++ feature by
not including the C++ grammatical construct _condition_, which allows
the second clause of `if` and `for` to be a second, completely separate
declaration:

    if (int x = 0; int y = x + 1) {
      y;
    }

Instead of over-complicating the grammar in a single change, and confusing
the change to `if` with largely-unrelated changes to `for`, we separate
this aspect out and will consider aligning with this for both statement kinds
in a subsequent proposal instead. Therefore, the second clause to `if` is
limited to being an expression only in this proposal.

This was an emergent feature that users are unlikely to try to use intentionally.

The _declaration-condition_ rule defined in this version of the wording is
intended to help future changes develop this if the Committee agrees with
the direction.

## Impact

See previous revisions for a discussion of impact on tooling.
This feature has now been adopted into C2Y.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3550][0]. Bolded text is new text when inlined into an existing sentence.

### Selection

Delete the _simple-declaration_ rule from the grammar in 6.7.1 "Declarations".

Delete 6.7.1 paragraph 14 and its associated footnote 127.

Delete the forward reference to 6.8.5 from 6.7.1.

Modify the statement grammar in 6.8.5.1 "Selection statements", Syntax, paragraph 1,
replacing _simple-declaration_ with _condition-declaration_:

> _selection-header:_  
>   _expression_  
>   _declaration_ _expression_  
>   **_declaration-condition_**  
>
> **_declaration-condition:_**  
>   **_attribute-specifier-sequence<sub>opt</sub> declaration-specifiers declarator = initializer_**

(This makes failure to initialize the object in the third form a syntax error,
and intentionally limits it to declaring a single object.)

Add a new paragraph to 6.8.5.1, before paragraph 2:

> **Constraints**
>
> Storage-class specifiers other than `auto`, `constexpr`, or `register` shall not
> appear in the declaration specifiers of a _declaration-condition_.

(WG14 voted to include this Constraint in Spring 2025.)

Modify paragraph 3 of "Semantics" to delete the second sentence "Otherwise, ...":

> If the _selection-header_ is the first or second form, the controlling expression
> is the _expression_ of the _selection-header_.

and add a new paragraph before paragraph 4:

> A _declaration-condition_ in the third form (with specifiers `T`, declarator `D` and
> initializer `X`), such as
>
>     T D = X
>
> is always treated exactly as if it had been written in the second form, as
>
>     T D = X; Di
>
> where `Di` is the identifier declared by the declarator `D`; except that it is
> implementation-defined whether the expression `Di` is evaluated for the purposes
> of volatile access.

### Iteration

The proposed change to 6.8.6 "Iteration statements" has been removed from this
feature proposal and will be revisited at a later time.

Adding a named production for _declaration-condition_ is intended to facilitate
extending the feature in this space at some later date.

## Questions for WG14

This feature was adopted at the 71st WG14 meeting in Fall 2024.

Does WG14 want to adopt the changes to the existing wording that remove
_simple-declaration_ and clarify how the third form is treated?

At the 72nd WG14 meeting in Spring 2025, WG14 voted to retain the Constraint
against other storage class specifiers in the declaration specifiers of the
_declaration-condition_ form.

## Acknowledgements

Huge thanks to Joseph Myers for detailed and thorough review of previous versions.
Thanks to Jens Gustedt, Martin Uecker and Jakub Lukasiewicz for usability feedback.

## References

[C2y public draft][0]  
[N740 Declarations in `for`][1]  
[p0305r1 Selection statements with initializer][2]  
[Anaphoric macros][3]  
[Example of `just` and `expect`][4]  
[C++17][5]  

[0]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3550.pdf
[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n740.htm
[2]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0305r1.html
[3]: https://en.wikipedia.org/wiki/Anaphoric_macro
[4]: https://github.com/C-Requests-for-Implementation/crfi-examples/blob/main/crfi-4-example.c#L82
[5]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4659.pdf
