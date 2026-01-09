    Proposal for C2y
    WG14 3355

    Title:               Named loops, v3
    Author, affiliation: Alex Celeste, Perforce
    Date:                2024-09-18
    Proposal category:   New feature
    Target audience:     Compiler implementers, users

## Abstract

The loop exit keywords `break` and `continue` in C only work with the most
immediate enclosing loop or `switch` structure. This proposal introduces a
named loop syntax consistent with other languages which allows breaking out
of control structures outside of the immediately enclosing one, by passing the
target loop of the jump by name.

-------------------------------------------------------------------------------

# Named loops, v3

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  N3355
    Revises:      N3268
    Date:         2024-09-18

## Summary of Changes

### 3355
 - rebase against N3301
 - address some Reflector commentary

### N3268
 - rebase wording on C2y draft
 - remove ambiguity that appeared to allow `break` from within `if`
 - discuss the choice for label position

### N3195
 - original proposal, but refer back to [N2859][1]
 
## Introduction

A facility for breaking out of outer loops that enclose an immediately-nested
one is a frequently-requested feature for C. This feature is found in other
languages and is a handy way to make clear where the intended result of a loop
exiting jump is actually intended to be.

Much of the rationale for the feature itself, the ability to break out of more
than one nested control structure while sticking to descriptive, structured jump
statements rather than falling back to `goto`, was already discussed in N2859
by Steenberg. It is widely accepted that the need to be able to _do_ this is
widely needed - any example with two nested loops can easily be constructed -
but is not necessarily easy to express in a way that is pleasing to the C user
(wither reader or writer), and most of the options significantly reduce the
readability of the loop over simply

    for (int i = 0; i < IK; ++ i) {
      for (int j = 0; j < JK; ++ j) {
        
        if (cond)
          just_give_up; // <- this is difficult to make elegant
        
      }
    }

Steenberg proposed `break break`, which was rejected by the Committee on 
multiple different grounds (it has no precedent; it is context-dependent; it is
subjectively ugly to most of the reviewers; it is harder to read; etc.). In the
past, previous proposals have also suggested passing a constant numeric
parameter to `break` or `continue`, which suffers from essentially all of the
same problems - it has no precedent, and is completely context-dependent, which
means it cannot reliably be used in combination with macros because intervening
lines of code can arbitrarily change the jump target.

### The problem with a depth operand

`break break;` is essentially the same proposal as `break N;`, but encoded in
the repetition of the `break` keyword rather than as an expression operand. The
core problem of the two proposals is therefore identical:

    for (int i = 0; i < IK; ++ i) {
      for (int j = 0; j < JK; ++ j) {
        
        if (cond)
          break 2; // clear enough for now
          
        // (several pages of in and out)
        if (cond)
          break 3; // ...how much do you trust the indent level?
        // (several pages of in and out)
        
      }
    }

The first problem is simply that because it doesn't _name_ the target, there is
immediate loss of clarity when the loop to be existed goes more than a couple of
lines away. The compiler has no clear way to bind the `break` to the loop or
`switch`, and will not be able to warn if the levels start to mismatch, which 
may not be found for some time if on a less-common path (this is likely error
handling). The human reader will have a hard time working it out if a lot of 
mouse-wheeling up and down is needed, and it relies on clear formatting to be
readable.

More insidiously, say we have a macro that abstracts block control (using the
Claire's Device for illustration, may also be used for e.g. `nogc` or `with`
or any number of other inventive schemes):

    #if CONFIG_1
    #define NEVER if (0)
    #else
    #define NEVER while (0)
    #endif
    
    for (int i = 0; i < IK; ++ i) {
      for (int j = 0; j < JK; ++ j) {
        
        // NEVER is equally correct either way...
        NEVER funclet: {
          some_work ();
          break 2;  // ...but this isn't!
        }
          
        if (cond)
          goto funclet;
        else
          goto junklet; // ...
        
      }
    }

That `break 2` depends entirely on exactly one definition of the funclet.

More commonly control structure macros build on something like a `BEFORE_AFTER`
block, which can be implemented with `for` loops:

    // one?
    #define M_BEFORE_AFTER(BEFORE, AFTER)  \
      for(int _b = (BEFORE, 1); _b; (AFTER, _b = 0))
      
    // or two? (if BEFORE needs to admit a decl)
    #define M_BEFORE_AFTER(BEFORE, AFTER)  \
      for (_Bool M_ONCEFOR_enter = 1; M_ONCEFOR_enter ; AFTER )  \
        for (BEFORE ; M_ONCEFOR_enter ; M_ONCEFOR_enter = 0)
          /* { body } */

