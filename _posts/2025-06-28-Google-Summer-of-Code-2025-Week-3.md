---
title: "Google Summer of Code 2025 Week 3"
date: 2025-06-28
author: "Tushar Jain"
words_per_minute : 30
read_time: true
tags:
    - GSoC
---

This post will cover the progress made in Week 3. This week was the start of devirtualization using
the class marking implemented previously. The focus of this week was to handle at lest the basic cases.
This is another late post, sorry about that :D

## Virtual calls in C++

The pointer to virtual table(s) is stored at the start of an object. The following diagram :

![Virtual Tables](/assets/images/vtable_real.png){: .align-center}

The tricky part is that we do not have the physical object available in memory while emulating.
So we need to simulate the same to have the desired address while calling the virtual function.

## Simulating objects

For the current implementation :
- Stack is located from 0x1000 to 0x1FFF
- 0x3000 to 0x4000 store the pointers to vtables simulating objects

Note that the memory blocks here might have different address direction sense hence the addresses at both terminals
are given. The implementation is as follows :

![Virtual Tables Implementation](/assets/images/vtable_impl.png){: .align-center}

## Devirtualization

Now we just run the VM for extracting the necessary register values and marking the calls accordingly.

![Virtual Tables Instructions](/assets/images/vtable_insr.png){: .align-center}

## Result in disassembly

For the following C++ code :
```c++
class C{
private:
int x;
public :
  C(int x):x(x){}
  virtual void print(){std::cout << "C\n";}
};

int main()
{
  C * obj = new C(5);
  obj->print();
}
```

The result is as follows : 
```c
[0x00400720]> avD @ main
[0x00400720]> pdf @ main
            ; DATA XREF from sym._start_c @ 0x40074b
            ;-- rip:
┌ int main(int argc, char **argv, char **envp);
│           ; var struct C *var_20h @ stack - 0x20
│           0x0040088d      push  rbp
│           0x0040088e      ...........
│           ..........      ...........
│           ..........      ...........
│           0x004008b4      mov   rax, qword [var_20h]
│           0x004008b8      mov   rax, qword [rax]
│           0x004008bb      mov   rdx, qword [rax]
│           0x004008be      mov   rax, qword [var_20h]
│           0x004008c2      mov   rdi, rax
│           0x004008c5      call  rdx                                  ; Virtual Call : method.C.print
│           ..........      ...........
│           ..........      ...........
│           0x004008d1      pop   rbp
└           0x004008d2      ret
```