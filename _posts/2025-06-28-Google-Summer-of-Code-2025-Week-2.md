---
title: "Google Summer of Code 2025 Week 2"
date: 2025-06-28
author: "Tushar Jain"
words_per_minute : 100
read_time: true
tags:
    - GSoC
---

Altough this is a late post, it will cover the changes and updates done in the second week. This week
focused on argument tracking as suggested by mentors and marking objects which can have multiple objects
in the same stack location / stack variable (more on this later in the post).

## Argument Tracking

In the first week, the implementation of marking stack variables with corresponding object type was majorly
dependent on variables discovered by **ESIL**, which is later to be replaced by **RzIL** and can require
a lot of cleanup and major changes.

The following is the implementation after discussion with mentors (xvilka and Rot127) :
- Variables are tracked using addresses at runtime **RzIL** emulation rather than their stack location
discovered by **ESIL**. 
- A list of variables is stored which using their accesses, which is shown by the current `RzVariable` API. Note : This
implementation still uses the **ESIL** API to get var reads/writes in a function but most probably, the API should remain largely same and get replaced by **RzIL** internally as per the ongoing Analysis migration.
- The implementation could have avoided **ESIL** entirely but then the visibility of variable marking would have been missing from the disassembly which is not reliable for the user.
- Each variable stores the runtime stack address during the tainting, and for devirtualization, only stack address is used making it unaffected by possible change in `RzVariable` API later.

```c
// Variable struct representing a single stack variable
typedef struct var_t {
	char *name; ///< for now, checks varibles discovered by ESIL
	ut64 stack_addr; ///< address of the stack where variable is stored according to RzIL VM
	ut64 instr_addr; ///< address of instruction which writes to variable
        char *class_name; ///< class which variable stores
} Variable;
```

## Handling multiple objects in a single stack variable

Let's look at the following C++ snippet:

```c++
// Class structure
class Mammal;
class Dog : public Mammal;
class Cat : public Mammal;
class Human : public Mammal;

// Instantiation
Mammal *m;
srand(time(NULL));
int x = rand();

if (x % 3 == 2) {
  m = new Dog();
} else if (x % 3 == 1) {
  m = new Cat();
} else {
  m = new Human();
}

m->walk();
delete m;
```

In this case the x86-64 assembly will look something like this :
```c
var int64_t var_20h @ stack - 0x20                          // Variable storing object
---------------------------------------------------------
// Same variable stores one of three instantiations
// decided by jump instructions on runtime

│   C    │-------------------------------------------------
│   O    │0x00400bd0      call  sym.imp.operator_new_unsigned_long
│   N    │....... Dog object .......
│   D    │0x00400be0      mov   qword [var_20h], rbx 
│   I    │-------------------------------------------------
│   T    │....... Cat object .......
│   I    │....... same variable .......
│   O    │-------------------------------------------------
│   N    │....... Human object .......
│   A    │....... same variable .......
│   L    │-------------------------------------------------

---------------------------------------------------------
// Virtual call to walk() method
0x00400c8c      mov   rax, qword [var_20h]
..........
0x00400ca1      call  rdx 
---------------------------------------------------------
```

Now the virtual call can call to any of the three `walk()` methods. For that, we need to mark
`var_20h` with all three variables.

Hence, the new struct has : 
```c
// Variable struct representing a single stack variable
typedef struct var_t {
	char *name; ///< for now, checks varibles discovered by ESIL
	ut64 stack_addr; ///< address of the stack where variable is stored according to RzIL VM
	ut64 instr_addr; ///< address of instruction which writes to variable
-       char *class_name; ///< class which variable stores
+       RzList /*<char*>*/ *class_names; ///< single variable might store multiple classes based on conditionals
} Variable;
```

And the visibility in the disassembly is handled by creating a union of structs
```c
var union RZ_var_20h_HYBRID var_20h @ stack - 0x20

// Composition of hybrid union
[0x00400a10]> t RZ_var_20h_HYBRID
pf "0***? (Human)var_20h (Cat)var_20h (Dog)var_20h"
```