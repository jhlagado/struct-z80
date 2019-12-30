# struct-z80 - Structured Programming in Z80 Assembly Language

## Structure stack

Programming structures are recursive by nature and so it stands to reason that our first task should be to build a stack. Assemblers weren't really designed for meta-programming but you can implement a stack structure by using a bucket brigade of re-assignable assembler variables. I've made my stack twelve levels deep but you may decide to make this stack deeper which is an easy thing to do.

```
STRUC_COUNT .set 0

STRUC_12 .set 0
STRUC_11 .set 0
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

    STRUC_12 .set STRUC_11
    STRUC_11 .set STRUC_10
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
    STRUC_10 .set STRUC_11
    STRUC_11 .set STRUC_12
    STRUC_12 .set 0
.endm
```

## Loop stack

Loops have structure similar to branching structures but because they have different exiting requirements which may be nested within control-flow logic, I decided to implement a separate stack to manage them.

```
DLOOP_COUNT .set 0

DLOOP_12 .set 0
DLOOP_11 .set 0
DLOOP_10 .set 0
DLOOP_9 .set 0
DLOOP_8 .set 0
DLOOP_7 .set 0
DLOOP_6 .set 0
DLOOP_5 .set 0
DLOOP_4 .set 0
DLOOP_3 .set 0
DLOOP_2 .set 0
DLOOP_TOP .set 0

.macro DLOOP_PUSH, arg
    DLOOP_COUNT .set DLOOP_COUNT + 1

    DLOOP_12 .set DLOOP_11
    DLOOP_11 .set DLOOP_10
    DLOOP_10 .set DLOOP_9
    DLOOP_9 .set DLOOP_8
    DLOOP_8 .set DLOOP_7
    DLOOP_7 .set DLOOP_6
    DLOOP_6 .set DLOOP_5
    DLOOP_5 .set DLOOP_4
    DLOOP_4 .set DLOOP_3
    DLOOP_3 .set DLOOP_2
    DLOOP_2 .set DLOOP_TOP
    DLOOP_TOP .set arg
.endm

.macro DLOOP_POP
    DLOOP_COUNT .set DLOOP_COUNT - 1

    DLOOP_TOP .set DLOOP_2
    DLOOP_2 .set DLOOP_3
    DLOOP_3 .set DLOOP_4
    DLOOP_4 .set DLOOP_5
    DLOOP_5 .set DLOOP_6
    DLOOP_6 .set DLOOP_7
    DLOOP_7 .set DLOOP_8
    DLOOP_8 .set DLOOP_9
    DLOOP_9 .set DLOOP_10
    DLOOP_10 .set DLOOP_11
    DLOOP_11 .set DLOOP_12
    DLOOP_12 .set 0
.endm
```

Also we need a utility macro `STRUCT_FWD` with which we can use to go back and rewrite jump addresses that we can't resolve until later.

```
.macro STRUCT_FWD
    CUR_ADR .set $
    org STRUC_TOP - 2
    dw CUR_ADR
    org CUR_ADR
.endm

.macro DLOOP_FWD
    CUR_ADR .set $
    org DLOOP_TOP - 2
    dw CUR_ADR
    org CUR_ADR
.endm


```

_Copyleft @ 2019 John Hardy, ALL WRONGS RESERVED_
This software is released under the GNU public license 3.0

_Contact me: [jh@lagado.com](mailto:jh@lagado.com) GitHub: [jhlagado](http://github.com/jhlagado) Twitter: [@jhlagado](https://twitter.com/jhlagado)_
