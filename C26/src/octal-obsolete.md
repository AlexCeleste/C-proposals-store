    Proposal for C2y
    WG14 3353

    Title:               Obsolete implicitly octal literals and add delimited escape sequences
    Author, affiliation: Alex Celeste, Perforce
                         Corentin Jabot
                         Aaron Ballman
    Date:                2024-09-18
    Proposal category:   Clarification/enhancement
    Target audience:     Compiler implementers, users

## Abstract

The use of base-8 instead of base-10 by integer literals that begin with a zero 
digit is the source of frequent confusion. We propose marking the use of such 
literals as obsolete in order to encourage a warning that will prompt rewrites, 
and the introduction of a new prefix to explicitly mark literals that are 
genuinely intended to be in base-8.

This proposal also revives N2785 to add a new escape sequence syntax for
string and character literals.

-------------------------------------------------------------------------------

# Obsolete implicitly octal literals and add delimited escape sequences

    Reply-to:     Alex Celeste (aceleste@perforce.com)
                  Corentin Jabot (corentin.jabot@gmail.com)
                  Aaron Ballman (aaron@aaronballman.com)
    Document No:  N3353
    Revises:      N3319
    Date:         2024-09-18

## Summary of Changes

### 3353
 - rebase against N3301 (incorporates "constant" -> "literal")
 - don't touch `printf` at this time
 - retitle to avoid confusion w.r.t inclusion of escape sequences
 - fixes based on feedback

### N3319
 - remove original proposal for escape sequences,
   replace with wording from [N2785](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2785.pdf)
 - incorporate suggested changes to `printf` family
 - fixed error in grammar
 - rebase against N3220

### N3193
 - original proposal

## Introduction

In C23, binary and hexadecimal integer literals are prefixed to indicate that
they describe a value using a base other than the default base-10. An unprefixed
number is usually therefore implicitly a decimal number, and is naturally read 
by most human readers in that base.

However, a leading zero digit _implicitly_ serves as a prefix that tells the
lexer to interpret the literal as a value described by base-8, rather than by
base-10. This is "obvious" enough to a human reader who is completely familiar
with the rules, but in practice is unexpected by most users and is also easy for
even an advanced user to miss.

Many users do not expect leading zeroes to be significant and like to use them
as visual padding. This can lead to unexpected value results, or unclear error
messages if they try to "pad" a literal that contains the digits 8 or 9 (though
an error is the better result here). The error can go unnoticed for some time
if by coincidence the user only had a sparse set of values, such as only values
smaller than 8, or all the other "true" decimal literals in use are sufficiently
large as to begin with a non-zero digit. The fact that other languages may allow
these literals to be base-10 adds to the confusion for non-expert users.

[MISRA C 2023][4] (and prior versions) prohibits the use of octal literals entirely
(Rule 7.1, Required) on the grounds that this is so unclear that it is more
likely to be misunderstood than not. There is an exception for a literal zero 
spelled with a single digit, which is technically an octal rather than decimal
literal but the distinction is not meaningful in practice.

### Example

This example is lifted directly from C++ document [p0085r0][3]:

    // The following literals all specify the same number.
    
    int literal_octal_prefered          = 0o52;
    int literal_octal_to_be_deprecated  = 052;
    int literal_decimal                 = 42;
    int literal_hex                     = 0x2A;
    int literal_binary                  = 0b00101010;

This is intended to highlight that the distinction between decimal and the
prefixed octal literal syntax is clearer than the distinction between the
decimal and traditional octal syntaxes.

### Reception

At the January 2024 meeting of WG14, this proposal was discussed. The proposal
to add the new literal syntax prefixed by `0o` or `0O` was well-received with
strong consensus to add to C2y (18 / 1 / 1). The second question, to immediately
make old-style octal literals (of the form `0123`) obsolete _without_ a grace
period in order to force warning messages as early as possible, also received
consensus to continue (11 / 4 / 4).

Alternative spellings for octal literals were discarded, as the `o` syntax was
universally preferred.

