---
title: "Google Summer of Code 2025 Week 1"
date: 2025-06-08
author: "Tushar Jain"
read_time: true
---

This was the first week of the project. As proposed, I started with analysis of C++ and the first step was
marking object pointers which are stored in variables.

## Marking variables

The general structure of an object allocation in C++ in **x86 assembly** is as follows : 

```c
0x00400896      mov   edi, 0x10                            // Argument Calculation
---------------------------------------------------------
0x0040089b      call  sym.imp.operator_new_unsigned_long   // Call to operator_new
---------------------------------------------------------
0x004008a0      mov   rbx, rax                             // .....
0x004008a3      mov   esi, 0x05                            // Argument manipulation                      
0x004008a8      mov   rdi, rbx                             // .....     
---------------------------------------------------------                 
0x004008ab      call  method.C.C_int                       // Call to constructor
---------------------------------------------------------
0x004008b0      mov   qword [var_20h], rbx                 // Store to variable
```

A similar structure is followed in **arm architecture** :

```c
0x1000030e8      mov   x0, 0x10                             // Argument Calculation
---------------------------------------------------------
0x1000030ec      bl    sym.imp.operator_new_unsigned_long   // Call to operator_new
---------------------------------------------------------
0x1000030f0      str   x0, [sp]                             // Argument manipulation
0x1000030f4      mov   w1, 5                                // .....
---------------------------------------------------------
0x1000030f8      bl    method.C.C_int                       // Call to constructor
---------------------------------------------------------
0x1000030fc      b     0x100003100
; CODE XREF from entry0 @ 0x1000030fc
0x100003100      ldr   x8, [sp]
0x100003104      stur  x8, [var_18h]                        // Store to variable
```

This structure comes with its own challenges especially in determining where
the pointer to allocated object is and how it is stored on stack - which is
used for virtual calls

The method agreed upon by mentors was **Tainting** using `RzEvent`s.

## Implementing basic Tainting

My implementation follows a three stage tainting : 
- `RZ_TAINT_MODE_OFF`   : The state till call to `operator_new`
- `RZ_TAINT_MODE_RED`   : The state after call to `operator_new` and till call to constructor
- `RZ_TAINT_MODE_BLUE`  : The state after call to constructor and till the write to variable

I first use xrefs to find all calls to `operator_new` in a given function.
In the next instruction, the **RzIL VM** is intitialized and 
`rax` is populated by `RZ_TAINT_VALUE`. There is also memory allocation and 
initialisation of `rsp` and `rbp`. The taint mode changes to `RZ_TAINT_MODE_RED`.

*Note : For now, the implementation is only for x86, but it can be easily extended using calling conventions*
*and is a TODO for the next week.*

The events are tracked. As soon as a function call is detected, which is most likely (always?)
the call to constructor, VM skips over it, the taint mode changes to `RZ_TAINT_MODE_BLUE` and VM proceeds to further instructions.

In the later instructions, if a write happens to the memory, the variable is noted using `afvW` and
is its type is changed to `struct obj *`. After this, the taint changes back to `RZ_TAINT_MODE_OFF`.

*Note : For now, the type is `struct obj *`. In the next week, the variables will be renamed properly.*

## Commands and Result

The implementation is a part of new command called `avD`. For now, it marks the objects of the function
at current offset.

```c
[0x00400720]> pdf @ main
            ; DATA XREF from sym._start_c @ 0x40074b
┌ int main(int argc, char **argv, char **envp);
│           ; var int64_t var_20h @ stack - 0x20
│           0x0040088d      push  rbp
│           0x0040088e      mov   rbp, rsp
│           0x00400891      push  rbx
│           0x00400892      sub   rsp, 0x18
│           0x00400896      mov   edi, 0x10                            ; 16
│           0x0040089b      call  sym.imp.operator_new_unsigned_long   ; sym.imp.operator_new_unsigned_long
│           0x004008a0      mov   rbx, rax
│           0x004008a3      mov   esi, 0x05                            ; int64_t arg2
│           0x004008a8      mov   rdi, rbx                             ; int64_t arg1
│           0x004008ab      call  method.C.C_int                       ; method.C.C_int ;  method.C.C_int(int64_t arg1, int64_t arg2)
│           0x004008b0      mov   qword [var_20h], rbx
│           0x004008b4      ...........
│           ...........     ...........
└           0x004008d2      ret
[0x00400720]> s main
[0x0040088d]> avD
[0x0040088d]> pdf @ main
            ; DATA XREF from sym._start_c @ 0x40074b
            ;-- rip:
┌ int main(int argc, char **argv, char **envp);
│           ; var struct obj *var_20h @ stack - 0x20
│           0x0040088d      push  rbp
│           0x0040088e      mov   rbp, rsp
│           0x00400891      push  rbx
│           0x00400892      sub   rsp, 0x18
│           0x00400896      mov   edi, 0x10                            ; 16
│           0x0040089b      call  sym.imp.operator_new_unsigned_long   ; sym.imp.operator_new_unsigned_long
│           0x004008a0      mov   rbx, rax
│           0x004008a3      mov   esi, 0x05                            ; int64_t arg2
│           0x004008a8      mov   rdi, rbx                             ; int64_t arg1
│           0x004008ab      call  method.C.C_int                       ; method.C.C_int ;  method.C.C_int(int64_t arg1, int64_t arg2)
│           0x004008b0      mov   qword [var_20h], rbx
│           0x004008b4      ...........
│           ...........     ...........
└           0x004008d2      ret

```

## TODO :

- Extend for arm and other architectures - calling conventions
- Mark variables with class names
- Check and resolve memory leaks (there are a few of them)


## Summary

This week completed the identification of an object pointer. Coming weeks and the later parts of the project
will use this marked identification for extracting virtual functions.