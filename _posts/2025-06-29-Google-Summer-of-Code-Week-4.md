---
title: "Google Summer of Code 2025 Week 4"
date: 2025-06-29
author: "Tushar Jain"
words_per_minute : 90
read_time: true
tags:
    - GSoC
---

On time post for this, yeahhhhh! This week was mostly around covering the leftover cases from the last week and making sure
things work for both **x86/64** and **ARM** architecture.

## Fixing Endianness

For a lot of tests, the issue I faced was how to stpre value on stack and similarly, how to store value on the object.
The memory was allocated using the API of `aeim` which is standard, but writing to memory required using API of `wx`.
The issue was that we cannot write the hex value directly because of endianness so correcting it before storing is required.

```c
// If 0x401210 is the address of vtable
[0x00400a10]> wx 0x401210 @ 0x3000
[0x00400a10]> px 8 @ 0x3000
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x00003000  4012 1000 0000 0000                      @.......           // INCORRECT while accessing
[0x00400a10]> wx 0x101240 @ 0x3000
[0x00400a10]> px 8 @ 0x3000
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x00003000  1012 4000 0000 0000                      ..@.....           // CORRECT while accessing
```

## Multiple objects in same variable

Returning to the previous example from week 2 :
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

For my current implementation, there are two important things :
- It labels all possibilities for a virtual call by using the info of all classes present in a var
- It labels some extra virtual calls which might not be possible (explained later)

The result is as follows :
```c
; var union RZ_var_20h_HYBRID var_20h @ stack - 0x20
push  rbp
...........
...........
mov   rax, qword [var_20h.var_20h]
mov   rax, qword [rax]
add   rax, 0x18                            ; 24
mov   rdx, qword [rax]
mov   rax, qword [var_20h.var_20h]
mov   rdi, rax
call  rdx                                  ; Virtual Call : method.Dog.walk / method.Cat.walk / method.Human.walk
```

A potential improvement to the implementation can be to include only practically possible calls.
If we modify the example above to :
```c++
if (x % 3 == 2) {
  m = new Dog();
  m->run();                 // extra call
} else if (x % 3 == 1) {
  m = new Cat();
} else {
  m = new Human();
}

m->walk();
delete m;
```

Even for `m->run()` in `Dog`'s conditional, we get the `run()` method of all three as my implementation currently fails
to determine the sections of instructions which will get executed only for limited objects. (If there exists a method
to implement this, please let me know :D).

## Multiple vtables for an object

The case of multiple vtables is not very different. We just need to increase the scope of the simulated object.
Earlier, we were storing only a single vtable pointer for an object, now we will store multiple pointers adjacent to
each other.

If we look at the following code :
```c++
class A {
public:
  virtual void showA() { cout << "A::showA()" << endl; }
};

class B {
public:
  virtual void showB() { cout << "B::showB()" << endl; }
};

class C : public A, public B {
public:
  void showA() override { cout << "C::showA()" << endl; }

  void showB() override { cout << "C::showB()" << endl; }

  virtual void showC() { cout << "C::showC()" << endl; }
};

int main() {
  C *obj = new C();
  obj->showA();
  obj->showB();
  obj->showC();
  return 0;
}
```

We get the output as expected :
```c
mov   rax, qword [var_20h]
mov   rax, qword [rax]
mov   rdx, qword [rax]
mov   rax, qword [var_20h]
mov   rdi, rax
call  rdx                                  ; Virtual Call : sym.non_virtual_thunk_to_C::showB / method.C.showA
mov   rax, qword [var_20h]
mov   rax, qword [rax]
add   rax, 0x08
mov   rdx, qword [rax]
mov   rax, qword [var_20h]
mov   rdi, rax
call  rdx                                  ; Virtual Call : method.C.showB
mov   rax, qword [var_20h]
mov   rax, qword [rax]
add   rax, 0x10                            ; rflags
mov   rdx, qword [rax]
mov   rax, qword [var_20h]
mov   rdi, rax
call  rdx                                  ; Virtual Call : method.C.showC
```