The original syntax proposal for prefixed octal escape sequences was rejected.
The Committee instead preferred to see a change along the lines of [N2785][5]
for consistency with C++, which has adopted [the equivalent change][6] into C++23.

N2785 was originally discussed by WG14 during the August 2021 meeting. At this
time WG21 was looking for WG14's opinion before deciding on final adoption, so the
feature had not yet been added to C++. The paper received strong consensus to
adopt directly into C23 (16 / 2 / 3), which did not happen because of scheduling,
but the strong direction and the context of adoption by C++ suggest this change
has strong support for inclusion.

It was observed that there is an existing asymmetry in the language between the
`printf` family of functions and the `strto_l` functions. No decision was reached
about modifying `strto_l`, but the precedent of adding `0b` to the functionality
on C23 suggests that these functions should also be modified for `0o`.

The Committee affirmed that there are **no plans** to fully remove the original
octal `0123` syntax in the foreseeable future.

## Proposal

We propose that a new syntax is added for explicit octal literals, with a new
prefix `0o` or an alternative spelling to mark the beginning of a base-8 
literal. The old syntax should be retained and marked as obsolescent to avoid 
breaking the meaning of existing code.

We **do not** propose that leading-zero ever change meaning to be accepted as
a base-10 literal. This syntax should remain obsolescent, or be fully deprecated
and removed, but cannot be recycled safely.

We _separately_ propose changing escape sequences within literals at this time.
Escape sequences are visually prefixed with a `\` and are therefore much less
subject to this issue. As with the existing hexademical escape syntax, there is
no leading zero on the prefix as this would interfere with a string that 
intentionally contained the nul character. Both of these existing features can
be significantly improved by adopting the "delimited escape sequences" feature
that was added to C++23.

We separately propose allowing the `strto_l` function family to recognize
prefixed octal digit sequences, whereas before they would have returned the
value zero. This is consistent with the change to add support for the binary
literal `0b` and `0B` prefixes in these functions.

We propose adding an equivalent _alternative implementation_ to the formatted
output functions (`printf` family) for prefixed octal numbers, that matches the
behaviour for prefixed hexadecimal and binary numbers. The formatted input
functions are defined in terms of the `strto_l` function family and are
therefore covered by the previous change.

### Choice of prefix

The character `o` is the most obvious choice for the prefix and is in common use
in other languages. 

There was discussion of using `c` or `t` or another prefix instead of `o` at the
January 2024 meeting of WG14. There was universal agreement not to bother pursuing
these alternative spellings due to lack of precedent and unnecessary divergence
from community consensus across languages (the "surprise factor" outweighs any
potential readability improvement).

### Status of zero

Zero "`0`" remains a traditional octal literal, because the rule defining decimal
literals requires them to begin with a non-zero.

This can potentially be changed, but as long as traditional octal literals remain
in the language, the definition of decimal literals has to be complicated in one
way or another. Therefore, for the time being, we leave this as-is to keep the
grammar as simple as possible (there is no obvious gain to over-complicating
the syntax just to move zero).

Any tool that depends on this distinction is hiding a silent logical error.

## Prior Art

A very similar change was proposed for C++ as document [p0085r0][3].
This proposal also added the prefixed form without removing the traditional 
literal syntax, and a new syntax for octal escape sequences in literals.

There does not appear to be a record of this proposal being discussed by WG21
and the change was not adopted.

## Impact

There is no impact to existing code, other than new deprecation warnings if the
user has this functionality enabled in their tool (obsolescence is not a
constraint violation and these warnings are not mandatory).

Causing tools to emit these warnings if they were not already doing so (any tool
aiming to check for MISRA C or similar Guidelines compliance is already warning 
on any use of octal) is considered a **goal of the proposal** and is not a 
compatibility failure.

The proposed spelling is not currently valid in C and therefore use of the new 
octal literal format would not break existing code. Adoption of the prefix does
rule out possible use of `o` or `O` as a suffix in future, but there have been
no proposals to this effect.

## Future directions

If the Standard evolves to incorporate a distinction between deprecation and
obsolescence, we would prefer implicit octal syntax to be marked as **fully
deprecated** in that version of the Standard. This would allow for its eventual
removal, and presumably require a stronger class of warning message (such as
a mandatory warning against uses, rather than the current opt-in for uses of 
obsolescent features).

Octal escape sequences within character or string literals have an outstanding
issue that the end of the sequence is not clear:

    "\1234"  // two characters, \123 followed by 4
    "\1289"  // three characters, \12 followed by 8 and 9

Apart from their variable length, octal escape sequences seem well-understood
compared to integer literals and their use does not seem to be confusing in
practice.

A future proposal could attempt to deprecate this syntax in favour of the new
delimited syntax, or to deprecate octal escapes with fewer than three digits.
We do not attempt to do so here.

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3301][1]. Bolded text is new text when inlined into an existing sentence. 
These changes are not compatible with the words from [p0085r0][3], which
describe a different Standard (C++).

### Integer literals

Within 6.4.5.2 "Integer literals", Syntax, paragraph 1 (the grammar):

Replace the existing _octal-literal_ rule with a new rule:

> _octal-literal:_  
>   _prefixed-octal-literal_  
>   _unprefixed-octal-literal_  

Rename the original _octal-literal_ rule to _unprefixed-octal-literal_ and modify it:

> _unprefixed-octal-literal:_  
>   `0`  
>   `0` `'` <sub>_opt_</sub> _octal-digit-sequence_  