The number of loops hidden here _shouldn't_ matter to the user, because they
aren't serving as loops, they are the assembly language opcodes for control
structure metaprogramming and the effect here does not involve anything that
the user interacts with _as_ a loop. Ideally the user shouldn't have to either
think to jump out of this block - which is probably being used as the basis
for, say, an autorelease pool, or a monadic `optional` handler. They certainly
don't want to need to know how many and which "opcodes" it uses to build its
braced structure.

Even in "normal" code that doesn't use such macros, using an explicit or 
implicit depth argument still harms readability because of `switch`, which
binds to a `break` but _not_ to a `continue`, creating an inconsistency even
when all levels of the control structures are controlled by builtin keywords.
This is potentially confusing (and is an existing flaw in the language where
every `break` or `continue` has an implicit `1`).

Therefore a `break 2` or `break break` _breaks_ in combination with this
technique.

### Named operand

In contrast, if loops can have names:

    outer_ij:
    for (int i = 0; i < IK; ++ i) {
      for (int j = 0; j < JK; ++ j) {
        
        if (cond)
          break outer_ij; // this is fine
        
        NEVER funclet: {
          some_work ();
          break outer_ij; // this is also fine
        }
          
        autorelease_pool {
          break outer_ij; // this is fine too
        }
        
      }
    }

All jumps clearly mark which loop they want to terminate (or potentially to
`continue`, which has almost exactly the same concerns).

The existing asymmetry between loops and `switch` is also improved:

    selector:
    switch (n) {
      
      for (int i = 0; i < IK; ++ i) {
        break selector; // break the switch from a loop!
      }
      
    }
    
    loop:
    for (int j = 0; j < JK; ++ j) {
      switch (n) {
       
        break loop; // break the loop from a switch!
        continue loop; // this was valid anyway, 
                       // but now it's symmetrical
      } 
    }

## Alternatives

There are three obvious alternatives using existing control structures: 
repetition, `return`, and `goto`.

### Repetition

This example was given by Steenberg:

    for (int i = 0; i < n; ++ i) {
      for (int j = 0; j < n; ++ j) {
        if (something (i, j))
          break;
      }
      
      if(j < n)
        break;
    }

The compiler can handle this well enough, but this makes the human reader do a
lot of work to understand the intent - jumping out of one loop and then checking
again is not what this code _actually means_. It simply adds cognitive and 
maintenance load because the user wanted to avoid the next option:

### `goto`

    for (int i = 0; i < n; ++ i) {
      for (int j = 0; j < n; ++ j) {
        if (something (i, j))
          goto end;
      }
    }
    end:

This code is clear enough, but the `goto` is socially problematic. The social
complaint is partially valid - although this _use_ of `goto` is structured,
nothing forces the user to put it at the end of the loop in this way. This is
an unstructured jump that relies entirely on user discipline, whereas naming
the loop target binds tightly to the control structures themselves and works
to encourage clearer usage directly.

This pattern probably also has a high "surprise factor" when used to emulate
`continue`.

### `return`

There isn't much to say here except that `return` can obviously be used as a
jump across any number of loops and other structures (though it cannot really
imitate `continue` easily), which is a common enough pattern in C++:

    auto const loop = [&] {
      for (int i = 0; i < n; ++ i) {
        for (int j = 0; j < n; ++ j) {
          
          if (cond)
            return;
            
        }
      }
    };
    
    loop ();

This is just about usable in a language with lambdas, but does not provide the
full functionality. Rewriting C this way with fully-separated function bodies 
is probably the least readable option much of the time.

### Putting the label somewhere else

This feature was discussed by WG14 at the January 2024 meeting and met with
directional consensus to continue. However, a common piece of feedback was that
the proposed position of the label was unclear:

    loop1:
    for (int i = 0; i < n; ++ i) {
      if (cond)
        continue loop1; // <-- this may _look like_ it jumps back to
    }                   //     all the way back to iteration 0 ??
    
    loop2:
    for (int i = 0; i < n; ++ i) {
      loop3:
      for (int j = 0; j < n; ++ j) {
        
        if (cond)
          continue loop3; // <-- possibly visually misleading
      }
    }

Several members opined preference for a different syntax that places the label
between the header and the brace:

    for (int i = 0; i < n; ++ i) loop4: {
      ...
    }

