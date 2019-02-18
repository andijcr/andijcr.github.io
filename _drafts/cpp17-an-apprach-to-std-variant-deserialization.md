---
title: cpp17 A Proxy for when you want to construct an object after you start using
  it
date: 2019-02-18 18:31:30.854000000 Z
categories:
- blog
tags:
- cpp
- c++
- cpp17
- c++17
layout: post
---

## Or: using RAII in a RAII-unfriendly codebase

#### Backstory - You can skip this

Working with an existing codebase, i was tasked with writing a new module that would extend the functionalities of a subsystem that i did not fully understand, with complex relationships with other subsystems and a not clear (to me) lifecycle.

The requirements asked for an "Executor" that would execute a set of compound actions based on a runtime configuration.

I isolated my work by creating an interface of how to interface with my module, pretending that i would have not integrate my module with the codebase

At the ending of the dev cycle i had to face my daemons and understand what the system wanted to do with my module. My design wasn't very off from the desired result, exept for a detail: the configuration guiding the Executor action could be changed multiple times during the app lifetime, or not be present at all. 

At the pratical level, in the codebase each method was composed by many  
\`\`\`if( configuration is not null ) execute(configuration) \`\`\`, and i wanted a more elegant way to do this.  
Why raii unfriendly? because the owner of the configuration ( the resource)  is not the consumer, and the use of the consumer has to be mediated by the external availability of the resource.

### The Problem

So i have to install my code into various methods, where each method follow this basic structure:

    ...
    if( this->resource != NULL ){
    	return this->helper_executor();
    }
    if( this->other_resource != NULL ){
    	return this->alternate_helper_executor();
    }
    ...

but my class if written to own the resource like so:

    ExternalExecutor my_executor{std::move(this->resource)}; 

and i would need to add method to reset the resource or relinquishing it without destroying \`\`\`my_executor\`\`\`.

The alternative is to use a proxy that manages the lifecycle of ExternalExecutor. Optionally, it could offer sintattic sugar to execute code based on the aliveness of the wrapped object, like so:

    // default state is not alive state:
    DelayProxy<ExternalExecutor> optional_executor{};
    
    // so this code is not executed, because the object has not been created yet
    auto res = optional_executor([&](ExternalExecutor &exec) -> bool {
    	std::cout << "not invoked\n";
    	return exec.action_one();
    });
    
    // prints "nothing"
    std::cout << (res? to_string(*res) : "nothing" ) << "\n";
    
    // now we create the underlying object
    optional_executor.setup(std::move(some_resource));
    // i should be able to receive back a result - even void
    optional_executor([&](ExternalExecutor &exec) -> void {
    	std::cout << "actually invoked\n";
    	exec.action_two_void_return();
    });
    

### My Solution

my proxy class uses a \`\`\`std::unique_ptr\`\`\` to manage the underlying object. I could have opted to store it inside the proxy, but that would be wasted space when the object is dead.

Next is the method to do something with the object if present:

    // template declaration omitted
    auto operator()(Fn fun){	// accept code as a lambda
    		if (subject)		// the wrapped object, as a std::unique_ptr
    			return std::invoke(fun, *subject);	//execute if present
    }

Pretty simple, but in the actual implementation i went for a more complete route:

    #include <memory>
    #include <optional>
    #include <type_traits>
    #include <utility>
    
    namespace lazy {
    
    template <typename T> class DelayProxy {
    	std::unique_ptr<T> subject{};		// the wrapped object
    
      public:
    	// delayed constructor: forward wathever argument i get to the T contructor,
        // passing through std::unique_ptr
        template <typename... P> 
        void setup(P &&... v) { 
        	subject = std::make_unique<T>(std::forward<P>(v)...); 
        }
    	
        // with this method i can reset the internal state
        void reset() { subject.reset(); }
    
    	// with this we can use the object.
        // the result should be a std::optional<Result>,
        // but if the code does not return anything, then return directly void
        // the dependant typenames help choose the right signature for this method
    	template <typename Fn, typename FnRet = std::invoke_result_t<Fn, T &>,
    			  typename Ret = std::conditional_t<!std::is_void_v<FnRet>, std::optional<FnRet>, void>>
    	auto operator()(Fn fun) noexcept(std::is_nothrow_invocable_v<Fn, T &>) -> Ret {
    		if (subject)
    			return std::invoke(fun, *subject);
            // this bit is to return an empty optional, in case we need the return type
    		if constexpr (!std::is_void_v<FnRet>)
    			return Ret{};
    	}
    };
    
    }

```cpp
```