    Proposal for C2y
    WG14 xxxx

    Title:               No commas in subscripts
    Author, affiliation: Alex Celeste, Perforce
    Date:                2023-11-30
    Proposal category:   Constraint restriction
    Target audience:     Compiler implementers, users

## Abstract

The use of a comma expression as the toplevel expression operand to a subscript
serves no obvious purpose. We propose either changing the syntax to make this a
hard error, or obsoleting the form.

-------------------------------------------------------------------------------

# No commas in subscripts

    Reply-to:     Alex Celeste (aceleste@perforce.com)
    Document No:  Nxxxx
    Revises:      (n/a)
    Date:         2023-11-30

## Summary of Changes
### Nxxxx
 - original proposal
 
## Introduction

C allows any whole expression to be the right-hand operand to a subscript.

There is no syntactic ambiguity to this, because the expression is completely
enclosed by the square brackets of the subscript. However, there is also no
clear purpose:
