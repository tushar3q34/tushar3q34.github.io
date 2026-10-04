+++
title = "Google Summer of Code 2025 Week 8-10"
date = 2025-09-01
description = "RTTI and VTables"
aliases = ["/Google-Summer-of-Code-2025-Week-8-10/"]

[taxonomies]
tags = ["gsoc"]
+++

Since the analysis of C++ with RTTI available was almost done, it was time to move on to analysing binaries which do not have RTTI.
Rizin analyses the basic things of all classes, even if RTTI is absent but there were two key things that were absent :
- Correspondence between classes and their virtual tables
- Heirarchy of classes

The work I did in these three weeks dealt with resolving the first which is connecting virtual tables with classes.

## What is RTTI?

**RTTI** stands for ***Run Time Type Information***. This information is available in most binaries and is used for accessing type
information at runtime. A common usecase of RTTI is `std::dynamic_cast<T>` which is used to perform dynamic casting that requires
runtime type access of object pointers.

General structure of RTTI in a binary is as follows ([source](https://www.blackhat.com/presentations/bh-dc-07/Sabanal_Yason/Paper/bh-dc-07-Sabanal_Yason-WP.pdf)):

![RTTI in C++](/images/cxx_rtti.png)

Rizin uses RTTI information for a lot of analysis including building heirarchy and linking analysis classes with virtual tables.
Hence without RTTI, we had virtual table information and class information but we did not know which virtual table(s) corresponded with which class.

## Identification of virtual tables without RTTI

The method I followed for identification is described in detail in this amazing [blog](https://alschwalm.com/blog/static/2016/12/17/reversing-c-virtual-functions/) by Adam Schwalm.

So let's look at the constructor information of a class in binary without RTTI.

```c
[0x00400890]> pdf @ method.Dog.Dog
            ; CALL XREF from main @ 0x400a54
            ;-- Dog::Dog():
┌ method.Dog.Dog(int64_t arg1);
│           ; arg int64_t arg1 @ rdi
│           ; var int64_t var_20h @ stack - 0x20
│           0x00400d5e      push  rbp                                  ; Dog::Dog()
│           0x00400d5f      mov   rbp, rsp
│           0x00400d62      push  rbx
│           0x00400d63      sub   rsp, 0x18
│           0x00400d67      mov   qword [var_20h], rdi                 ; arg1
│           0x00400d6b      mov   rax, qword [var_20h]
│           0x00400d6f      mov   rdi, rax                             ; int64_t arg1
│           0x00400d72      call  method.Mammal.Mammal                 ; method.Mammal.Mammal ;  method.Mammal.Mammal(int64_t arg1)
│           0x00400d77      mov   edx, data.00401098                   ; 0x401098
│           0x00400d7c      mov   rax, qword [var_20h]
│           0x00400d80      mov   qword [rax], rdx
│           0x00400d83      mov   esi, str.Dog::Dog                    ; 0x400fec ; "Dog::Dog\n"
│           0x00400d88      mov   edi, obj.std::cout                   ; sym..bss
│                                                                      ; 0x6020a0
│           0x00400d8d      call  method.std::basic_ostream_char__std::cha.....
│       ┌─< 0x00400d92      jmp   0x400dae
..
│       │   ; CODE XREF from method.Dog.Dog @ 0x400d92
│       └─> 0x00400dae      add   rsp, 0x18
│           0x00400db2      pop   rbx
│           0x00400db3      pop   rbp
└           0x00400db4      ret
```

If we look closely at address `0x00400d77`, we can see a reference to a data object. This data object is none other than the virtual table
which we want to connect with our class. But this direct analysis only works for x86. For arm64, the constructor calls to subfunctions which
have references to virtual tables. An example is as follows :

```c
[0x100002c70]> pdf @ method.Dog.Dog
            ; CALL XREF from entry0 @ 0x100002c8c
            ;-- Dog::Dog():
            ;-- func.100002d1c:
┌ method.Dog.Dog(int64_t arg1);
│           ; arg int64_t arg1 @ x0
│           ; var int64_t var_20h @ stack - 0x20
│           ; var int64_t var_18h @ stack - 0x18
│           ; var int64_t var_10h @ stack - 0x10
│           0x100002d1c      sub   sp, sp, 0x20                        ; Dog::Dog()
│           0x100002d20      stp   fp, lr, [var_10h]
│           0x100002d24      add   fp, sp, 0x10
│           0x100002d28      str   x0, [var_18h]                       ; arg1
│           0x100002d2c      ldr   x0, [var_18h]                       ; [0x8:4]=-1 ; 8 ; int64_t arg1
│           0x100002d30      str   x0, [sp]
│           0x100002d34      bl    sym.Dog::Dog_sym.Dog::Dog_0x100002d48                                // Call to another function
│           0x100002d38      ldr   x0, [sp]
│           0x100002d3c      ldp   fp, lr, [var_10h]
│           0x100002d40      add   sp, sp, 0x20
└           0x100002d44      ret
[0x100002c70]> pdf @ sym.Dog::Dog_0x100002d48
            ; CALL XREF from method.Dog.Dog @ 0x100002d34
            ;-- func.100002d48:
┌ sym.Dog::Dog_0x100002d48(int64_t arg1);
│           ; arg int64_t arg1 @ x0
│           ; var int64_t var_40h @ stack - 0x40
│           ; var int64_t var_38h @ stack - 0x38
│           ; var int64_t var_30h @ stack - 0x30
│           ; var int64_t var_10h @ stack - 0x10
│           0x100002d48      sub   sp, sp, 0x40                        ; Dog::Dog()
│           0x100002d4c      stp   fp, lr, [sp, 0lr]
│           0x100002d50      add   fp, sp, 0x30
│           0x100002d54      adrp  x8, reloc._Unwind_Resume            ; 0x100004000
│           0x100002d58      add   x8, x8, 0xb0                        ; 0x1000040b0
│                                                                      ; sym.vtable_for_Dog
│           0x100002d5c      add   x9, x8, 0x10
│           0x100002d60      str   x9, [sp]
│           0x100002d64      add   x8, x8, 0x40
│           0x100002d68      str   x8, [var_38h]
│           0x100002d6c      stur  x0, [fp, -8]                        ; arg1
│           0x100002d70      ldur  x0, [fp, -8]                        ; int64_t arg1
│           0x100002d74      str   x0, [var_30h]
│           0x100002d78      bl    method.Animal.Animal                ; method.Animal.Animal ;  method.Animal.Animal(int64_t arg1)
│           0x100002d7c      ldr   x8, [var_30h]                       ; [0x10:4]=-1 ; 16
│           0x100002d80      add   x0, x8, 8                           ; int64_t arg1
│           0x100002d84      bl    method.Mammal.Mammal                ; method.Mammal.Mammal ;  method.Mammal.Mammal(int64_t arg1)
│       ┌─< 0x100002d88      b     0x100002d8c
│       │   ; CODE XREF from sym.Dog::Dog_0x100002d48 @ 0x100002d88
│       └─> 0x100002d8c      ldr   x9, [var_30h]                       ; [0x10:4]=-1 ; 16
│           0x100002d90      ldr   x8, [var_38h]                       ; [0x8:4]=-1 ; 8
│           0x100002d94      ldr   x10, [sp]
│           0x100002d98      str   x10, [x9]
│           0x100002d9c      str   x8, [x9, 8]
│           0x100002da0      adrp  x0, reloc._Unwind_Resume            ; 0x100004000
│           0x100002da4      ldr   x0, [x0, 0x50]                      ; [0x100004050:4]=0xc058 ; "X\xc0" ; int64_t arg1
│           0x100002da8      adrp  x1, data.100003000                  ; 0x100003000                    // Data xref to vtable
│           0x100002dac      add   x1, x1, 0xe44                       ; 0x100003e44 ; "Dog::Dog\n" ; int64_t arg2
│           0x100002db0      bl    method.std::__1::basic_ostream_char__std::__1::cha....
│       ┌─< 0x100002db4      b     0x100002db8
│       │   ; CODE XREF from sym.Dog::Dog_0x100002d48 @ 0x100002db4
│       └─> 0x100002db8      ldr   x0, [var_30h]                       ; [0x10:4]=-1 ; 16
│           0x100002dbc      ldp   fp, lr, [sp, 0lr]
│           0x100002dc0      add   sp, sp, 0x40
└           0x100002dc4      ret
```

Here, constructor `method.Dog.Dog` calls another function `sym.Dog::Dog_0x100002d48` which has the desired data reference.
I handled this using a recursive check of maximum depth `5` (which I chose arbitrarily because it is a large enough depth).

## Marking methods virtual and Obtained Result

After we know which virtual table(s) is(are) connected to which class, we can use this information to mark methods virtual.
Once these methods are marked virtual, the classic `avD/avx` analysis works on them.

The result is as follows :

```c
[0x00400890]> pdf @ method.Dog.Dog
            ; CALL XREF from main @ 0x400a54
            ;-- Dog::Dog():
┌ method.Dog.Dog(int64_t arg1);
│           ; arg int64_t arg1 @ rdi
│           ; var int64_t var_20h @ stack - 0x20
│           0x00400d5e      push  rbp                                  ; Dog::Dog()
│           0x00400d5f      mov   rbp, rsp
│           0x00400d62      push  rbx
│           0x00400d63      sub   rsp, 0x18
│           0x00400d67      mov   qword [var_20h], rdi                 ; arg1
│           0x00400d6b      mov   rax, qword [var_20h]
│           0x00400d6f      mov   rdi, rax                             ; int64_t arg1
│           0x00400d72      call  method.Mammal.Mammal                 ; method.Mammal.Mammal ;  method.Mammal.Mammal(int64_t arg1)
│           0x00400d77      mov   edx, vtable.Dog.0                    ; 0x401098                   // Virtual table is marked
│           0x00400d7c      mov   rax, qword [var_20h]
│           0x00400d80      mov   qword [rax], rdx
│           0x00400d83      mov   esi, str.Dog::Dog                    ; 0x400fec ; "Dog::Dog\n"
│           0x00400d88      mov   edi, obj.std::cout                   ; sym..bss
│                                                                      ; 0x6020a0
│           0x00400d8d      call  method.std::basic_ostream_char__std::cha....
│       ┌─< 0x00400d92      jmp   0x400dae
..
│       │   ; CODE XREF from method.Dog.Dog @ 0x400d92
│       └─> 0x00400dae      add   rsp, 0x18
│           0x00400db2      pop   rbx
│           0x00400db3      pop   rbp
└           0x00400db4      ret
[0x00400890]> acll
......
[Dog]
  (vtable at 0x401098)                                              // Virtual table connected and methods marked as VIRTUAL
nth       addr  vt_offset type    name                   
---------------------------------------------------------
  1 0x00400d5e ---------- DEFAULT Dog
  2 0x00400db6 0x00000000 VIRTUAL ~Dog
  3 0x00400e18 0x00000010 VIRTUAL run
  4 0x00400e36 0x00000018 VIRTUAL walk
  5 0x00400dec 0x00000008 VIRTUAL sym.Dog::_Dog_0x400dec
  6 0x00400c42 0x00000020 VIRTUAL method.Mammal.move
......
```

This works even for classes which have multiple virtual table.

## What's left and what's next?

Currently for C++ binaries, my project adds the following features :
- Marking stack variables with class name of object
- Devirtualizing register calls to virtual functions
- Support for XREFs of virtual functions
- Linking virtual tables with classes in non-RTTI binaries

What is left in the analysis of C++ binaries :
- Building class heirarchy for non-RTTI binaries
- Improving graph analysis (`acg` command)
- Improving class analysis in interactive mode

Most of the crucial tasks have been handled, which will heavily improve the analysis of C++ binaries involving classes
and virtual functions. The leftover tasks are mostly small and can be done later as well. Hence, to respect project's 
timeline, from the next week, I will start my work on Swift and ObjectiveC binaries. Hoping to return to the remaining
tasks soon!