We agree that this is more visually elegant, but it faces the problem of conflicting
with _all_ existing Prior Art - every other major language with braced syntax seems
to place the label before the statement header. If C chooses the more "beautiful"
syntax, it introduces uncertainty by arbitrarily meaning something different from
the control flow in Java, JS, _et al._

In edge cases this can even change the meaning:

    loop5:
    for (int i = 0; i < n; ++ i) // no brace
      loop6:
      for (int j = 0; j < n; ++ j) {
        if (cond)
          continue loop6; // <-- this would DO SOMETHING DIFFERENT
      }                   //     in C vs Java et al.

We consider that this break with precedent would be too confusing and believe
that existing languages have effectively chosen the aesthetic for us at this stage.
While C is not bound to be compatible with any other language, an _arbitrary_
break with praxis is not user-friendly.

The difference between a skip-iteration and a retry is ultimately still communicated
by which keyword is implementing the jump: `goto label` is a full retry, and
`continue` is never a full retry. We think users are familiar enough with the
existing meaning of `continue` (and `break`) not to be confused by this in practice.

## Prior Art

Named loops also have a distinct advantage of having _substantial_ prior art
across multiple other programming languages. C is not bound by any other language
but to have a control feature behave in exactly the same way as precedent set by
the wider Community is extremely good for readability and lowers the surprise
factor. The idiom has been proven to work well in practice, and there is no good
reason for C to diverge from a model the rest of the programming language 
meta-community seems to have found clearest.

For instance, in [Rust][4]:

    #![allow(unreachable_code, unused_labels)]
    
    fn main() {
        'outer: loop {
            println!("Entered the outer loop");
    
            'inner: loop {
                println!("Entered the inner loop");
    
                // This would break only the inner loop
                //break;
    
                // This breaks the outer loop
                break 'outer;
            }
    
            println!("This point will never be reached");
        }
    
        println!("Exited the outer loop");
    }

In [Javascript][3] (see also [ECMA-262][8] 14.8, 14.9, 14.13):

    let str = '';
    
    loop1: for (let i = 0; i < 5; i++) {
      if (i === 1) {
        continue loop1;
      }
      str = str + i;
    }

In [Java][2] (see also [JLS][7] 14.7, 14.15, 14.16):

    search:
    for (i = 0; i < arrayOfInts.length; i++) {
        for (j = 0; j < arrayOfInts[i].length;
             j++) {
            if (arrayOfInts[i][j] == searchfor) {
                foundIt = true;
                break search;
            }
        }
    }

The proposed syntax matches all three of these languages exactly (modulo Rust's
slightly different syntax for the label name itself). This is therefore almost
certainly the least confusing and most user-friendly option to imitate.

### Teachability

(i.e. consistency with the Prior Art of C itself)

The advantage of the existing structured jump operators is that the user does not
need to think about where the jump _to_, whereas with `goto`, knowing where the
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

Therefore we believe this feature addition generally helps both teachability and also
localized reasoning about program behaviour. It also helps communicate intent more
effectively that using the equivalent `goto` (which can _implement_ the same jump,
but does not tell the reader that "I want a forward-and-outward, block-ending jump" in
the same way that `break` does).

## Proposed wording

The proposed changes are based on the latest public draft of C2y, which is
[N3301][0]. Bolded text is new text when inlined into an existing sentence.

### Labels

Add two new paragraphs to 6.8.2 "Labeled statements" in Semantics, after 
paragraph 4:

> If _statement_ is an iteration or `switch` statement, that statement is _named_
> by the label _label_. If _statement_ is a _labeled-statement_, the statement 
> _named_ by this label is the same statement _named_ within the nested
> _labeled-statement_, if any. <sup>footnote</sup>
>
> <small>footnote) a statement may therefore be _named_ by more than one label.</small>
>
> **EXAMPLE**  Only an iteration or `switch` statement may be _named_ by a
> label, by appearing as its immediate syntactic operand:
>
>     loop:
>     for (int i = 0; i < IK; ++ i) { // this for is named by loop:
>       ...
>     }
>     
>     braces:
>     {        // no statement is named by braces:
>       for (int i = 0; i < IK; ++ i) {
>         ...
>       }
>     }

(We do not list the kinds of iteration or selection statements here. If for some
reason the list would change, this should change implicitly and automatically.)

Add forward references to `break` and `continue`:

> **Forward references:** the `goto` statement (6.8.7.2), **the `continue` statement**
> **(6.8.7.3), the `break` statement (6.8.7.4),** the `switch` statement (6.8.5.3).

Modify the last sentence of 6.8.3 "Compound statement", paragraph 2:

