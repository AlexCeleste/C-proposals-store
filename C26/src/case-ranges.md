    Proposal for C2y
    WG14 3370

    Title:               Case range expressions, v3.1
    Author, affiliation: Alex Celeste, Perforce
    Date:                2024-10-01
    Proposal category:   New feature
    Target audience:     Compiler implementers, users

## Abstract

GCC supports an expression syntax to allow the user to describe a contiguous 
range of integer constant expression in certain contexts. We propose adopting
this expression kind and the simplest use case supported by GCC, along with
comments on its integration and future directions for expansion of the feature.

-------------------------------------------------------------------------------

# Case range expressions, v3.1

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3370
    Revises:      N3354
    Date:         2024-10-01

## Summary of Changes

### N3370
 - simplify after session feedback so that only a single question is presented to for adoption
   - remove the recommended practice to warn on single-value ranges
   - make the warning on empty ranges a recommended practice instead of a constraint

### N3354
 - rebase against N3301

### N3269
 - rebase wording on C2y draft
 - just use the GCC syntax, scrap the arrows and `::`
 - remove right-open ranges
 - remove arithmetic conversion (conflicted with GCC behavior)

### N3194
 - original proposal
 
## Introduction

The GNU C dialect includes an operator which can be used to describe a range
of integer constant expressions, which is valid in two contexts where the kind 
of repetition implied makes sense:

    switch (n) {
      case 1 ... 10:
        something ();
        break;
    }

[which is equivalent to writing][1]

    if (n >= 1 && n <= 10) {
      something ();
    }

except that `n` is only evaluated once; and

    int arr[] = {
      [3 ... 6] = n,
    };

[which is equivalent to writing][2]

    int arr[] = {
      [3] = n,
      [4] = n,
      [5] = n,
      [6] = n,
    };

except that again, `n` is only evaluated once.

A range expression is defined as:

    integer-constant-expression  ellipsis  integer-constant-expression

and does not describe a kind of value that otherwise exists in the language;
in GNU C, the `case` and designator syntactic constructs can accept either a
single integer constant expression, or a range expression, and the evaluation
or interpretation of the range does not result in a value that can be copied or
passed around further.

Range expressions are very useful for both conciseness of expression, and also
to avoid repeated evaluations in the designator case (by avoiding the need for
a temporary variable). If a `switch` has a large number of possible ranges it
can be expressed much more easily by a "wide" range than by stacking the `case`
labels (e.g. `case 1 ... 1000:` is quite plausible but would not be readable if
written out normally). While any `switch` can be expressed as an `if`, the
readability of the code may be affected and the choice between the two is
important when deciding how best to convey the author's intent.

There are some complexities in the designator case that arise from the fact
that designators within an initializer do not need to be unique, and the lack
of specification for evaluation order and effects in the existing initializer
semantics. Therefore, while we propose introducing range expressions as a
general reusable construct, we do not propose adding them for designators until
the order of evaluation issues with existing initializer syntax are cleaned up.

## Proposal

We propose that a _constant-range-expression_ is added as a top-level expression 
kind alongside _constant-expression_ and _expression_ for use by selected 
syntactic forms.

We propose that the `case` label construct be allowed to admit either a
_constant-expression_ or a _constant-range-expression_ as its operand. The
constraints this range would be subject to are largely the same as those which 
currently apply to the _constant-expression_ operand.

The _constant-range-expression_ can be reused in other contexts as and when they
emerge, so we do not believe this should be a property locked to `case` itself
specifically. There are several other compelling use cases for ranges, including
GCC's own use to describe designator ranges.

### Case constraints

Extending the constraints on the current `case` label form is relatively simple
as there are no repeated-side effects to consider.

    void foo (void);

    void use (int n)
    {
      switch (n) {
        case 1:
          foo ();
          break;
        // case 4 : // error, overlaps 2 ... 5
        //   foo ();
        //   break;
        case 2 ... 5:
          foo ();
          break;
        case 6 ... 6: // OK (but questionable)
          foo ();
          break;
        case 8 ... 7: // not a GCC error
          foo ();
          break;
        case 10 ... 4: // not a GCC error despite overlap
          foo ();
          break;
      }
    }

