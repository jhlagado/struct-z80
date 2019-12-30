# struct-z80 - Structured Programming in Z80 Assembly Language


Looping is another one of those pain points in assembly language and potential source of bugs.

Consider following code written with a structured language:

```
let a = 0;
while (a < 10) {
    ; do something here
    a++;
}
```

In conventional assembly code you would normally use a conditional branch to a label

```
    ld A, 0
LOOP1:
    cp 10
    jp nc, LOOP_EXIT
    nop                 ; do something here
    inc A
    jp LOOP1
LOOP_EXIT:
```

In this case I'm testing for the _opposite_ condition to the structured version. I'm deciding whether to exit the loop rather than to stay inside it. I then do some work before incrementing the counter and jumping back to do the test.

This code with two jumps and two labels is somewhat confusing and the situation only gets more confusing when loops are nested within one another.

The structured macro way to do the same thing is to use the `_do` macro.

```
ld A, 0
_do
    cp 10           ; test
_while c
    nop             ; do something here
    inc A
_enddo
```

The code between `_do` and `_enddo` is repeated and a test is conducted just before the `_while` which will jump to the `_enddo` if the test fails.

Sometimes it's more convenient to terminate on the success of a test.

```
ld A, 0
_do
    cp 10           ; test
_until ncs
    nop             ; do something here
    inc A
_enddo
```

An alternative to terminating a loop is to `_continue` a loop, that is, to conditionally jump to the start of a loop.

```
ld B, 0
_do
    ld A,B
    and $01          ; test
    _continue nz
                     ; get here only on even values
    inc B            ; test
_until z
_enddo
```

Note: both `_while`, `_until` are optional and may appear zero or more times inside a loop.

Loops can be nested easily as long as the values of counter variables are preserved.

```
ld A, 0
_do
    cp 2       ; test
_while z
    push AF
    ld A, 0
    _do
        cp 5   ; test
    _while z

        nop    ; do something here
        inc A
    _enddo
    pop AF
    inc A
_enddo
```

A loop can also built using the the Z80's own `djnz` instruction. This assumes that the counter value is stored in the B register which is decremented automatically each time through the loop. When B reaches zero the loop terminates.

```
ld B, 10
_do
    nop             ; do something here
_djnz
```

Note: `_while` and `_until` work inside `_do`...`_djnz` loops exactly the same way as they do in `_do`...`_enddo` loops.

The implementation of macros for looping are as follows:

```
.macro _do
    jr L_%%M
    jp $                    ; placeholder jump to enddo
    DLOOP_PUSH $
L_%%M:
.endm

.macro _while, flag
    jr flag, L_%%M
    jp DLOOP_TOP - 3         ; jump to jump to enddo
    DLOOP_FWD                ; needed?
L_%%M:
.endm

.macro _until, flag
    jp flag, DLOOP_TOP - 3  ; jump to jump to enddo
    DLOOP_FWD               ; needed?
.endm

.macro _break
    jp DLOOP_TOP - 3        ; start of loop
.endm

.macro _continue
    jp DLOOP_TOP            ; start of loop
.endm

.macro _enddo
    jp DLOOP_TOP
    DLOOP_FWD
    DLOOP_POP
.endm

.macro _djnz                ; start of loop
    djnz DLOOP_TOP
    DLOOP_FWD
    DLOOP_POP
.endm
```

_Copyleft @ 2019 John Hardy, ALL WRONGS RESERVED_
This software is released under the GNU public license 3.0

_Contact me: [jh@lagado.com](mailto:jh@lagado.com) GitHub: [jhlagado](http://github.com/jhlagado) Twitter: [@jhlagado](https://twitter.com/jhlagado)_