Add a new rule _prefixed-octal-literal_ immediately below _unprefixed-octal-literal_:

> _prefixed-octal-literal:_  
>   _octal-prefix_ _octal-digit-sequence_  

Add a new rule _octal-prefix_ immediately below _binary-literal_:

> _octal-prefix:_ one of  
>   `0o` `0O`  

Add a new rule _octal-digit-sequence_ immediately below _nonzero-digit_:

> _octal-digit-sequence:_  
>   _octal-digit_  
>   _octal-digit-sequence_ `'` <sub>_opt_</sub> _octal-digit_  

Modify paragraph 4:

> A decimal literal begins with a nonzero digit and consists of a sequence of decimal digits. An
> octal literal consists of the prefix **`0o` or `0O`** followed by a sequence of the digits `0` through `7`
> only. A hexadecimal literal consists of the prefix `0x` or `0X` followed by a sequence of the decimal
> digits and the letters `a` (or `A`) through `f` (or `F`) with values 10 through 15 respectively. A binary
> literal consists of the prefix `0b` or `0B` followed by a sequence of the digits `0` or `1`.

Add a new paragraph immediately after paragraph 4:

> An _unprefixed octal literal_ begins with the digit `0` optionally followed by a sequence of the 
> digits `0` through `7` only. Use of an _unprefixed octal literal_ with digits other than `0` is an 
> obsolescent feature.

### Future language directions

Add a new entry between 6.11.3 "External names" and 6.11.4 "Character escape 
sequences":

> **6.11.x  Octal integer literals**
> The use of non-zero octal integer literals without the prefix `0o` or `0O` is an 
> obsolescent feature.

### The `printf` family of functions

Changes to the `printf` family of functions are removed from this proposal. All existing features
will still work with the current `o` specifier, since the `0123` octal syntax is not being removed.
A change to `printf` needs detailed and separate consideration of the compatibility impact.

### The `strto_l` functions

Add a new sentence near the end of 7.24.2.8 paragraph 3:

> If the value of `base` is 2, the characters `0b` or `0B` may optionally precede
> the sequence of letters and digits, following the sign if present. **If the**
> **value of `base` is 8, the characters `0o` or `0O` may optionally precede**
> **the sequence of letters and digits, following the sign if present.**
> If the value of `base` is 16, the characters `0x` or `0X` may optionally
> precede the sequence of letters and digits, following the sign if present.

### The `wcsto_l` functions