`case 4` and `case 2 ... 5` clearly and unambiguously overlap. This should be
treated as a violation of 6.8.5.3 paragraph 3, which says "no two of the `case`
constant expressions ... shall have the same value". The order of the two labels
produces a slightly different error message in tools, but this is a UX issue and
does not make the code more or less valid either way - we **do not** propose
that single-expression cases should be able to override parts of a range
expression, as this is not existing practice and seems highly likely to lead to
user confusion. (Subjectively, such forms would seem to invalidate the argument 
that range-based `case` labels would be clearer, too.)

Because a range describes a sequence of value from _low_ to _high_, GCC does
not consider `8 ... 7` to be an error, but an empty range which cannot match 
`n`. We propose a recommended practice to warn on encountering this.

Because `10 ... 4` also describes an empty range, tools will usually ignore the
_apparent_ overlap with `2 ... 5`, because the range does not actually specify
any values and therefore does not overlap despite the arithmetic values of the
endpoints.

Tools allow `6 ... 6` because the range contains at least one value. We believe
this is fine, as the resulting `case` label will have a value that can match.

### Operator syntax

Although the use of the `...` operator is existing practice in GCC and compatible
compilers, it does have a minor usability problem in practice:

    case 11...12: // syntax error, PP-number with too many dots
      foo ();
      break;

A space is needed between the _low_ and _high_ values and the range operator,
because a dot is a valid part of a PP-number in the preprocessor syntax and
therefore the above form `11...12` is lexed by most tools as a single number,
which then fails translation into a valid C-language token.

This is slightly inconsistent:

    enum {
      ELEVEN = 11,
      TWELVE = 12,
    };
    
    case ELEVEN...TWELVE: // this is fine 🍵🔥
      foo ();
      break;

The above snippet parses, because dots do not form part of an identifier, but
is fragile, and may break if the code is edited by hand.

Essentially this operator is making the a similar mistake that compound assignment
did in the original B language with the `=+`, `=-`, `=*` operators.
The problem is not as serious since it only results in a syntax error and not in
valid code that silently does something different.

The problem with `...` is not unresolvable and it would be fairly simple to
add a conversion from "PP-number containing `...`" to a sequence of two or three
C-language tokens; but this still seems confusing as it breaks the 1-1 between
preprocessor and C tokens; and anyway neither Clang nor GCC have done so.

The opinion of the Committee at the January 2024 meeting was that the existing
practice for operator spelling should be adopted directly, and to treat any
problems parsing adjacent tokens as a QoI issue (it is the responsibility of an
implementation to preprocess identifier and ellipsis tokens separately and a tool
that merged them into a single PP-number would already be non-conforming).

Alternative operators were proposed but would lack the advantage of working on
any existing code. The proposed `::` and `->` tokens also caused ambiguity
problems at the grammar level, which `...` does not suffer from.

In light of the fact that this problem can only manifest as an error, it was not
considered an obstacle to direct adoption.

### Open intervals

The GCC feature describes a closed interval (in the range expression `low ... high`,
the values in the range include both `low` and `high`).

    case 1 ... 4:
    
    // exactly equivalent to
    case 1:
    case 2:
    case 3:
    case 4:

A right-open interval is often also intuitive, especially for future uses in
array contexts, as it can refer to the element count as the right hand operator:

    case 1 ~> 4: // pretend ~> operator exists for right-open
    
    // would be equivalent to
    case 1:
    case 2:
    case 3:
    // 4 is past-the-end

However, at the January 2024 meeting the Committee decided that the right-open
range would not be useful enough to standardize as part of this proposal.

## Prior Art

This is a core GNU C language feature and is supported by most compilers that
claim any level of GCC compatibility. GCC itself, as well as [Clang][5], and
third party tools emulating GNU C such as [QAC][3], all support this feature.

Experience implementing the feature in Helix QAC suggests that the burden on 
implementers is minimal for both current usages. The `case` feature is essentially
trivial as the relevant `case` labels can, if necessary, be expanded out just
after parsing-time without even needing the backend to be aware of them (though
in practice we found that large ranges expanded this way could potentially
cause surprising slowdowns in an optimizing or dataflow backend, and that it was
better to integrate a backend-aware form of support; this was not complicated but 
will depend on the tool architecture, if it can handle `if` style expressions in
the branches).