> A label **that is not followed by another label or an _unlabeled-statement_**
> shall be translated as if it were followed by a null statement.

Add a new paragraph after paragraph 2:

> A statement that follows one or more labels without any other intervening
> _block-items_ is _named_ by those labels, if it would be eligible to be _named_
> by the labels of a _labeled-statement_.

### Jumps

Modify 6.8.7 "Jump statements", Syntax, paragraph 1:

> _jump-statement:_  
>   `goto` _identifier_ ;  
>   `continue` **_identifier_<sub>opt</sub>** ;  
>   `break` **_identifier_<sub>opt</sub>** ;  
>   `return` _expression_<sub>opt</sub> ;  

### `continue`

Add a new paragraph to 6.8.7.3 "The `continue` statement" in Constraints, after 
paragraph 1:

> A `continue` statement with an identifier operand shall appear within an iteration
> statement _named_ by the label with the corresponding identifier.

Modify the first sentence of paragraph 2, removing the reference to "innermost":

> A `continue` statement causes a jump to the loop-continuation portion of **an**
> enclosing iteration statement; that is, to the end of the loop body.

Add a new paragraph after paragraph 2:

> If the `continue` statement has an identifier operand, the jump is to the
> loop-continuation of the iteration statement named by the label with the
> corresponding identifier. Otherwise, the jump is to the loop-continuation
> of the innermost enclosing iteration statement.

Add an example:

> **EXAMPLE** In the following code, `continue` only jumps to the loop-continuation
> of the inner nested loop, whereas `continue outer` jumps to the loop-continuation
> of the outermost loop:
>
>     outer:
>     for (int i = 0; i < IK; ++ i) {
>       for (int j = 0; j < JK; ++ j) {
>         
>         continue;       // jumps to CONT1
>         
>         continue outer; // jumps to CONT2
>         
>         // CONT1
>       }
>       // CONT2
>     }

### `break`

Add a new paragraph to 6.8.7.4 "The `break` statement" in Constraints, after 
paragraph 1:

> A `break` statement with an identifier operand shall appear within a `switch`
> or iteration statement _named_ by the label with the corresponding identifier.

Modify paragraph 2, removing the reference to "innermost" and actually explaining
_how_ the loop is broken out from (!):

> A `break` statement terminates execution of **a** `switch` or iteration statement**,**
> **as if by jumping to a label immediately following it in the surrounding scope**
> **using `goto`.**

Add a following paragraph:

> Therefore, the following statements are equivalent:
>
>     while (/* ... */) {            while (/* ... */) {
>       break;                         goto there;
>     }                              }
>     // break jumps here            there:

Add a following paragraph:

> If the `break` statement has an identifier operand, the jump exits the
> `switch` or iteration statement _named_ by the label with the corresponding
> identifier. Otherwise, the jump exits the innermost enclosing `switch` or
> iteration statement.

Add an example:

> **EXAMPLE** In the following code, `break` only exits
> the switch, whereas `break outer` exits the enclosing loop as well:
>
>     outer:
>     for (int i = 0; i < IK; ++ i) {
>       switch (i) {
>         case 1:
>           break;       // jumps to CONT1
>         case 2:
>           break outer; // jumps to CONT2
>       }
>       // CONT1
>     }
>     // CONT2

## Questions for WG14

Does WG14 want to add named loops to C using the proposed syntax and wording?

## Acknowledgements

Huge thanks to Joseph Myers for detailed and thorough review of previous versions.

## References

[C2y public draft][0]  
[N2859][1]  
[Named loops in Java][2]  
[Named loops in Javascript][3]  
[Named loops in Rust][4]  
[Claire's Device][5]  
[M_BEFORE_AFTER in CRFI-4][6]  
[Java language specification][7]  
[ECMAScript 2023][8]  

[0]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3301.pdf
[1]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2859.pdf
[2]: https://docs.oracle.com/javase/tutorial/java/nutsandbolts/branch.html
[3]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label
[4]: https://doc.rust-lang.org/rust-by-example/flow_control/loop/nested.html
[5]: https://web.archive.org/web/20210203022142/http://www.clifford.at/cfun/cliffdev/
[6]: https://github.com/C-Requests-for-Implementation/crfi-examples/blob/main/crfi-4-example.c#L48C1-L51C21
[7]: https://docs.oracle.com/javase/specs/jls/se18/jls18.pdf
[8]: https://ecma-international.org/wp-content/uploads/ECMA-262_14th_edition_june_2023.pdf