Add a new sentence near the end of 7.31.4.2.4 paragraph 3:

> If the value of `base` is 2, the characters `0b` or `0B` may optionally precede
> the sequence of letters and digits, following the sign if present.  **If the**
> **value of `base` is 8, the [wide] characters `0o` or `0O` may optionally precede**
> **the sequence of letters and digits, following the sign if present.**
> If the value of `base` is 16, the wide characters `0x` or `0X` may optionally
> precede the sequence of letters and digits, following the sign if present.

Note: the inconsistency between "the characters" (for base 2) and "the wide characters"
(for base 16) is present in the existing text. The inserted sentence should use whichever
form is correct, and the other existing sentence should probably change to match.

### Delimited escape sequences

These changes are taken directly from N2785, rebased against N3301:

Within 6.4.4 "Universal character names", Syntax, paragraph 1 (the grammar):

Modify the _universal-character-name_ rule:

> _universal-character-name:_  
>   `\u` _hex-quad_  
>   `\U` _hex-quad_ _hex-quad_  
>   `\u{` _simple-hexadecimal-digit-sequence_ `}`  

Add a new rule immediately below _hex-quad_:

> _simple-hexadecimal-digit-sequence:_  
>   _hexadecimal-digit_  
>   _simple-hexadecimal-digit-sequence_ _hexadecimal-digit_  

(NOTE the intent of the changes to the Constraints and Semantics specified by
N2785 appears to have already been incorporated into C23 by [N3124][7].)

Within 6.4.5.5 "Character literals", Syntax, paragraph 1 (the grammar):

Add a new rule _simple-octal-digit-sequence_ immediately below _simple-escape_sequence_:

> _simple-octal-digit-sequence:_  
>   _octal-digit_  
>   _simple-octal-digit-sequence_ _octal-digit_  

Modify the _octal-escape-sequence_ rule:

> _octal-escape-sequence:_  
>   `\` _octal-digit_  
>   `\` _octal-digit_ _octal-digit_  
>   `\` _octal-digit_ _octal-digit_ _octal-digit_  
>   **`\o{` _simple-octal-digit-sequence_ `}`**  

Modify the _hexadecimal-escape-sequence_ rule:

> _hexadecimal-escape-sequence:_  
> `\x` _simple-hexadecimal-digit-sequence_  
> **`\x{` _simple-hexadecimal-digit-sequence_ `}`**  

Note that this change also adds a new syntax for hexadecimal escapes, with
a clear ending delimiter.

## Questions for WG14

Does WG14 want to add the new spelling for base-8 integer literals with an
explicit prefix?  
(WG14 previously voted yes to this question)

Does WG14 want to mark the use of unprefixed base-8 integer literals, apart
from zero itself, as obsolete, without a grace period?  
(WG14 previously voted yes to this question)

Would WG14 like to see a future paper proposing changes to support the `0o`
and `0O` prefixes in the `printf` family of functions?

Does WG14 want to change the behaviour of the `strto_l` function family to
allow them to interpret the new octal prefix, consistently with how they
interpret the `0x` and `0b` prefixes?

Does WG14 want to adopt the "delimited escape sequences" change previously
proposed as N2785 and subsequently adopted into C++23?  
(WG14 previously voted yes to a variant of this question)
(WG14 previously encouraged this feature to be added alongside the octal change)

## Acknowledgements

Huge thanks to Joseph Myers for detailed and thorough review of previous versions.

## References

[C2y latest public draft][1]  
[Wikipedia on use of octal][2]  
[C++ proposal P0085R0][3]  
[MISRA C 2023][4]  
[N2785 Delimited escape sequences][5]  
[C++ proposal P2290R3][6]  
[N3124 Aligning Universal Character Names Constraints with C++][7]  


[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3301.pdf
[2]: https://en.wikipedia.org/wiki/Octal#In_computers
[3]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0085r0.html
[4]: https://misra.org.uk/product/misra-c2023/
[5]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2785.pdf
[6]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2290r3.pdf
[7]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3124.pdf