The designator feature is not complicated to implement at all, but does raise
potential user confusion issues as the designators can allow repetition, the
overriding of side effects, etc. These are issues that already exist within the
Standard initializer rules. GCC also surprisingly allows empty ranges to compile
in this context, which it rejects for `case` labels; these introduce some more
potential for user confusion as there is now another way for side effects to be
silently eliminated even without the user overriding an initializer. As a result
we think the syntax should be reserved but addition of this part of the feature 
held off until the existing issues are cleaned up.

### Slice syntax

A very related use to designator ranges, which initialize a subrange of an array,
is slices, which read from an initialized object (or perhaps describe an lvalue).
[A number of languages][4] provide this feature and use a range-like expression 
as the operand to the subscript operator or its equivalent in order to create a 
slice value (whatever value category that is in the respective language).

We do not propose standardizing anything like this at this time, but by making 
the range expression its own expression kind instead of baking it directly into 
the syntax of `case` specifically, we leave the door open to reuse a single 
syntax for all range- and slice-related features that may be introduced in 
future. This will keep the language as consistent as possible.

In languages which support slices, one or both operands to the range operator
can be omitted and implicitly refer to the maximum extent of the range. Since
we are not currently proposing slices, we do not include this in the current
proposal. It may prove useful to add in future.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3301][0]. Bolded text is new text when inlined into an existing sentence.

### Constant expressions

Add a new section within 6.6 "Constant expressions":

> #### 6.6.1  Constant range expressions
>
> **Syntax**
> 
> _constant-range-expression:_  
>    _constant-expression_  `...`  _constant-expression_  
> 
> **Description**
> 
> A _constant range expression_ is a special form to describe a sequence of constant
> expressions with contiguous incrementing values, from a low value on the left, to 
> a high value on the right.
> 
> The resulting sequence cannot be used as a value expression and is only permitted
> in specific contexts.
> 
> **Constraints**
> 
> The left-hand value and the right-hand value shall be integer constant expressions.
> 
> **Semantics**
> 
> A range expression describes a sequence of integer constant expressions without
> listing each intermediate value explicitly. This is permitted as the operand to
> a `case` label to indicate that a single label matches multiple values.
>
> The sequence described by the `...` operator is a _closed range_, which contains
> all integer values in sequence starting from and including the left, low value,
> up to and including the right, high value<sup>footnote)</sup>.
>
> <small>footnote) a range is not itself usable as a value and therefore does not
> have any specific type or representation, or perform any type conversion.</small>
>
> If the right-hand value of the _closed range_ is less than the left-hand value,
> the sequence described by the range is empty.
> 
> **Recommended practice**
> 
> Implementations are encouraged to emit a diagnostic message when a range is empty.
> 
> **EXAMPLE 1**  Range expressions may be used as the operand to a `case` label to
> concisely describe a sequence of matching values in one label:
> 
>     switch (n) {
>       case 2 ... 5: // equivalent to
>                     //   case 2:
>                     //   case 3:
>                     //   case 4:
>                     //   case 5:
>         f ();
>         break;
>     }
> 
> **EXAMPLE 2**  Because a range expression describes a closed range, it is
> possible to match past-the-end values such as the size of an array:
> 
>     int arr[N];
>     
>     switch (i) {
>       case 0 ... N:      // matches the past-the-end range of arr
>         f (arr[i]);      // not OK, will dereference arr[N]
>         g (&arr[i]);     // may be OK depending on purpose
>         break;
>     }
>     
>     switch (i) {
>       case 0 ... N - 1:  // only matches the valid element range of arr
>         f (arr[i]);      // OK
>         break;
>     }
>
> **EXAMPLE 3**  This range expression describes a range with elements that have
> different types. The implementation will create a range with the _values_ from
> -10 to +10 in whatever representation it needs in order to preserve the meaning
> of the expressed range:
>
>     case -10 ... 10ul:
> 
> **Forward references:** `switch` statements (6.8.5.3)

