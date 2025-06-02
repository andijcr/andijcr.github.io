+++
title='Reducing Variadic Template Code Bloat with Type Erasure in C++'
date=2025-06-01
+++

## A Type Erasure Pattern for Variadic Templates

[Fmtlib](https://fmt.dev/) is a great library, significantly improving upon the previous status quo of formatting in C++.

One of the clever techniques it employs relates to how it handles the code bloat (or explosion) caused by variadic template formatting functions.

### Understanding Template Instantiation and Code Bloat

What is the issue? When we invoke a template, the compiler will instantiate it, either generating a new procedure or inlining it at the call site.

Each new concrete template instantiation will be independent from other type instantiations; there is no general way for the compiler to reuse a common implementation across different instantiations.

This phenomenon is called template bloat (or code bloat/template explosion) and it can be problematic when considering the generated code size.

With a variadic template, the problem is even worse.

Since the number and types of arguments are now unbounded at the template definition site, each invocation might generate and inline a unique version of the function at the call site, leading to a lot of repetitive code.

Invoking the function again with the same sequence of types might not reuse the previous instantiation but instead inline it again at the call site.

### Illustrating the Problem: A Basic Variadic Printer

Let's set up an example with a formatting function that prints its arguments separated by a space and ending in a newline. I'll write something that resembles the core functionality of `fmtlib`.

First, let's define some basic type formatting functions.

The user can provide an overload of `print_fn` to print their custom type, while the library will provide a set of base formatters:

```c++
// provide an overload of print_fn to output your type
void print_fn(std::string_view);
void print_fn(int);
void print_fn(double);

```

With these in place, we can write this kind of variadic formatter:

```c++
// simple variadic printer: call print_fn on every argument. look at the code
// explode!
template <typename THead, typename... Ts>
void variadic_print(THead const& start, Ts const&... args) {
    print_fn(start);
    (
        [&] {
            print_fn(std::string_view(" "));
            print_fn(args);
        }(),
        ...);
    print_fn("\n");
}

void variadic_printer_user() { variadic_print(1, 20.4, -10, "numbers"); }

void variadic_printer_user_multiple() {
    variadic_print(1, 20.4, -10, "numbers");
    variadic_print(2, 20.3, -11, "other numbers");
}
```

The assembly below shows how the `print_fn` calls were inlined into the instantiations of `variadic_print` by the compiler.

```assembly
void variadic_print<int, double, int, char [8]>(int const&, double const&, int const&, char const (&) [8]) (.isra.0):
        push    r12
        mov     r12, rsi
        push    rbp
        mov     rbp, rdx
        push    rbx
        mov     rbx, rcx
        call    print_fn(int)
        mov     edi, 1
        mov     esi, OFFSET FLAT:.LC0
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        movsd   xmm0, QWORD PTR [r12]
        call    print_fn(double)
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, 1
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        mov     edi, DWORD PTR [rbp+0]
        call    print_fn(int)
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, 1
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        mov     rdi, rbx
        call    strlen
        mov     rsi, rbx
        mov     rdi, rax
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        pop     rbx
        mov     edi, 1
        pop     rbp
        mov     esi, OFFSET FLAT:.LC1
        pop     r12
        jmp     print_fn(std::basic_string_view<char, std::char_traits<char>>)
variadic_printer_user():
        sub     rsp, 24
        mov     rax, QWORD PTR .LC2[rip]
        mov     ecx, OFFSET FLAT:.LC3
        mov     edi, 1
        lea     rdx, [rsp+4]
        lea     rsi, [rsp+8]
        mov     DWORD PTR [rsp+4], -10
        mov     QWORD PTR [rsp+8], rax
        call    void variadic_print<int, double, int, char [8]>(int const&, double const&, int const&, char const (&) [8]) (.isra.0)
        add     rsp, 24
        ret
variadic_printer_user_multiple():
        sub     rsp, 24
        mov     rax, QWORD PTR .LC2[rip]
        mov     ecx, OFFSET FLAT:.LC3
        mov     edi, 1
        lea     rdx, [rsp+4]
        lea     rsi, [rsp+8]
        mov     DWORD PTR [rsp+4], -10
        mov     QWORD PTR [rsp+8], rax
        call    void variadic_print<int, double, int, char [8]>(int const&, double const&, int const&, char const (&) [8]) (.isra.0)
        mov     edi, 2
        call    print_fn(int)
        mov     edi, 1
        mov     esi, OFFSET FLAT:.LC0
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        movsd   xmm0, QWORD PTR .LC4[rip]
        call    print_fn(double)
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, 1
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        mov     edi, -11
        call    print_fn(int)
        mov     edi, 1
        mov     esi, OFFSET FLAT:.LC0
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        mov     edi, 13
        mov     esi, OFFSET FLAT:.LC5
        call    print_fn(std::basic_string_view<char, std::char_traits<char>>)
        mov     edi, 1
        mov     esi, OFFSET FLAT:.LC1
        add     rsp, 24
        jmp     print_fn(std::basic_string_view<char, std::char_traits<char>>)
```

The assembly shows that `variadic_print` effectively expands to a sequence of manual calls to `print_fn`.

Interestingly, the compiler generated a distinct `variadic_print<int, double, int, char[8]>` instantiation to use for both `variadic_printer_user` and the first line of `variadic_printer_user_multiple`.

However, it could not reuse this instantiation for the second line because of the different sizes of string literals, which are represented as separate types: `char[8]` and `char[14]`.

This shows that even including a different-sized string literal changes the type signature of `variadic_print`, requiring a new instantiation.

In general, this code bloat is a genuine concern and can be painful in contexts where binary size is important.

### A Solution: Type Erasure for Efficient Codegen

We can do better. We can use type erasure to reduce code generation (codegen), similar to how `printf` operates internally, while retaining the ergonomics and features of a variadic interface.

The trick is to convert the variadic list of types to an array of a single, type-erased object type.

This object will be capable of selecting the correct `print_fn` overload for its stored runtime value:

```c++
// type-erased formatting function
void vprint(std::span<const carrier>);

// frontent for vprint
template <typename... Ts>
void print(Ts const&... args) {
    vprint(std::array{
        carrier{args}...,
    });
}
```

A temporary `std::array` is created with the `args` converted to the common `carrier` type and passed to the non-templated, type-erased `vprint` function.

As long as we are careful not to create dangling references within the `carrier`'s constructor (e.g., by storing a pointer to a temporary created *inside* the constructor itself), we are fine since the `carrier` objects in the array will all live until the end of the statement (the call to `vprint`).

### Introducing the `carrier` Struct

The `carrier` is conceptually a `void` pointer to the value and a function pointer to the formatting function (in `fmtlib`, this would be a formatting object).

There are, however, some optimizations to account for primitive types and string literals to reduce unnecessary indirections and further optimize codegen.

We use a `union` to store either a primitive type or a `void const*` to the value.

The overloaded `carrier::carrier` constructors are responsible for creating a lambda (which becomes a function pointer) that will "remember" the type of `arg` and select the correct `print_fn` overload.

```c++
// type-erased printer argument carrier. the argument has to outlive the
// carrier.
struct carrier {
    // optimization: simple types can be stored inside instead of by pointer
    union fmt_arg {
        int i;
        double d;
        void const* ptr;
    };
    using union_printer = void (*)(fmt_arg const&);

    // type-erased parameter
    fmt_arg arg;
    // type-erased print_fn
    union_printer printer;

    // optimization for ints
    carrier(int i)
        : arg{.i = i},
          printer{
              [](fmt_arg const& arg) { print_fn(arg.i); },
          } {}

    // optimization for doubles
    carrier(double d)
        : arg{.d = d},
          printer{
              [](fmt_arg const& arg) { print_fn(arg.d); },
          } {}

    // optimization for null terminated strings
    carrier(const char* str)
        : arg{.ptr = str},
          printer{
              [](fmt_arg const& arg) {
                  print_fn(reinterpret_cast<char const*>(arg.ptr));
              },
          } {}

    // optimization for string literals: convert them to const char* to reduce
    // codegen. otherwise we would get a new lambda for each length of c_string
    template <size_t N>
    carrier(char const (&strlit)[N]) : carrier{&strlit[0]} {}

    // type erase constructor: generate a type erased lambda for each type,
    // capable of invoking the correct print_fn.
    template <typename T>
    carrier(T const& in)
        : arg{.ptr = &in},
          printer{
              [](fmt_arg const& arg) {
                  print_fn(*reinterpret_cast<T const*>(arg.ptr));
              },
          } {}

    // print the argument
    void print() const { printer(arg); }
};

```

Some details:
1.  The lambdas are captureless. This means that they can be safely converted to function pointers and that they will not dangle or rely on local constructor state once the constructor terminates. In assembly, they will appear as normal non-member functions.
2.  Be cautious with lifetimes: If a `carrier` constructor were to create a temporary object (e.g., `std::string_view temp_sv = some_char_ptr;`) and then store a pointer to that temporary (`arg.ptr = temp_sv.data();`), `arg.ptr` would become a dangling pointer when the constructor exits. The `carrier` must either store small types by value (as done with `int`, `double`) or store pointers/references to objects whose lifetimes exceed the `vprint` call. (Always use tools like AddressSanitizer to catch lifetime issues!).
3.  `int`, `double`, and other small/primitive types are directly stored in the `carrier` object's `union`. This avoids an indirection and can improve cache locality.
4.  String literals have a `char[N]` type that is distinct for each size `N`. To reduce codegen for `carrier` constructors, the `carrier(char const (&strlit)[N])` constructor is templated on `N` and defers to the `const char*` constructor. This gets optimized well by the compiler. If we instead implemented the logic directly in the `char const (&strlit)[N]` constructor, we would get code bloat by generating `N` different constructor instantiations and associated lambdas, all doing essentially the same work.
5.  For any other type `T` (handled by the generic `carrier(T const& in)` constructor), we save a pointer to the argument `in` and instantiate a templated lambda to reverse the type erasure at the point of printing.

### The Result: Examining the Type-Erased Assembly

We can see the effects in the codegen for these two functions:

```c++
void printer_user() { print(1, 20.4, -10, "numbers"); }

void printer_user_multiple() {
    print(1, 20.4, -10, "numbers");
    print(2, 20.3, -11, "other numbers");
}
```

```assembly
printer_user():
        sub     rsp, 72
        mov     rax, QWORD PTR .LC2[rip]
        mov     esi, 4
        mov     rdi, rsp
        mov     QWORD PTR [rsp], 1
        mov     QWORD PTR [rsp+16], rax
        mov     eax, 4294967286
        mov     QWORD PTR [rsp+8], OFFSET FLAT:carrier::carrier(int)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+24], OFFSET FLAT:carrier::carrier(double)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+32], rax
        mov     QWORD PTR [rsp+40], OFFSET FLAT:carrier::carrier(int)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+48], OFFSET FLAT:.LC3
        mov     QWORD PTR [rsp+56], OFFSET FLAT:carrier::carrier(char const*)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        call    vprint(std::span<carrier const, 18446744073709551615ul>)
        add     rsp, 72
        ret
printer_user_multiple():
        push    rbx
        mov     esi, 4
        sub     rsp, 64
        mov     rax, QWORD PTR .LC2[rip]
        mov     rdi, rsp
        mov     QWORD PTR [rsp], 1
        mov     QWORD PTR [rsp+16], rax
        mov     eax, 4294967286
        mov     QWORD PTR [rsp+32], rax
        mov     QWORD PTR [rsp+8], OFFSET FLAT:carrier::carrier(int)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+24], OFFSET FLAT:carrier::carrier(double)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+40], OFFSET FLAT:carrier::carrier(int)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+48], OFFSET FLAT:.LC3
        mov     QWORD PTR [rsp+56], OFFSET FLAT:carrier::carrier(char const*)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        call    vprint(std::span<carrier const, 18446744073709551615ul>)
        mov     rax, QWORD PTR .LC4[rip]
        mov     rdi, rsp
        mov     esi, 4
        mov     QWORD PTR [rsp], 2
        mov     QWORD PTR [rsp+16], rax
        mov     eax, 4294967285
        mov     QWORD PTR [rsp+8], OFFSET FLAT:carrier::carrier(int)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+24], OFFSET FLAT:carrier::carrier(double)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+32], rax
        mov     QWORD PTR [rsp+40], OFFSET FLAT:carrier::carrier(int)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        mov     QWORD PTR [rsp+48], OFFSET FLAT:.LC5
        mov     QWORD PTR [rsp+56], OFFSET FLAT:carrier::carrier(char const*)::'lambda'(carrier::fmt_arg const&)::_FUN(carrier::fmt_arg const&)
        call    vprint(std::span<carrier const, 18446744073709551615ul>)
        add     rsp, 64
        pop     rbx
        ret
```

The assembly shows the `print` function setting up an array of `carrier` objects on the stack (pointers/values and their corresponding printer function pointers) and then calling the `vprint` function.

There is less template machinery going on directly within `printer_user` and `printer_user_multiple`, making it clearer what code is part of the formatting setup versus the actual printing logic within `vprint`.

Also, across the codebase, we are avoiding the combinatorial explosion of function template instantiations needed to satisfy all the type sequences that we would require for our purely variadic format functions.

In a way, this is an extensible and type-safe C++ reimplementation of the variadic mechanism used internally by functions like `printf`.

Finally, we can print any type by defining its printing function, and the templated `carrier` constructor will pick it up automatically:

```c++
struct our {
    int a;
    int b;
};
void print_fn(our const&);

void print_our() { print(our{1, 47}); }
```

### Conclusion: A Glimpse into Library Design

This is just a small part of the techniques used in a library like `fmtlib`; we are missing key aspects such as compile-time format string parsing and the associated compile-time type safety checks against that format string.

Still, I had fun studying and reimplementing this detail of such a library.

### Complete Example on Godbolt

Here is the complete code from this post:
[https://gcc.godbolt.org/z/8j31P1MzM](https://gcc.godbolt.org/z/8j31P1MzM)

```c++
#include <array>
#include <span>
#include <string_view>

// provide an overload of print_fn to output your type
void print_fn(std::string_view);
void print_fn(int);
void print_fn(double);

// simple variadic printer: call print_fn on every argument. look at the code
// explode!
template <typename THead, typename... Ts>
void variadic_print(THead const& start, Ts const&... args) {
    print_fn(start);
    (
        [&] {
            print_fn(std::string_view(" "));
            print_fn(args);
        }(),
        ...);
    print_fn("\n");
}

// type-erased printer argument carrier. the argument has to outlive the
// carrier.
struct carrier {
    // optimization: simple types can be stored inside instead of by pointer
    union fmt_arg {
        int i;
        double d;
        void const* ptr;
    };
    using union_printer = void (*)(fmt_arg const&);

    // type-erased parameter
    fmt_arg arg;
    // type-erased print_fn
    union_printer printer;

    // optimization for ints
    carrier(int i)
        : arg{.i = i},
          printer{
              [](fmt_arg const& arg) { print_fn(arg.i); },
          } {}

    // optimization for doubles
    carrier(double d)
        : arg{.d = d},
          printer{
              [](fmt_arg const& arg) { print_fn(arg.d); },
          } {}

    // optimization for null terminated strings
    carrier(const char* str)
        : arg{.ptr = str},
          printer{
              [](fmt_arg const& arg) {
                  print_fn(reinterpret_cast<char const*>(arg.ptr));
              },
          } {}

    // optimization for string literals: convert them to const char* to reduce
    // codegen. otherwise we would get a new lambda for each length of c_string
    template <size_t N>
    carrier(char const (&strlit)[N]) : carrier{&strlit[0]} {}

    // type erase constructor: generate a type erased lambda for each type,
    // capable of invoking the correct print_fn.
    template <typename T>
    carrier(T const& in)
        : arg{.ptr = &in},
          printer{
              [](fmt_arg const& arg) {
                  print_fn(*reinterpret_cast<T const*>(arg.ptr));
              },
          } {}

    // print the argument
    void print() const { printer(arg); }
};

// type-erased formatting function
void vprint(std::span<const carrier>);

// frontent for vprint
template <typename... Ts>
void print(Ts const&... args) {
    vprint(std::array{
        carrier{args}...,
    });
}

void variadic_printer_user() { variadic_print(1, 20.4, -10, "numbers"); }

void variadic_printer_user_multiple() {
    variadic_print(1, 20.4, -10, "numbers");
    variadic_print(2, 20.3, -11, "other numbers");
}

void printer_user() { print(1, 20.4, -10, "numbers"); }

void printer_user_multiple() {
    print(1, 20.4, -10, "numbers");
    print(2, 20.3, -11, "other numbers");
}

struct our {
    int a;
    int b;
};
void print_fn(our const&);

void print_our() { print(our{1, 47}); }

#if 0
// impl, this is just for test
#include <print>

void print_fn(std::string_view sv) { std::print("{}", sv); }
void print_fn(int i) { std::print("{}", i); }
void print_fn(double d) { std::print("{}", d); }

void vprint(std::span<const carrier> args) {
    if (args.empty()) {
        return;
    }

    args[0].print();

    for (auto i = 1u; i < args.size(); ++i) {
        print_fn(" ");
        args[i].print();
    }

    print_fn("\n");
}

int main() {
    variadic_printer_user_multiple();
    printer_user_multiple();
    print_our();
}

#endif
```
