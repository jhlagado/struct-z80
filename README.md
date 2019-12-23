# struct-z80 - Structured Programming in Z80 Assembly
```
title: Structured Programming in Z80 Assembly 
published: true
description: Using assembler macros to write high level code
tags: Z80, macros, assembler, structured programming
```

One of the great pains of writing assembly language for old-school microprocessors such as the Z80 is the complexity of implementing algorithms due to the lack of high-level control and looping structures in machine code. All you have are jumps and labels and there's really no exaggeration to the claim that [GOTOs are considered harmful](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf) 

...at least to your state of mental well-being! ;-) 

This lack of structuring is what often ends up driving programmers toward languages like C which people often think of as "low level", "assembler++" and "close to the metal". On 8-bit CPUs however this far from true and C adds a lot of overhead to your machine cycle and memory cell constrained code. This is why assembler is still the tool of choice for 8-bit programming despite also being a major source of frustration.

Macros are a huge boon to writing assembly. Recently I starting using a set inspired by Garth Wilson and Dave Keenan which enabled me to write something close to structured code like C in Assembler.

See:

[Program Structures with Macros](http://wilsonminesco.com/StructureMacros/)

[Adding Structured Control Flow to Any Assembler](http://dkeenan.com/AddingStructuredControlFlowToAnyAssembler.htm)

Both authors were heavily influenced by the [Forth programming language](https://www.forth.com/forth/) and the way it introduced high-level structured programming concepts to low level programmers years before systems languages like C and Pascal became commonplace.

The examples I'm giving here were written using the asm80 macro system but I'm sure they could be easily adapted to your assembler of choice's macro syntax.

See:

[Asm80](https://www.asm80.com/)

[Asm80: Macros](https://maly.gitbooks.io/asm80/macros.html)

## Step one: Create a stack with assembler variables.

This first bit is a bit hacky but you can implement a stack structure by using a bucket brigade of assembler variables. This is necessary to allow for nesting structures in your code. I've made my stack ten levels deep but you may decide to make this stack deeper which is easy to do.

```
STRUC_COUNT .set 0
STRUC_10 .set 0
STRUC_9 .set 0
STRUC_8 .set 0
STRUC_7 .set 0
STRUC_6 .set 0
STRUC_5 .set 0
STRUC_4 .set 0
STRUC_3 .set 0
STRUC_2 .set 0
STRUC_TOP .set 0

.macro STRUC_PUSH, arg
    STRUC_COUNT .set STRUC_COUNT + 1
    STRUC_10 .set STRUC_9
    STRUC_9 .set STRUC_8
    STRUC_8 .set STRUC_7
    STRUC_7 .set STRUC_6
    STRUC_6 .set STRUC_5
    STRUC_5 .set STRUC_4
    STRUC_4 .set STRUC_3
    STRUC_3 .set STRUC_2
    STRUC_2 .set STRUC_TOP
    STRUC_TOP .set arg
.endm

.macro STRUC_POP
    STRUC_COUNT .set STRUC_COUNT - 1
    STRUC_TOP .set STRUC_2
    STRUC_2 .set STRUC_3
    STRUC_3 .set STRUC_4
    STRUC_4 .set STRUC_5
    STRUC_5 .set STRUC_6
    STRUC_6 .set STRUC_7
    STRUC_7 .set STRUC_8
    STRUC_8 .set STRUC_9
    STRUC_9 .set STRUC_10
    STRUC_10 .set 0
.endm
```

Also we need a utility macro _JP_FORWARD which we use to go back and rewrite jump addresses for addresses we don't resolve until later.

```
.macro _JP_FORWARD
    CUR_ADR .set $
    org STRUC_TOP - 2
    dw CUR_ADR
    org CUR_ADR
.endm

```

## Step 2: implement _if, _else and _endif macros:

In assembly language, program logic is obscured by the way it is expressed in terms of state flags and branches. Consider this simple example of inverting a bit value which we'll first express in a structured language:

```
let a = input;
if (a == 0) {
  a = 1;
} else {
  a = 0;
}
let result = a;
```

In assembly, you would load the accumulator with the input and compare it with zero. This may set the zero flag. If the test fails could branch conditionally over the code in the "then" clause and go to the "else" clause. If the test succeeded it, it would execute the "then" clause but then we want to skip over the "else" clause with an unconditional branch to the end of the if statement (i.e. the "endif" label).

```
    ld a,(input)
    cp 0
    jp nz, elseLabel
thenLabel:
    ld a, 1
    jp endIfLabel
elseLabel:
    ld a, 0
endIfLabel:
    ld (result),a        
```

So in a typical situation, we have two branches and at least two labels (the thenLabel is not really needed in this case). When logic gets nested, this can lead to very difficult to read assembly code. Surely a more natural way to read this code would be like this:

```
ld a,(input)
cp 0
_if z
  ld a, 1
_else    
  ld a, 0
_endif
ld (result),a        
```

In this arrangement we have no explicit branches and no labels to confuse the reader. Each statement `_if`, `_else` and `_endif` is an assembler macro which expands out into code very similar to the original assembly (I used an underscore a the front to distinguish it from any similar names in the code). The `_if` macro contains a conditional jump to the "else" clause if the condition is not met. Because the `_else` macro always "bookends" with `_if`, it begins with a jump that allows execution from the "then" clause to jump over the "else" clause and jump to the "endif" point. 

```
.macro _if, cond
    jr cond, L_%%M
    jp $
    STRUC_PUSH $
L_%%M:
.endm

.macro _else
    jp  $
    _JP_FORWARD
    STRUC_TOP .set $          ;reuse TOS
.endm

.macro _endif
    _JP_FORWARD
    STRUC_POP
.endm
```

NOTE: `L_%%M` is a local label which is unique to each macro invocation. $ is the current assembler address. 

The way this works is the `_if` sets up a jump on conditional which, if the test fails, branches to an `_else` or the `_endif` depending on which occurs first. The problem is that `_if` cannot know when the `_else` or `_endif` will occur so it writes the opcode for the jump and a placeholder value for the jump address. It then pushes the address of this jump so it can find it again later.

When an `_else` is encountered, it looks on the stack for the address of the last `_if` occurrence. It goes back and fills in the correct address of the _else. It then outputs another jump, this time to the _endif, once again uses a placeholder address to be updated later and pushes the address of this jump on to the stack.

When an `_endif` is encountered, it looks on the stack for the address of the last jump instruction pushed on the stack (either an `_if` or `_else` occurrence). It goes back and fills in the address of the branch to point to itself. This means that the `_else` macro is optional and you could just use `_if` and `_endif` without `_else` if you wanted to.

This rewriting magic is achieved by getting the assembler to save the current assembler address $ in a temporary variable and then use the org directive to push the assembler address back towards the code it has already written. By backing up the address pointer it can rewrite the branch placeholder address to the temporary variable. It then restores the assembler address and continues. 

It will probably take you a few reads to fully understand this logic but I reckon it's extremely elegant. If you are familiar with the way that the Forth language handles control structures then you may grasp it immediately.

Now, with these macros in hand, you can easily write nested if...else...endif logic in your assembly code without using a single JP or even a label!

```
ld A,1
or a     ; test for the zero condition just before the _if
_if z        
    cmp $10  ; test for the not carry condition just before the _if
    _if nc  
        nop
        nop
    _else
        nop
        nop
        nop
    _endif
_else
    nop
    nop
    nop
_endif
```

NOTE: I'm using `nop` here to stand it for any Z80 instruction

## Switch case

When you have a lot of alternatives to deal with `_if` ... `_else` ... `_endif` becomes nested and harder to read. 

For example, consider the following scenario in a structured language:

```
let a = input
if (a == 'a') {
    a = 'A';
} elseif (a = 'b') {
    a = 'B';
} elseif (a = 'c') {
    a = 'C';
} else {
    a = 'D';
}
```
Without some kind way of chaining "if" statements we end up with a heavily nested sequence like this:

```
ld A, input
cp 'a'
_if 
    ld A,'A'
_else
    cp 'b'
    _if 
        ld A,'B'
    _else
        cp 'c'
        _if 
            ld A,'C'
        _else
            ld A,'D'
        _endif
    _endif
_endif
```

One solution to this is to implement an approach that is similar to C's "switch" statement. 

```
ld A, input
_switch

    cp 'a'          ; test for 'a'
    _case z
        ld A,'A'
    _endcase

    cp 'b'          ; test for 'b'
    _case z
        ld A,'B'
    _endcase
    
    cp 'c'          ; test for 'c'
    _case z
        ld A,'C'
    _endcase

    ld A,'D'        ; default case

_endswitch
```

With this set of macros between `_switch` and `_endswitch`, each case is tested in turn and, if the condition is met, the immediately following `_case` is executed followed by a jump to `_endswitch`. If the condition fails then it falls through to the next test and so on. If none of the cases execute then execution falls through to the "default" case just before the `_endswitch`.

The implementation is as follows:

```
.macro _switch
    jr L_%%M
    jp $                    ; jump to endswitch
    STRUC_PUSH $
L_%%M:
.endm

.macro _case, cond
    jr cond, L_%%M
    jp $                    ; jump to endcase
    STRUC_PUSH $
L_%%M:
.endm

.macro _endcase
    jp STRUC_2 - 3          ; jump to exit
    _JP_FORWARD
    STRUC_POP
.endm

.macro _endswitch
    _JP_FORWARD
    STRUC_POP
.endm
```

## Loops

You can also do loops pretty easily as well with `_do`...<test> `_while` cond ... `_enddo` and `_do` ... <test> `_until` cond:

```
.macro _do
    STRUC_PUSH $
.endm

.macro _while, cond
    jr cond, L_%%M
    jp $
    STRUC_PUSH $
L_%%M:
.endm

.macro _enddo
    jr STRUC_2
    _JP_FORWARD
    STRUC_POP
    STRUC_POP
.endm
```

For example:

```
ld B, 3
_do
    dec B
_while nz
    nop
_enddo
```

and:

```
ld B, 5
_do
    ld A,3
    _do
        nop
        dec A
    _until Z
    dec B
_until
```

and finally:

```
_do
    nop
_forever
```

Note: in some assemblers (unfortunately not asm80) you can stop the stack operations from adding noise to your listings by using an assembler directive like:
```
.list "on"
```
and
```
.list "off"
```

So there you have it, a pretty painless way to improve the readability of your code and increase your productivity as an assembly language programmer (you think I'm exaggerating but, no, I genuinely believe this). The best thing is that if you examine the final generated machine code you'll see that this code does not look weird and doesn't really add overhead to the way you would have done it natively. 

There are possible optimisations of course. For example, you could calculate whether the  jumps should be near or far jumps and replace JP with JR in those cases. You could use a macro to calculate this. 

A repo of all the macros discussed here can be found here:

[struct-z80](https://github.com/jhlagado/struct-z80)

Anyway, if you do give it a try let me know how it goes! 

*Copyleft @ 2019 John Hardy, ALL WRONGS RESERVED*

*Contact me: [jh@lagado.com](mailto:jh@lagado.com) GitHub: [jhlagado](http://github.com/jhlagado) Twitter: [@jhlagado](https://twitter.com/jhlagado)*
