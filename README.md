<!---
{
  "id": "f51efa1c-ef4b-41e0-a6c9-3fcd0b95a155",
  "teaches": "Cpp: Inheritance and Virtual Functions",
  "depends_on": [
    "dd36c1f7-e644-45a8-b11b-cf150a18d38c",
    "c32afdd3-48c3-4ff2-b5f7-a2179a2f8093"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-08",
  "keywords": [
    "C++",
    "inheritance",
    "base class",
    "derived class",
    "virtual",
    "override",
    "polymorphism",
    "vtable",
    "dynamic dispatch"
  ]
}
--->

# Cpp: Inheritance and Virtual Functions

> In this exercise you build the lecture's vehicle hierarchy: a base class `Vehicle` and derived classes that extend it, then make a base-class pointer call the right method of the actual object with `virtual`. You will look at the object layout to see the vtable pointer appear, and at the generated code to see the indirect call. No `sudo` is required.

## Introduction

**Inheritance** lets a new class take over all attributes and methods of an existing one and add its own: `Auto` inherits from `Fahrzeug`, gets `fahre()` for free and adds `hupe()`. Memory-wise, a derived object *starts with* a complete base object, which is why a pointer to the base can point at a derived object.

That is where **polymorphism** begins. If the base pointer calls a method, which version runs: the base one or the derived one? Without `virtual`, the compiler decides at compile time from the pointer's type (static dispatch). With `virtual`, the decision is made at run time from the object's type (dynamic dispatch), using a hidden table of function pointers, the **vtable**. You have built such a table by hand in C in the exercise on function pointers; here the compiler does it.

### 1.1) Further Readings and Other Sources

* [cppreference: Derived classes](https://en.cppreference.com/w/cpp/language/derived_class)
* [cppreference: virtual function specifier](https://en.cppreference.com/w/cpp/language/virtual)
* [C++ Core Guidelines: C.120 to C.138 (class hierarchies)](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#SS-hier)
* [Itanium C++ ABI: vtable layout](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable)

## Tasks

### Task 1: A base and a derived class

```bash
mkdir -p ~/oop-7 && cd ~/oop-7
```

```c++
// file: vehicle.cpp
#include <cstdio>

class Vehicle {
public:
    void fahre(int m) { position += m; }
    void beschreibe() const { std::printf("Fahrzeug bei %d m\n", position); }
    int position = 0;
};

class Auto : public Vehicle {
public:
    void hupe() const { std::printf("Hup!\n"); }
    void beschreibe() const { std::printf("Auto bei %d m\n", position); }
};

int main() {
    Auto a;
    a.fahre(10);         // inherited
    a.hupe();            // own
    a.beschreibe();      // Auto's version
    std::printf("sizeof Vehicle=%zu Auto=%zu\n", sizeof(Vehicle), sizeof(Auto));
}
```

```bash
g++ -Wall -Wextra -g vehicle.cpp -o vehicle && ./vehicle
```

### Task 2: Through a base pointer

```c++
    Vehicle *p = &a;
    p->beschreibe();
```

It prints `Fahrzeug bei 10 m`, although `p` points at an `Auto`. The compiler chose the method from the *pointer's* type. This is static dispatch.

### Task 3: `virtual`

Add `virtual` in the base class and `override` in the derived one:

```c++
    virtual void beschreibe() const { ... }      // Vehicle
    void beschreibe() const override { ... }     // Auto
```

Run again: `Auto bei 10 m`. Then print the sizes again. `Vehicle` grew by one pointer: the vtable pointer. Misspell the method name in `Auto` (`beschreibe2`) and see that `override` turns a silent non-override into a compile error.

### Task 4: Look at the dispatch

```bash
g++ -O0 -g -c vehicle.cpp && objdump -d -C vehicle.o | grep -B4 -A4 'call.*\*'
```

Find the indirect call `call *%rax` (or similar) in `main` where `p->beschreibe()` is: the address is loaded from the vtable at run time. Compare with the direct `call Vehicle::fahre` for the non-virtual method. Then in GDB:

```
(gdb) break main
(gdb) run
(gdb) next 3
(gdb) print *p
(gdb) print p->_vptr.Vehicle[0]
(gdb) info vtbl p
```

### Task 5: A second derived class and a heterogeneous array

```c++
class Fahrrad : public Vehicle {
public:
    void beschreibe() const override { std::printf("Fahrrad bei %d m\n", position); }
};

Auto a; Fahrrad f;
Vehicle *fuhrpark[] = { &a, &f, &a };
for (Vehicle *v : fuhrpark) { v->fahre(5); v->beschreibe(); }
```

One loop, three objects, two different methods: this is what "abgeleitete Typen können polymorph verwendet werden" means.

### Task 6: Virtual destructors

Allocate on the heap through a base pointer:

```c++
    Vehicle *h = new Auto;
    delete h;
```

Give `Auto` a destructor that prints, and `Vehicle` a non-virtual one. The `Auto` destructor does not run. Make the base destructor `virtual` and check again. Compile with `-Wall -Wextra -Wnon-virtual-dtor` to let the compiler warn about this class of bug.

### Task 7: Slicing

```c++
    Vehicle copy = a;      // copies only the Vehicle part
    copy.beschreibe();
```

Even with `virtual`, this prints the base version: `copy` *is* a `Vehicle`, the `Auto` part was sliced off. Polymorphism works through pointers and references, not through copies. Change the line to `Vehicle &ref = a;` and compare.

## Questions

- What does `class Auto : public Vehicle` mean for the memory layout of an `Auto` object? Use the `sizeof` values as evidence.
- Why did `p->beschreibe()` call the base version in Task 2 and the derived version in Task 3?
- What is a vtable, who fills it, and when is it consulted? Compare it with the function-pointer table you wrote in C.
- What does `override` add, given that the program compiles without it?
- Why must a base class with virtual functions have a virtual destructor?
- What is object slicing, and why does it not happen with references?

## Advice

Inheritance is cheap to write and expensive to undo: once code depends on a hierarchy, every change to the base class touches every derived class. The Core Guidelines therefore say to use it for *interfaces* (a base class with virtual functions and no data) and to prefer composition for reuse of implementation. The next exercise is about exactly that choice.

Two rules to keep: a base class with virtual methods gets a virtual destructor, and polymorphic objects are handled by pointer or reference, never by value.