### Case

Within 6.8.2 "Labeled statements", Syntax, paragraph 1 (the grammar), add a new
production to _label_:

> _label:_  
>   _attribute-specifier-sequence_ <sub>opt</sub> _identifier_  `:`  
>   **_attribute-specifier-sequence_ <sub>opt</sub> `case` _constant-range-expression_  `:`**  
>   _attribute-specifier-sequence_ <sub>opt</sub> `case` _constant-expression_  `:`  
>   _attribute-specifier-sequence_ <sub>opt</sub> `default`  `:`  

Constraints are added to the description of the range expression itself and are
therefore not needed in this section.

Add a new paragraph after paragraph 4:

> A `case` label _specifies_ the value of the following constant expression, if
> followed by a single _constant expression_, or each value in the sequence described
> by the following range, if followed by a _constant range expression_.

### Switch

Within 6.8.5.3 "The `switch` statement", modify paragraph 3:

> The expression of each `case` label shall be an integer constant expression **or**
> **a constant range expression** and no two of the `case` constant expressions 
> associated to the same `switch` statement shall **specify** the same value after
> conversion. There may be at most one `default` label associated to a `switch` statement.
> (Any enclosed `switch` statement may have a `default` label or `case`**-specified** 
> **values** that duplicate `case`**-specified values** in the enclosing
> `switch` statement.)

Add a new paragraph after paragraph 3:

> The arithmetic values specified by a constant range expression shall not change as
> a result of conversion to the promoted type of the controlling expression.

(This used to be permitted by GCC and Clang with a warning, but has since been made
a hard error. There is no reason to specify a behaviour for what would always be an
unintentional value change here.)

Modify existing paragraph 4:

> ...and on the presence of a `default` label and the **specified** values of any 
> `case` labels on or in the `switch` body...

Modify paragraph 5:

> The integer promotions are performed on the controlling expression. The 
> **values specified by** each `case` label **are** converted to the promoted
> type of the controlling expression. If a converted value matches that of the
> promoted controlling expression, control jumps to the statement or declaration
> following the matched `case` label. Otherwise, if there is a `default` label,
> control jumps to the statement or declaration following the `default` label.
> If no converted `case` **value** matches and there is no `default` label, no
> part of the `switch` body is executed.

Add a new sentence to the end of paragraph 5:

> A `case` label with a _range expression_ that describes an _empty range_ (which
> occurs when the low value has a value greater than the high value) never matches
> any value of the controlling expression <sup>footnote</sup>.
>
> <small>footnote) even if the controlling expression has the same value as one of
> the operands to the _empty range_ expression.</small>

Modify paragraph 6:

> As discussed in 5.3.5.2, the implementation may limit the number of `case`
> **labels** in a `switch` statement.

Add a new example after paragraph 7:

> **EXAMPLE 2**  A value specified as part of the sequence in a range expression
> argument to `case` shall not be repeated by a different label:
>
>     switch (expr)
>     {
>     case 1 ... 5:
>       // ...
>       break;
>     case 6:   // OK, 6 has not been specified
>       // ...
>     case 4:   // constraint violation:
>       // ...  // 4 was specified by 1 ... 5
>       break;
>     case 3 ... 8: // constraint violation:
>       // ...      // 3, 4, 5, 6 were specified above
>       break;
>     }

## Questions for WG14

Does WG14 want to add this feature as-described to C2Y?

Would WG14 like to see further work on ranges for designators?

## Acknowledgements

Huge thanks to Joseph Myers for detailed and thorough review of previous versions.

## References

[C2y public draft][0]  
[GCC case ranges][1]  
[GCC designator ranges][2]  
[Helix QAC][3]  
[Wikipedia examples of array slicing][4]  
[Clang `case` implementation details][5]  

[0]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3301.pdf
[1]: https://gcc.gnu.org/onlinedocs/gcc/Case-Ranges.html
[2]: https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html
[3]: https://www.perforce.com/products/helix-qac
[4]: https://en.wikipedia.org/wiki/Array_slicing
[5]: https://clang.llvm.org/doxygen/classclang_1_1CaseStmt.html
