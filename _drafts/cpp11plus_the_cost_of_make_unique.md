link to godbolt: https://gcc.godbolt.org/z/BBt4vA

```c++
#include <array>
#include <memory>
/*
 * a somewhat artificial example about the cost of std::make_unique.
 * problem: i want to use a factory to creante an object on the heap
 * std::make_unique uses stack space to implements safety, so a clean
 * implementation can incur in some costs. being less clean (a factory that
 * returns directly a unique_ptr) or less safe (skipping make_unique) are
 * options worth exploring, when the programmer chooses so
 */

// a generic big obj
using big_obj = std::array<int, 600>;

// forward define the factory for big_obj. implementation left to the reader
auto big_obj_factory() noexcept -> big_obj;

// this uses stack space
auto full_safe() -> std::unique_ptr<big_obj> {
  return std::make_unique<big_obj>(big_obj_factory());
}

// this is what make_unique is doing under the hood (kinda)
auto full_safe_manual() -> std::unique_ptr<big_obj> {
  auto obj = big_obj_factory();
  return std::unique_ptr<big_obj>(new big_obj(std::move(obj)));
}

/*
 * both version allocate space on the stack
 * so in case of exceptions, no memory pointer is lost (note that the noexcept
 * on the factory does nothing) 2 tidbits: sub rsp ... reserve stack space, rep
 * ... is the opcode that implements the while loop to copy memory from stack to
 * the heap https://www.felixcloutier.com/x86/rep:repe:repz:repne:repnz
 */

// not using make_unique poses a risk of losing a memory ref, but note that no
// temp stack space is used
auto unsafe() -> std::unique_ptr<big_obj> {
  return std::unique_ptr<big_obj>{new big_obj{big_obj_factory()}};
}

// wrapper that used the factory in the constructor
struct big_obj_wrapper {
  big_obj _detail{big_obj_factory()};
};

// wrapping the factory in the consturctor lets us use again make_unique at cost
// of less flexibility the rationale is that the constructor is now responsible
// to not be broken not that this wrapper incurs in a bit of hoverhead if
// compared with unsafe(): the member _details is constructed by first zeroing
// the content, then using the factory to initialize it
auto wrapper_safe() -> std::unique_ptr<big_obj_wrapper> {
  return std::make_unique<big_obj_wrapper>();
}
```

