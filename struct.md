# struct-z80 - Structured Programming in Z80 Assembly Language

In assembly language, program logic is obscured by the way it gets expressed in terms of state flags and branches. Consider this simple example of inverting a binary value which we'll first express by using a structured language:

```
let a = input;
if (a == 0) {
  a = 1;
} else {
  a = 0;
}
let result = a;
```

In assembly language, we could do the same thing by loading the accumulator with the input and comparing it with zero. This may or may not set the zero flag. If the flag is false (test fails) we conditionally jump over the code in the "then" section and go straight to the "else" section. If the test succeeds we execute the "then" section but now we want to skip over the "else" section with an unconditional branch to the end of the if statement (i.e. to "endif").

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

So in a typical situation, we have two branches and at least two labels (the thenLabel is not really needed in this case). When logic gets nested, this can lead to difficult to read assembly code. Surely a more natural way to express this code would be a little more like this?

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

In this code we are using macros. I am using an underscore in front to distinguish these macros from other uses of these words. You'll notice that we have no explicit branches and no labels. Each `_if`, `_else` and `_endif` macro expands out into code that is very similar to the hand-written assembly code.

```
.macro _if, flag
    jr flag, L_%%M
    jp $              ; placeholder jump to _else or _endif
    STRUC_PUSH $
L_%%M:
.endm

.macro _else
    jp $              ; placeholder jump to _endif
    JUMP_FWD
    STRUC_TOP .set $  ;reuse top of stack
.endm

.macro _endif
    JUMP_FWD
    STRUC_POP
.endm
```

Note: `L_%%M` is a local label which is unique to each macro invocation. \$ is the current assembler address.

The way this works is the `_if` sets up a jump on condition flag which, if the test fails, branches to an `_else` or the `_endif` depending on which occurs first.

The problem to solve is that `_if` cannot know where the `_else` or `_endif` will occur so it writes the jump instruction with a placeholder for the jump address. It then pushes the address of this jump on the assembler stack so it can be found again later.

When an `_else` is encountered, it looks on the stack for the address of the last `_if` occurrence. It goes back and fills in the placeholder address in the `_if` to point to the `_else` code. It then writes another jump instruction with a placeholder address to point to the `_endif`. Once again, it pushes the address of this jump onto the assembler stack.

When an `_endif` is encountered, it looks on the stack for the address of the last jump instruction pushed on the stack (it will be either an `_if` or `_else` occurrence). Then it goes back and fills in the placeholder address of the jump instruction to point to itself. **This means that the** `_else` **macro is completely optional** and that you could just use `_if` and `_endif` without an `_else` if you wanted to. For example:

```
ld a,(input)
cp 0
_if z
  ld a, 1
_endif
ld (result),a
```

This rewriting magic is achieved by the `JUMP_FWD` macro which saves the current assembler address `$` in a temporary variable and then uses the `org` directive to move the current assembler address back towards the code it has already written. By backing up the assembler's address pointer it can then rewrite the branch placeholder address to the value of the saved value. It then restores from the saved assembler address and continues.

It will probably take you a few reads through to fully understand this logic, it's fiddly but not too complicated. If you are already familiar with the way the Forth language implements its control structures then you may grasp it immediately.

Now, with these macros in hand, you can easily write nested if...else...endif logic in your Assembly code without using a single jump or even a label!

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

Note: I'm using `nop` here to stand in for any Z80 instruction

## Avoiding deep nesting by using Switch

When you have a lot of alternatives to deal with in your conditional logic, `_if` ... `_else` ... `_endif` can quickly become cumbersome and hard to read. For example, consider the following scenario in a structured language:

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

Without some way to chain these "if" statements we end up with a heavily nested sequence like this:

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

This is pretty ugly but you can solve this nesting by using the `_switch` macro.

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

The way `_switch` works is that each case is tested in turn and, if a condition is met, the `_case` immediately following the test is executed. After that it jumps to the `_endswitch`.

If the condition fails it falls through to the next case and so on. If none of these cases execute then it falls through to the final "default" case just before the `_endswitch`.

The implementation of the switch macro is as follows:

```
.macro _switch
    jr L_%%M
    jp $            ; placeholder jump to endswitch
    STRUC_PUSH $
L_%%M:
.endm

.macro _case, flag
    jr flag, L_%%M
    jp $            ; placeholder jump to endcase
    STRUC_PUSH $
L_%%M:
.endm

.macro _endcase
    jp STRUC_2 - 3  ; jump to placeholder jump to endswitch
    JUMP_FWD
    STRUC_POP
.endm

.macro _endswitch
    JUMP_FWD
    STRUC_POP
.endm
```

_Copyleft @ 2019 John Hardy, ALL WRONGS RESERVED_
This software is released under the GNU public license 3.0

_Contact me: [jh@lagado.com](mailto:jh@lagado.com) GitHub: [jhlagado](http://github.com/jhlagado) Twitter: [@jhlagado](https://twitter.com/jhlagado)_
