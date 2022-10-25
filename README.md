QBasic reversing notes
======================

This work-in-process documentation is based on reverse engineering and does
not come from any official source. You've been warned.

Except where noted, this work is licensed under a
[Creative Commons Attribution-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0/).

-- Mauro A. Meloni <maumeloni@gmail.com>


Microsoft QuickBasic Compiled EXE file format
---------------------------------------------

### Notes

QBasic source code lines can have up to 254 characters. If you need to write a
longer sencence, you can split lines using underscores as continuators `_`
(similar to `\` in Python).

There are also limits on line count and total size of `.bas` files.


### Compression

Many QBasic programs are linked with Microsoft Overlay Linker (LINK.EXE) [^masm],
and usually with the `/EXEPACK` argument, which applies a kind of RLE compression [^exepack]
on code and data of the executable. There are several unpackers available, both
in source code and in executable format [^lzexe][^unpack.py][^unpack.c][^unexepack.c].

[^masm]: https://github.com/wolfram77/dos-masm
[^exepack]: http://www.shikadi.net/moddingwiki/Microsoft_EXEPACK
[^lzexe]: http://www.bellard.org/lzexe.html
[^unpack.py]: https://github.com/scrippie/game-utils/blob/master/EXEPACK/unpack.py
[^unpack.c]: https://github.com/w4kfu/unEXEPACK/blob/master/unpack.c
[^unexepack.c]: https://sourceforge.net/p/openkb/code/ci/master/tree/src/tools/unexepack.c


### Constants

It seems that constants are not preceded by codes nor ended by terminators.

    DATATYPE    REPRESENTATION      EXAMPLE             VALUE
    integer     (immed operand)     mov ah, 0x10        16
    long        (immed operand)     mov ax, 0x78        120
    single      0x00112233          0x1f854541          12.345&
    double      0x0011223344556677  0x713d0ad7a3b02840  12.345#
    string      0x00 0x00 0x0000 0xaa 0xbb 0xcc
                |    |     |     +----------- start of string value
                |    |     +----------------- offset in memory
                |    +----------------------- unknown (seems to be 0x00)
                +---------------------------- string length


### Conditionals

Equality conditions are usually checked by `JNZ IFTRUE`. ??

Inequality conditions are usually checked by `JZ IFTRUE`. ??

Conditionals chained by `OR` or `AND` are expressed by `DEC`'ing cx time after time.

After the last condition in the expression there can be one of the following:

    AND CX, AX   ; chain of ANDs
    AND CX, CX
    JNZ IFTRUE

    OR CX, AX    ; chain of ORs
    AND CX, CX
    JNZ IFTRUE

Complex conditions (such as ones with parenthesis) require using temporary 
local variables.

Sometimes, if some variable is used in multiple expressions, it is possible 
that the compiler avoids memory access by refreshing the var with `OR AX,AX` and 
then testing again. The same occurs when comparing against `CONST 0`, but the 
comparison is omitted. In other words, conditionals that check for equality to 
0 (integer) may be expressed as `OR AX,AX`. REMEMBER THAT.

`SELECT CASE` conditionals are similar to `IF / ELSEIF` blocks, but are easily 
found because they usually test a local variable and end with a `B$SERR` call.


### Arithmetic operations

Aritmethic operations are usually performed with the usual operators `ADD, 
SUB, IMUL, IDIV`. An exception to this is multiplication by a power of two value 
such as 2, 16, 256, etc. where the operation is replaced by succesive `SAL/SHL` 
operations (log2(second_operand) times) over the original value.

Another exception is substraction of integers, as in some circumstances it is
done via adding the remainder from (0x100 - value). For example, to substract 
2 from a var, `ADD [var], 0x0fe` does the trick. The difference between that and 
adding 254 is indicated by the number of left zeroes present). See following:

    ADD [var], 0x0fe    ; is   var = var - 2
    ADD [var], 0x00fe   ; is   var = var + 254
    ADD [var], 0xfff7   ; is   var = var - 9


### Register usage

Sometimes the compiler tries to re-use currently assigned CPU registers (AX, BX, 
CX, etc) to avoid re-reading a variable from memory, or skips using a register 
due to the fact that it will be used in a following instruction. Usually this 
happens when a conditional follows an assigment that defines the tested variable.

To avoid this behavior (that means, to force reading a memory address with MOV), 
then it is useful to add a label before the instruction that used the register. 
In that way, the compiler cannot guarantee that the register is already set, and 
is forced to update it before the conditional or the relevant instruction.


### Floating point operations

Operations with SINGLE or DOUBLE variables are done via interrupts `0x35` to `0x3D`,
which map to corresponding FP opcodes according to the [wine source code][^fpu.c]. 
A complete list of FP calls can be found [online][^fpcalls].

So, all those floating point interruptions can be replaced by their equivalent
FPU x86 instructions as seen on [^fpu.c].

[^fpu.c]: https://github.com/alexhenrie/wine/blob/526b245237b9571beebac8a4653e7dd74c62c7de/dlls/krnl386.exe16/fpu.c
[^fpcalls]: http://www.csee.umbc.edu/courses/undergraduate/313/fall04/burt_katz/lectures/Lect12/floatingpoint.html


### FOR/NEXT loops

`FOR/NEXT` loops are implemented as the following sequence:

    MOV AX, <INITIALVALUE>
    JMP COMPARISON
    COMPARISON:
    MOV <VARIABLE>, AX
    CMP AX, <FINALVALUE>
    JLE INNERLOOP
    LOOPEND:
    <INSTRUCTIONS>
    INNERLOOP:
    <INSTRUCTIONS>
    MOV AX, <VARIABLE>
    INC AX

This accounts for STEP 1 loops. Other loops may be different.
If the `FOR/NEXT` does not have instructions inside (ex: a FOR ...: NEXT all on
one line), then the `MOV AX, <VARIABLE>` prepending the increment is ommited. 


### Local SUB/FUNCTION calls

`SUB/FUNCTION` procedures usually start with an `B$ENRA` or `B$ENSA` system call, 
preceded by an `mov cx, 0xXXXX` which is meant to indicate the reserved space 
for local variables.

Arguments are used via `BP` as base pointer plus an offset `[bp+0x0a]`. The
order of arguments is inverse to the offset, so for a binary function, the 
first argument would be `[bp+0x08]` and the second argument `[bp+0x06]`.

Local variables are used via `BP` as base pointer minus an offset `[bp-0x0a]` or 
can be specified as absolute addresses as in the main routine.

Almost all arguments for calls that are "constants in source" are previously 
stored into a variable (local or global) via `LEA`, pushed to stack, and then 
freed via `B$STDL`. Return values for functions which return strings (at least)
are passed with a call to `B$SCPF`.


### GOSUB calls

`GOSUB` calls are implemented as standard `CALL`s to line-number or label, with 
a possible/corresponding `RETURN` as `ret`.


### System Call signatures

A call signature is a local reference that jumps to a far location of a
library function. Call signatures start with `0xCD` or `0xC7` and are 3 to 5 bytes 
in length. Function prototypes are detailed below.


### Function prototypes

Each row lists the signature prototype, the function mnemonic, the datatypes of 
its arguments (`PUSH`ed), the datatype of the result value (in `REG_AX`) and the 
QBasic function that probably represents. Datatypes can be STRing, VARiant, 
INTeger, LONG, SNG single, DBL double, and ARGN which indicates the number 
of arguments pushed for functions that accept a variable number of arguments.


    PROTOTYPE         OFFSET    MNEMONIC    ARGUMENTS       RETURN  FUNCTION

    C7 06 "e0X" 04              B$RUN       str             -       RUN s$
    CD 3F 00                    B$ASSN  ign str int ign str int -    (DIM v AS STRING * 1) = "c"
    CD 3F 01          0x320     B$BEEP      -               -       BEEP
    CD 3F 02          0x323     B$BLOD      str var var     -       BLOAD
    CD 3F 03                    B$BSAV      str var var     -       BSAVE
    CD 3F 04                    (unknown)
    CD 3F 05                    B$CDIR      str             -       CHDIR s$
    CD 3F 06                    (unknown)
    CD 3F 07          0x86      B$CHOU      int             -       PRINT #i, s$ (precedes B$PESD)
    CD 3F 08                    B$CIRC      var var var     -       CIRCLE
    CD 3F 09          0x89      B$CLOS      argn ...        -       CLOSE [i%], ...
    CD 3F 0A          0x2b6     B$COLR      argn ...        -       COLOR
    CD 3F 0B          0x2b9     B$CSCN      argn ...        -       SCREEN i%, ...
    CD 3F 0C                    B$CSRL      -               -       CSRLIN
    CD 3F 0D                    (unknown)
    CD 3F 0E                    B$CSTT      var var         -
    CD 3F 0F          0x8c      B$DDIM      int int int var var -   DIM array(min, max, elemsize, 0x101, destvar)
    CD 3F 10                    B$DRAW      str             -       DRAW s$
    CD 3F 11                    B$DSEG      int             -       DEF SEG i%
    CD 3F 12                    B$DSG0      -               -       DEF SEG
    CD 3F 13          0x8f      B$DSKI      int             -       LINE INPUT #i (precedes B$LNIN)
    CD 3F 14                    (unknown)
    CD 3F 15                    (unknown)
    CD 3F 16                    (unknown)
    CD 3F 17                    (unknown)
    CD 3F 18                    B$EPE0      -               -       PEN ON
    CD 3F 19                    B$EPE1      -               -       PEN OFF
    CD 3F 1A                    B$EPE2      -               -       PEN STOP
    CD 3F 1B                    B$ERAS      var             -       ERASE
    CD 3F 1C                    B$ERDS      -               str     ERDEV$
    CD 3F 1D                    B$ERDV      -               int     ERDEV
    CD 3F 1E                    (unknown)
    CD 3F 1F                    (unknown)
    CD 3F 20                    (unknown)
    CD 3F 21                    B$ETC0      int             -       COM(i%) ON
    CD 3F 22                    B$ETC1      int             -       COM(i%) OFF
    CD 3F 23                    B$ETC2      int             -       COM(i%) STOP
    CD 3F 24                    B$ETK0      int             -       KEY(i%) ON
    CD 3F 25                    B$ETK1      int             -       KEY(i%) OFF
    CD 3F 26                    B$ETK2      int             -       KEY(i%) STOP
    CD 3F 27                    B$ETL0      -               -       PLAY ON
    CD 3F 28                    B$ETL1      -               -       PLAY OFF
    CD 3F 29                    B$ETL2      -               -       PLAY STOP
    CD 3F 2A                    B$ETS0      int             -       STRIG(i%) ON
    CD 3F 2B                    B$ETS1      int             -       STRIG(i%) OFF
    CD 3F 2C                    B$ETS2      int             -       STRIG(i%) STOP
    CD 3F 2D                    B$ETT0      -               -       TIMER ON
    CD 3F 2E                    B$ETT1      -               -       TIMER OFF
    CD 3F 2F                    B$ETT2      -               -       TIMER STOP
    CD 3F 30          0x380     B$FASC      str             int     ASC(s$)
    CD 3F 31                    B$FATR      int int         int     FILEATTR
    CD 3F 32          0x9b      B$FCHR      int             str     CHR$(i%)
    CD 3F 33                    B$FCMD      -               str     COMMAND$
    CD 3F 34                    B$FCVD      str             dbl
    CD 3F 35                    B$FCVI      str             int
    CD 3F 36                    B$FCVL      str             long
    CD 3F 37                    B$FCVS      str             sng
    CD 3F 38                    B$FDAT      -               str     DATE$
    CD 3F 39                    B$FEOF      int             int     EOF(i%)
    CD 3F 3A                    (unknown)
    CD 3F 3B                    B$FEVS      str             str     ENVIRON$()
    CD 3F 3C                    B$FHEX      long            str     HEX$(i%)
    CD 3F 3D                    B$FICT      int             str
    CD 3F 3E          0x392     B$FIEL      int str         -       FIELD #i, len AS var
    CD 3F 3F                    B$FILS      str             -       FILES
    CD 3F 40          0xb3      B$FINP      int var         str     INPUT$()
    CD 3F 41          0xb6      B$FLDP      int             -       FIELD #i, len AS var (precedes B$FIEL)
    CD 3F 42          0xb9      B$FLEN      str             int     LEN(s$)
    CD 3F 43                    B$FLOC      int             int
    CD 3F 44                    B$FLOF      int             long
    CD 3F 45                    B$FMDF      dbl             str     MKDMBF$()
    CD 3F 46          0xbf      B$FMID      str int int     str     MID$($s, i%, j%)
    CD 3F 47                    B$FMKD      dbl             str     MKD$()
    CD 3F 48                    B$FMKI      int             str     MKI$()
    CD 3F 49                    B$FMKL      long            str     MKL$()
    CD 3F 4A                    B$FMKS      sng             str     MKS$()
    CD 3F 4B                    B$FMSF      sng             str     MKSMBF$()
    CD 3F 4C                    B$FOCT      long            str     OCT$(i%)
    CD 3F 4D                    B$FPEN      int             int     PEN()
    CD 3F 4E                    (unknown)
    CD 3F 4F                    B$FPOS      int             int     POS()
    CD 3F 50                    B$FREF      -               int     FREEFILE
    CD 3F 51                    B$FRI2      int             int     FRE()
    CD 3F 52                    (unknown)
    CD 3F 53                    B$FSCN      int int int     int     SCREEN()
    CD 3F 54                    B$FSEK      int             long
    CD 3F 55                    (unknown)
    CD 3F 56                    B$FSPC      int             -       SPC(i%)
    CD 3F 57                    B$FSTG      int             int     STRIG(i%)
    CD 3F 58          0xe6      B$FTAB      int                     TAB()
    CD 3F 59                    B$FTIM      -               str     TIME$
    CD 3F 5A          0xec      B$FVAL      str             -       VAL(s$)
    CD 3F 5B                    B$FWID      int int         -       WIDTH #i, i%
    CD 3F 5C                    B$GET1      int             -       GET #i
    CD 3F 5D          0x3b0     B$GET2      int long        -       GET #i, l&
    CD 3F 5E                    B$GET3      int ign var int -       GET #i, , var
    CD 3F 5F                    B$GET4      int long ign var int -  GET #i, l&, var
    CD 3F 60          0x2c5     B$GGET      addr32 addr16   -       GET (graphical)
    CD 3F 61          0x2c8     B$GPUT      addr32 addr16 mode -    PUT (graphical)
    CD 3F 62          0xf2      B$INKY      -               str     INKEY$
    CD 3F 63                    B$INPP      ign int         -       INPUT # TODO
    CD 3F 64          0xf8      B$INS2      str str         int     INSTR(s$, s$)
    CD 3F 65                    B$INS3      int str str     int     INSTR(i%, $s, $s)
    CD 3F 66                    B$KFUN      int             -       KEY ON|OFF|LIST
    CD 3F 67                    B$KILL      str             -       KILL s$
    CD 3F 68          0x101     B$KMAP      int str         -       KEY i%, s$
    CD 3F 69                    B$LBND      var int         int     LBOUND
    CD 3F 6A          0x107     B$LCAS      str             str     LCASE$()
    CD 3F 6B                    B$LDFS      addr str int    str     PRINT (DIM var$ * len)
    CD 3F 6C          0x10d     B$LEFT      str int         str     LEFT$()
    CD 3F 6D          0x2cb     B$LINE      var var var     -       LINE
        # parameter 1 defines the color
        # parameter 2 is unknown
        # parameter 3 defines the shape: 0x0=LINE/, 0x1=RECT/B, 0x2=FILLED/BF
    CD 3F 6E                    B$LNIN  str var str var int -       LINE INPUT
    CD 3F 6F                    B$LOCK  int long long int   -       LOCK/UNLOCK
    CD 3F 70          0x113     B$LOCT      argn            -       LOCATE
    CD 3F 71                    B$LPOS      int             int
    CD 3F 72                    B$LPRT      -               -
    CD 3F 73                    B$LSET      str ign str var -       LSET
    CD 3F 74                    B$LTRM      str             str     LTRIM$(s$)
    CD 3F 75                    B$LWID      int             -       WIDTH LPRINT
    CD 3F 76                    B$MCVD      str             dbl
    CD 3F 77                    B$MCVS      str             sng
    CD 3F 78                    B$MDIR      str             -       MKDIR s$
    CD 3F 79                    (unknown)
    CD 3F 7A          0x2ce     B$N1I2      int int         -       set point 1 (i%, i%)
    CD 3F 7B          0x2d1     B$N1R4      sng sng         -       set point 1 (f!, f!)
    CD 3F 7C          0x2d4     B$N2I2      int int         -       set point 2 (i%, i%)
    CD 3F 7D          0x2d7     B$N2R4      sng sng         -       set point 2 (f!, f!)
    CD 3F 7E                    B$NAME      str str         -       NAME s$ AS t$
    CD 3F 7F                    B$ONCA      int ign addr    -       ON COM(i%) GOSUB Offset
    CD 3F 80                    B$ONKA      int ign addr    -       ON KEY(i%) GOSUB Offset
    CD 3F 81                    B$ONLA      ign addr        -       ON PLAY(i%) GOSUB Offset
    CD 3F 82                    B$ONPA      ign addr        -       ON PEN GOSUB Offset
    CD 3F 83                    B$ONSA      int ign addr    -       ON STRIG(i%) GOSUB Offset
    CD 3F 84                    (unknown)
    CD 3F 85                    B$ONTA      ign int ign addr -      ON TIMER(i%) GOSUB Offset
    CD 3F 86          0x12e     B$OOPN      str int str int -       OPEN (2)
        # parameter 1 is a one character str with the mode ("I", "O", "R", "A", "B")
        # parameter 2 indicates the file number
        # parameter 3 indicates the file name
        # parameter 4 indicates the record length or 0xFFFF for unknown
    CD 3F 87                    B$OPEN      str int int int -       OPEN
        # parameter 1 indicates the file name
        # parameter 2 indicates the file number
        # parameter 3 indicates the record length or 0xFFFF for unknown
        # parameter 4 are flags:
        #    0x0001=INPUT, 0x0002=OUTPUT, 0x0004=RANDOM, 0x0008=APPEND, 0x0020=BINARY, 
        #    0x0100=ACCESS_READ, 0x0200=ACCESS_WRITE, 0X0300=ACCESS_READ_WRITE,
        #    0X4000=LOCK_SHARED, 0x3000=LOCK_READ, 0x2000=LOCK_WRITE, 0x1000=LOCK_READ_WRITE,
    CD 3F 88                    B$PAIN      var var         -       PAINT
    CD 3F 89                    (unknown)
    CD 3F 8A                    B$PAL2      int long        -       PALETTE
    CD 3F 8B                    B$PALU      var var var     -
    CD 3F 8C          0x134     B$PCI2      int             -       PRINT #i, BYTE/INTEGER, LIST ?
    CD 3F 8D                    (unknown)
    CD 3F 8E                    B$PCPY      int int         -       PCOPY
    CD 3F 8F                    (unknown)
    CD 3F 90                    (unknown)
    CD 3F 91          0x140     B$PCSD      str             -       PRINT s$,
    CD 3F 92                    (unknown)
    CD 3F 93          0x143     B$PEI2      int             -       PRINT i%
    CD 3F 94                    B$PEI4      long            -       PRINT i&
    CD 3F 95          0x149     B$PEOS      -               -
    CD 3F 96                    B$PER4      sng             -       PRINT f!
    CD 3F 97                    B$PER8      dbl             -       PRINT d#
    CD 3F 98          0x152     B$PESD      str             -       PRINT s$
    CD 3F 99                    (unknown)
    CD 3F 9A                    (unknown)
    CD 3F 9B                    B$PMAP      var var var     int     PMAP()
    CD 3F 9C                    B$PNI2      int int         int     POINT()
    CD 3F 9D                    (unknown)
    CD 3F 9E                    B$PNT1      int             int     POINT()
    CD 3F 9F                    (unknown)
    CD 3F A0                    (unknown)
    CD 3F A1                    (unknown)
    CD 3F A2                    (unknown)
    CD 3F A3          0x155     B$PSI2      int             -       PRINT i%;
    CD 3F A4                    B$PSI4      long            -       PRINT i&;
    CD 3F A5                    B$PSR4      sng             -       PRINT f!;
    CD 3F A6                    B$PSR8      dbl             -       PRINT d#;
    CD 3F A7          0x161     B$PSSD      str             -       PRINT s$;
    CD 3F A8                    B$PSTC      int             -       PRESET()
    CD 3F A9                    B$PUT1      int             -       PUT #i
    CD 3F AA                    B$PUT2      int long        -       PUT #i, l&
    CD 3F AB                    B$PUT3      int ign var int -       PUT #i, , var
    CD 3F AC                    B$PUT4  int long ign var int -      PUT #i, l&, var
    CD 3F AD          0x164     B$RDI2      ign int         -       READ i%
    CD 3F AE                    B$RDI4      lng             -       READ l&
    CD 3F AF                    B$RDIM      int int int var var -   REDIM
    CD 3F B0                    B$RDIR      str             -       RMDIR s$
    CD 3F B1                    B$RDR4      sng             -       READ f!
    CD 3F B2                    B$RDR8      dbl             -       READ d#
    CD 3F B3                    B$RSDS      ign str var     -       READ s$
    CD 3F B4                    B$REST      -               -       RESET
    CD 3F B5          0x176     B$RGHT      str int         str     RIGHT$()
    CD 3F B6                    (unknown)
    CD 3F B7          0x3f8     B$RND0      -               -       RND
    CD 3F B8                    B$RND1      long            -       RND(l&)
    CD 3F B9                    (unknown)
    CD 3F BA          0x401     B$RNZP      dbl             -       RANDOMIZE
    CD 3F BB                    B$RSET      str ign str var -       RSET
    CD 3F BC                    B$RTRM      str             str     RTRIM$(s$)
    CD 3F BD                    B$S1I2      int int         -       set point 1 STEP(i%, i%)
    CD 3F BE                    (unknown)
    CD 3F BF                    B$S2I2      int int         -       set point 2 STEP(i%, i%)
    CD 3F C0                    (unknown)
    CD 3F C1                    B$SADD      str             int     SADD()
    CD 3F C2          0x185     B$SASS      str var         -       s$ = (LET)
    CD 3F C3          0x188     B$SCAT      str str         str     s$ + s$ (CONCAT)
    CD 3F C4          0x18b     B$SCLS      int             -       CLS # TODO
    CD 3F C5                    (unknown)
    CD 3F C6                    B$SDAT      str             -       DATE$ = s$
    CD 3F C7                    B$SENV      str             -       ENVIRON
    CD 3F C8          0x407     B$SERR      int             -       ERROR / END SELECT
    CD 3F C9                    (unknown)
    CD 3F CA                    B$SICT      int str         -
    CD 3F CB                    B$SMID      str int int     -       MID$(s$, i%, j%) = s$
    CD 3F CC          0x197     B$SOND      int sng         -       SOUND i%, f!
    CD 3F CD          0x19a     B$SPAC      int             str     SPACE$(i%)
    CD 3F CE          0x19d     B$SPLY      str             -       PLAY s$
    CD 3F CF                    B$SSEK      int long        -
    CD 3F D0          0x413     B$SSHL      str             -       SHELL s$
    CD 3F D1          0x1a0     B$STDL      str             -       free local/temporal string var memory
    CD 3F D2          0x1a3     B$STI2      int             str     STR$(i%)
    CD 3F D3                    B$STI4      long            str     STR$(l&)
    CD 3F D4                    B$STIK      int             int     STIK()
    CD 3F D5                    (unknown)
    CD 3F D6          0x1a9     B$STR4      sng             str     STR$(f!)
    CD 3F D7                    B$STR8      dbl             str     STR$(d#)
    CD 3F D8                    B$STRI      int int         str     STRING$(i%, i%)
    CD 3F D9                    B$STRS      int str         str     STRING$(i%, s$)
    CD 3F DA                    (unknown)
    CD 3F DB                    (unknown)
    CD 3F DC                    (unknown)
    CD 3F DD                    (unknown)
    CD 3F DE          0x1c1     B$TIMR      -               -       TIMER
    CD 3F DF                    B$UBND      var int         int     UBOUND()
    CD 3F E0                    B$UCAS      str             str     UCASE$()
    CD 3F E1                    B$USNG      str             -       PRINT USING
    CD 3F E2                    B$VARP      var var         int
    CD 3F E3                    (unknown)
    CD 3F E4                    B$VIEW      int * 7         -       VIEW # TODO
    CD 3F E5                    B$VWPT      int int         -       VIEW PRINT
    CD 3F E6                    (unknown)
    CD 3F E7          0x1d3     B$WIDT      int int         -       WIDTH
    CD 3F E8                    (unknown)
    CD 3F E9                    B$WIND      long * 4 var    -       WINDOW # TODO
    CD 3F EA          0x1d6?    B$WRIT      -               -       WRITE
    CD 3F EB                    B$?EVT      -               -       start of event trapping
    CD 3F EC          0x10      B$CEND      -               -       SYSTEM
    CD 3F ED          0x13      B$CENP      -               -       END (MAIN PROGRAM)
    CD 3F EE                    (unknown)
    CD 3F EF          0x1df     B$ENFA      -               -       DEF FNname
    CD 3F F0          0x1e2     B$ENRA      -               -       ENTER DYNAMIC SUB/FUNC
    CD 3F F1                    B$ENSA      -               -       ENTER STATIC SUB/FUNC
    CD 3F F2                    B$EVCK      -               -       end of event trapping
    CD 3F F3          0x1eb     B$EXFA      -               -       END DEF
    CD 3F F4          0x1ee     B$EXSA      -               -       EXIT SUB/FUNCTION
    CD 3F F5                    (unknown)
    CD 3F F6                    B$FERL      -               int     ERL
    CD 3F F7                    B$FERR      -               int     ERR
    CD 3F F8                    B$GOSA      -               -       GOSUB [ax]
    CD 3F F9                    (unknown)
    CD 3F FA                    (unknown)
    CD 3F FB                    B$OEGA      ign addr        -       ON ERROR GOTO [Offset]
    CD 3F FC                    B$OGSA      -               -       ON i% GOSUB offset1, offset2... (db)
    CD 3F FD          0x209     B$OGTA      -               -       ON i% GOTO offset1, offset2... (db)
    CD 3F FE                    (unknown)
    CD 3F FF                    (unknown)
    CD 3F FF 01                 (unknown)
    CD 3F FF 02                 (unknown)
    CD 3F FF 03                 B$RETA      -               -       RETURN
    CD 3F FF 04                 B$RSTA      -               -       RESTORE
    CD 3F FF 05                 B$RSTB      addr            -       RESTORE [DataOffset]
    CD 3F FF 06                 (unknown)
    CD 3F FF 07                 B$SCHN      str             -       CHAIN s$
    CD 3F FF 08                 B$SCLR      var var         -       CLEAR
    CD 3F FF 09       0x220     B$SCMP      str str         -       IF s$ = s$ (COMPARE)
    CD 3F FF 0A       0x224     B$SCPF      var             -       RETURN STRING FROM FUNCTION
    CD 3F FF 0B                 (unknown)
    CD 3F FF 0C                 B$STOP      -               -       STOP
    CD 3F FF 0D                 B$SWSD      -               -       SWAP s$, s$
    CD 3F FF 0E                 (unknown)
    CD 3F FF 0F                 (unknown)
    CD 3F FF 10                 (unknown)
    CD 3F FF 11                 (unknown)
    CD 3F FF 12                 (unknown)
    CD 3F FF 13                 (unknown)
    CD 3F FF 14                 (unknown)
    CD 3F FF 15                 B$ATN4      -               -       ATN(f!)
    CD 3F FF 16                 B$ATN8      -               -       ATN(d#)
    CD 3F FF 17                 B$COS4      -               -       COS(f!)
    CD 3F FF 18                 B$COS8      -               -       COS(d#)
    CD 3F FF 19                 B$EXP4      -               -       EXP(f!)
    CD 3F FF 1A                 B$EXP8      -               -       EXP(d#)
    CD 3F FF 1B       0x234     B$FCMP      -               -       FLOATING POINT COMPARISON
    CD 3F FF 1C       0x258     B$FIL2      -               -       convert int to sng and stack in FP
    CD 3F FF 1D                 B$FILD      -               -       convert long to sng and stack in FP
    CD 3F FF 1E       0x260     B$FIS2      -               int     unstack FP and convert to int
    CD 3F FF 1F                 B$FIST      -               long    unstack FP and convert to long
    CD 3F FF 20                 B$FIX4      -               -       FIX(f!)
    CD 3F FF 21                 B$FIX8      -               -       FIX(d#)
    CD 3F FF 22                 (unknown)
    CD 3F FF 23       0x274     B$INT4      -               -       INT(f!)
    CD 3F FF 24                 B$INT8      -               -       INT(d#)
    CD 3F FF 25                 B$LOG4      -               -       LOG(f!)
    CD 3F FF 26                 B$LOG8      -               -       LOG(d#)
    CD 3F FF 27                 B$POW4      -               -       POW(f!, f!)
    CD 3F FF 28                 B$POW8      -               -       POW(d#, d#)
    CD 3F FF 29                 B$SGN4      -               -       SGN(f!)
    CD 3F FF 2A                 B$SGN8      -               -       SGN(d#)
    CD 3F FF 2B                 B$SIN4      -               -       SIN(f!)
    CD 3F FF 2C                 B$SIN8      -               -       SIN(d#)
    CD 3F FF 2D                 B$TAN4      -               -       TAN(f!)
    CD 3F FF 2E                 B$TAN8      -               -       TAN(d#)
    CD 3F FF 2F                 (unknown)


### How to find signatures/stubs

    OFFSET          SIZE
    ...
    0x????............6    CD 3F EC CD 3F ED   base of call signatures
    [call_signatures]
    ...
    0x0220            2    base of constants in memory
                           (address where the first const will be stored)
    0x0230                 start of TEXT section (assembly)
    [assembly_code]        (C7 06 BC 18 in THIS case, but this is no constant)
    ...
    0x????           11    "C_FILE_INFO"
    ...
    0x???? + 95            base of constants in image
                           (offset of file where the first const is stored)
    [constants]
    0x????            7    52 42 8C C0 05 10 00


### Disassembling with Radare2

[Radare2][^radare] is a useful disassembler and general reverse engineering tool.

Some basic configuration directives are:

    e asm.cmtright = true
    e asm.pseudo = true
    # e cmd.stack = true
    # eco solarized
    e scr.utf8 = true

    # e anal.arch=x86.udis
    # e anal.arch=x86

    # define start_of_text_section
    f start_of_text_section @ 0000:152b   # it should be via search
                                          # B801005050 B8020050
    f end_of_text_section @ 0000:deaf

    # define search space and options
    e search.from = start_of_text_section
    e search.to = end_of_text_section
    e search.align = true
    # e search.flags = 0

    # go to start_of_text_section
    s start_of_text_section
    af
    pd
    VV

    # function definition
    af+ 0xd41:0x2b9 1 test_func

    # live patching of a binary
    e io.cache=true
    # f--

    # replace int fpu emulation instructions
    # with real fpu instructions
    # /a int 0x35
    # /A int
    e cmd.hit = wx 9bd8
    # /c int 0x34
    /x cd34

    e cmd.hit = wx 9bd9
    # /c int 0x35
    /x cd35

    e cmd.hit = wx 9bda
    # /c int 0x36
    /x cd36

    e cmd.hit = wx 9bdb
    # /c int 0x37
    /x cd37

    e cmd.hit = wx 9bdc
    # /c int 0x38
    /x cd38

    e cmd.hit = wx 9bdd
    # /c int 0x39
    /x cd39

    e cmd.hit = wx 9bde
    # /c int 0x3a
    /x cd39

    e cmd.hit = wx 9bdf
    # /c int 0x3b
    /x cd3b

    e cmd.hit = wx 9b90
    # /c int 0x3d
    /x cd3d

There is an [official cheatsheet][^radintro] and other [tutorials][^radtutor].


[^radare]: https://github.com/radare/radare2
[^radintro]: https://github.com/radare/radare2/blob/master/doc/intro.md
[^radtutor]: https://blog.techorganic.com/2016/03/08/radare-2-in-0x1e-minutes/
[^getput]: http://www.angelfire.com/scifi/nightcode/prglang/qbasic/function/graphics/get_put_graphics.html
