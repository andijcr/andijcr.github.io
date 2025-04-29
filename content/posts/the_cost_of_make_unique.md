+++
title = 'The Cost of `std::make_unique`'
date = 2025-04-29T00:00:00+02:00
+++

## Intro

What I'm about to write is fairly obvious, but it's always interesting to
observe this kind of behavior in the assembly.

A common pattern for factory functions is to return by value.
This approach is supported by Return Value Optimization (RVO), which ensures that the return value is constructed directly in the final memory location, skipping intermediate copies.
This process works even if the destination memory is located on the heap.

Heap memory should be managed with some sort of smart pointer, and to hide naked `new`, we can use `make_...` functions.

However, while RVO and `make_` functions can work together, they are not optimal in combination: the use of `make_` functions forces the constructor's arguments to reside on the stack, which means that they will be copied into the heap destination. Let's take a look at some assembly code to illustrate this.

## The experiment

This is the type that we will construct from a factory. It's "big" just to reiterate the kind of cost we could incur.

```c++
using big_obj = std::array<int, 600>;
```

We will build it from this factory and return a value. Maybe we are
reading data from a file to populate the result? It's `noexcept` so that the compiler generates a simpler assembly.

```c++
auto big_obj_factory() noexcept -> big_obj;
```

### `std::make_unique`

So we want to create a std::unique_ptr<big_obj> from big_obj_factory().
A standard solution could be this:

```c++
auto use_make_unique() -> std::unique_ptr<big_obj> {
    return std::make_unique<big_obj>(big_obj_factory());
}
```

The assembly shows the potential pitfall of this approach:
```assembly
use_make_unique():                   # @use_make_unique()
        push    r15
        push    r14
        push    rbx
        sub     rsp, 2400            # allocate stack space for big_obj
        mov     r15, rdi
        mov     r14, rsp
        mov     rdi, r14
        call    big_obj_factory()    # call big_obj_factory with a pointer to our stack
        mov     edi, 2400
        call    operator new(unsigned long) # allocate a big_obj with new
        mov     rbx, rax
        mov     edx, 2400
        mov     rdi, rax
        mov     rsi, r14
        call    memcpy               # copy our stack big_obj on the heap 
        mov     qword ptr [r15], rbx
        mov     rax, r15
        add     rsp, 2400
        pop     rbx
        pop     r14
        pop     r15
        ret
```

This is what's essentially happening: to invoke make_unique, we construct a big_obj on the stack since this is where parameters go. Then, the value is moved to the final location, and the assembly of this function is virtually identical to the previous one.

```c++
auto pretend_use_make_unique() -> std::unique_ptr<big_obj> {
    auto obj = big_obj_factory();
    return std::unique_ptr<big_obj>(new big_obj(std::move(obj)));
}
```

### Using `std::unique_ptr` directly

To ensure that big_obj_factory constructs the value directly on the heap, we need to skip the stack altogether. This works, at the cost of using `new`:

```c++
auto use_naked_new() -> std::unique_ptr<big_obj> {
    return std::unique_ptr<big_obj>{new big_obj{big_obj_factory()}};
}
```

And the assembly shows that we get the desired behavior:

```assembly
use_naked_new():                     # @use_naked_new()
        push    r14
        push    rbx
        push    rax
        mov     r14, rdi
        mov     edi, 2400
        call    operator new(unsigned long)   # allocate a big_obj on the heap
        mov     rbx, rax
        mov     rdi, rax                      # pass the pointer to big_obj_factory           
        call    big_obj_factory()             # call big_obj_factory to populate the heap space
        mov     qword ptr [r14], rbx
        mov     rax, r14
        add     rsp, 8
        pop     rbx
        pop     r14
        ret
```

### Using a wrapper constructor

The issue with `make_unique` arises because we materialize the result of `big_obj_factory` on the stack, since it needs to be passed as a parameter.
However, we can achieve a similar outcome to using a raw `new` by embedding the factory directly within the constructor, as this wrapper does.
That said, this approach somewhat undermines the purpose of the factory pattern and may violate the principle of least astonishment, as it hides much of the complexity within the constructor wrapper that utilizes the factory.

```c++
struct big_obj_wrapper {
    big_obj _detail{big_obj_factory()};
};

auto use_a_wrapper() -> std::unique_ptr<big_obj_wrapper> {
    return std::make_unique<big_obj_wrapper>();
}
```

The assembly looks similar to use_naked_new.
Note, however, the call to memset: it results from calling the default constructor of std::array.
This is likely a missed optimization since compilers (allegedly) find it difficult to prove that a write to 0 is not needed.

```assembly
use_a_wrapper():                     # @use_a_wrapper()
        push    r14
        push    rbx
        push    rax
        mov     r14, rdi
        mov     edi, 2400
        call    operator new(unsigned long)   # allocate a big_obj on the heap
        mov     rbx, rax
        mov     edx, 2400
        mov     rdi, rax
        xor     esi, esi
        call    memset                        # call the base constructor of big_obj (memset to zero for std::array)
        mov     rdi, rbx                      # pass the pointer to big_obj_factory 
        call    big_obj_factory()             # call big_obj_factory to populate the heap space
        mov     qword ptr [r14], rbx
        mov     rax, r14
        add     rsp, 8
        pop     rbx
        pop     r14
        ret
```

## Final Considerations

All in all, this is kind of an unneeded optimization, something that _might_ make sense in an embedded setting. Still, I found it interesting to analyze this behavior and see how these two features compose.

Bonus: gcc on x64 generates [`rep movsd`](https://www.felixcloutier.com/x86/rep:repe:repz:repne:repnz#description), which is an inlined form of memcpy, and I think it's cool.
