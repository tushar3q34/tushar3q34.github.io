+++
title = "Google Summer of Code 2025 Week 5-7"
date = 2025-08-29
description = "Virtual XREFs and updates"
aliases = ["/Google-Summer-of-Code-2025-Week-5-7/"]

[taxonomies]
tags = ["gsoc"]
+++

Continuing with GSoC, weeks 5-6 were the last two weeks before mid evaluation. I was not very active in week 7 due to internship 
tests & interviews in my college. And I am also writing this blog late, very late infact. TL;DR I passed mid evaluation and 
also got an internship of choice for Summer 2026 :D.

## Virtual XREFs

The weeks 1 to 4 revolved around devirtualization of C++ virtual calls. Now since the devirtualization was done and working for
most cases, I with my mentors decided to work on virtual XREFs.

For the readers who do not know what XREFs (called as cross references) are, they are basically a reference to another point
from the current offset.

![Meaning of XREFs](/images/xrefs_demo.png)

The task was simple, adding XREFs for virtual calls as well so that user can know wherever the virtual function has been called
strictly virtually. Implementation details can be saved for the interested readers, who can visit the PR and check the relevant commits.

The result was as such after I added the command `avx[t] <vfunc_name>` which returns the virtual xrefs to `<vfunc_name>` :

```c
[0x10000282c]> aaaa
[0x10000282c]> avD @ main
[0x10000282c]> pdf @ main
            ; UNKNOWN XREF from aav.0x100000028 @ +0xa8
            ;-- main:
            ;-- section.0.__TEXT.__text:
            ;-- _main:
            ;-- func.10000282c:
            ;-- pc:
┌ entry0(int64_t arg1);
│           ; arg int64_t arg1 @ x0
│           ; var int64_t var_58h @ stack - 0x58
│           ; var int64_t var_50h @ stack - 0x50
│           ; var int64_t var_48h @ stack - 0x48
│           ; var int64_t var_40h @ stack - 0x40
│           ; var int64_t var_24h @ stack - 0x24
│           ; var union RZ_var_20h_HYBRID var_20h @ stack - 0x20
│           ; var int64_t var_14h @ stack - 0x14
│           ; var int64_t var_10h @ stack - 0x10
│           ; var int64_t var_8h @ stack - 0x8
│           0x10000282c      sub   sp, sp, 0x60                        ; [00] -r-x section size 4980 named 0.__TEXT.__text
│           0x100002830      stp   fp, lr, [var_10h]
│           ......................
│           ......................
│       │   0x10000289c      blr   x8                                  ; Virtual Call : method.Dog.run / method.Cat.run / method.Human.run
│           ......................
│           ......................
│     │││   0x10000290c      blr   x8                                  ; Virtual Call : method.Dog.run / method.Cat.run / method.Human.run
│           ......................
│           ......................
│   ││││    0x100002958      blr   x8                                  ; Virtual Call : method.Dog.run / method.Cat.run / method.Human.run
│           ......................
│           ......................
└  ││ │     0x1000029c8      ret
[0x10000298c]> avx method.Dog.run
Virtual xrefs to method.Dog.run
C 0x100002958 blr x8
C 0x10000289c blr x8
C 0x10000290c blr x8
[0x10000290c]> avxt method.Dog.run
       from 
------------
0x100002958
0x10000289c
0x10000290c
```

## Resolving comments and life update

The most I did in week 7 was resolving comments because I was very packed with internship tests and interviews. I passed
my mid evaluation (I was very scared T_T) and I also got an internship at a quant company which was my target, and my 
interviewers were also very impressed by my contributions to open source and GSoC participation. The next targets
will be working on analysis of C++ without RTTI.