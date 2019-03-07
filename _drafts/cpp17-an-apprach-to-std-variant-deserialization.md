---
title: cpp17 A Proxy for when you want to construct an object after you start using
  it
date: 2019-03-07 09:56:44.768000000 Z
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
```if( configuration is not null ) execute(configuration) ```, and i wanted a more elegant way to do this.  
Why raii unfriendly? because the owner of the configuration ( the resource ) is not the consumer, and the use of the consumer has to be mediated by the external availability of the resource.

### The Problem

So i have to install my code into various methods, where each method follow this basic structure:

    ...
    if( this->resource != NULL ){
    	return this->helper_executor(this->resource);
    }
    if( this->other_resource != NULL ){
    	return this->alternate_helper_executor(this->other_resource);
    }
    ...

but my class if written to own the resource like so:

    ExternalExecutor my_executor{std::move(this->resource)}; 

and i would need to add method to reset the resource or relinquishing it without destroying ```my_executor```.

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

my proxy class uses a ```std::unique_ptr``` to manage the underlying object. I could have opted to store it inside the proxy, but that would be wasted space when the object is dead.

Next is the method to do something with the object if present:

    // template declaration omitted
    auto operator()(Fn fun){	// accept code as a lambda
    		if (subject)		// the wrapped object, as a std::unique_ptr
    			return std::invoke(fun, *subject);	//execute if present
    }

Pretty simple, but in the actual implementation i went for a more complete route:
(gist link)[https://gist.github.com/andijcr/405bf98c60987165deda8fc358bfa349]

    template <typename T> class DelayProxy {
        std::unique_ptr<T> subject{};

      public:
        template <typename... P> void setup(P &&... v) {
            subject = std::make_unique<T>(std::forward<P>(v)...);
        }
        void reset() { subject.reset(); }

        template <typename Fn, typename FnRet = std::invoke_result_t<Fn, T &>,
                  typename Ret = std::conditional_t<!std::is_void_v<FnRet>, std::optional<FnRet>, void>>
        auto operator()(Fn fun) noexcept(std::is_nothrow_invocable_v<Fn, T &>)
            -> Ret {
            if (subject)
                return std::invoke(fun, *subject);

            if constexpr (!std::is_void_v<FnRet>)
                return Ret{};
        }
    };
    
to use it, i wrap my code in a lambda. 
This lambda will receive (if it exists) the object as a reference.

	// default state, SomeClass is not created
    DelayProxy<SomeClass> delay;

	// so this lambda will not be executed
	auto res = delay([&](auto &sc) -> int {
		std::cout << "not invoked\n";
		std::cout << sc.a << " " << sc.c << "\n";
		return sc.a + sc.c;
	});

	std::cout << res.value_or(-1) << "\n";

	// setup just forward the arguments to the constructor
	delay.setup(3, 2);
    
    // so now this lambda is executed
	delay([&](auto &sc) {
		std::cout << "actually invoked\n";
		std::cout << sc.a << " " << sc.c << "\n";
	);
    
The return type of the lambda dictate the return type outside: for a ```void``` lambda ```void``` is returned, for whaterver ```T```, ```std::optional<T>``` is returned.

### Personal use experience

I have to say that using this code is almost painless, when used for small bits.
For one i don't have troubles anymore for a missing nullcheck, and the syntax (for me at least) flows effortlessly.
On the other hand, this system **is** a glorified ```if not null```, even tought it adds a bit of compile time guarantees, and dealing with the returned std::optional<> can be annoying, since we lack some convenience composition methods in the std library.
In general it's an interface between two worlds, so it's better to restrict the programming time spent there.

### Code generation

i tested gcc 8.3 and clang trunk (it wont compile on 7 for some missing templates in type_traits, still have to investigate)

<iframe width="800px" height="200px" src="https://gcc.godbolt.org/embed-ro#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAKxAEZSAbAQwDtRkBSAJgCFufSAZ1QBXYskwgA5NwDMeFsgYisAag6yAwgDMRignlQsmDDdg4AGAIJyFSlZnVaAtpmckAnmcs2u8xcpqGpqoAA4GRibe1rYBDk6aBB6hmAD6BMRMhILRvv72QVoiBgyEXrLmMdbGroKhTBKqzABeHuoA7HxVVgRuocy9CUkpNY4AKmaqSkyCgqoAIpjMHgAKxKgAHm0cnT6WAJyCBOggIHp4AI4iaeHEwRMVqoIiAEZUmMgEO3zt8xpdNmsqlUoVepWQID2Fn2vWc/SYg2Cw0wowAdOjVCtJgA3VB4dBPTAEEShCArdRcABs3Ep6NRqmxAEoOgCDgdnm8PgQnPMnsdTs4mABrNLnK6Ye5mCBHE4gbQkADuTGI6GCWIqECZdMZ/yh%2Bx2fyq%2B1x%2BNUxEwgiJEGZ3yer3en1R5stBGt/w6hsBgJhfQGjiRyRRTFcqgAYixSKpkaMwywAEpEnl82UKXEi1LOkQMAjpYLhyNjCnUiqkPVs6HA6PBxwJ7kaXky05oFj4CLGBi5rRgMCNkB4QSpE3oQd5%2BNEsyR3thNtRLTh2sThl41UVXIHJjFVCqMKYTJEYjWiDh1S6FjMlioTAbCThaX8vsDi8EBDrBWpVOoZBMF4MNLY0cFkWZg6kaBwALSTLWLJltCeDaKo0r2lyIHeuW%2BzmsSxAsMmpwfiKECnpGABUHIOgQOqyKyoGwfBzZHFeoTEAh3a9v2g7LiOc5jl8FQoWhBwYWI2ELrsvy6qBYkxGJlFQjYkleiIggKMAqijHUDSOC05RUVYRzECInyqAAyqgriaMwszQUaCjckw4nejZUz2QcJlmRZggQI5qRMJGXnIMyICqHZnTeQakacCFEW8t8Bp7NJOmLMsaybOUmiuZg5kzDkjxYMsznWI5jEpakp7WlZXrrpuZoWkmuVMB4EAcAArHwVLNfMEAbkQRaCP5qgQY8jkxdR6FEkJTzIKiTAUjwE2opwMkSfMFE6QccEIc6qLYiY1ypCQEBgbQzLdvW/VHcNVjAldV2CVhZ32ddsXdAcdUeKiLokhAsiRlwK0%2BMCepdVuzqDrVSz1Y1LU0u1nVVTSvXMgN2CqENuwjb1U2qERp2yPl%2BrQrd2EY9N/BzQtVH6r8f3dNdhPVQO2JbTtaT7Yd1NyZ6PjpZlsxESCGRJiwWYMLc%2BVWIVGQlXoZUXf9V2ORhSaHQ98vaBAtyqCdsi8kLDAixkOpo5d103Ymp23EjJO8PzxBI%2BTcsA/JerrRAita7ybMXSbpuYdhyuLcbjuc0Cqvqxk7u6/rxCGwC3vArlRKOKLAePfJ10a6dLCYAqxmmRl7lfT97MO9Vdba/dKehxrEfC7cMcwfsFtmNN2Pl7jAdoYr5sZJbM023bKtB7JtNjXdGHOb8UiMow0hNVIpAsNIFjz6g0iaPwpPCGIjRyLQ88EEvU/T0KIAACyn6ifin/s%2BwAByn1w%2By0E1p%2B3/sM9SKf8%2BL1Iy%2BkKvUh56CBABYUgB9f5T1IHAWASA0BwjwL%2BMgFAIBwNCAg3cIBgC32%2BtoBBvRiDAIgC8Q%2BpAXgKGVB4aQe9SBwNcCwAgAB5FgDBKEQNIFgQUbBfwkPwOaT4eBsQWhIVeD4xRJBSGoTZJYJDSgvEyMQDwIQsBUP3sQPAzhD7T2YGwFAG9eCMDwC8YBkBp7TkMCwYBUgwIynrJwXg/BaDtH6gwrgQDRDiEkEdD%2Bc8F4kIARsW%2BlIwKUlPqoYAyBkCqFvqiWQCFcCEBIBSWQ9BVAhHgYgpJR1Ul6J4PvTRx8QBcC4KiU%2ByTKSUlvn4doTVaCONvh/L%2BpANG0AsKAn%2Bf8AFAJAWAzRUCYCIBQKZNBiDyCUFQeg4gKBtHAHSPpFgQpSC4OzLuQhxC2FkOMAolRNDTJ0MYcw1hf8OGsGANwthvCuQCKEWwkRyAxHbKkQwGRhj5GKIwOI6hGR1EqK0Sc3R9j9GyOMdaf%2B4RzGWOsccWxOTHHONcUIdxEg6BaNnt/Px0gAlBJCVMaZCEMh6CFMyCA8T9yZMjGk4Zu4km/WyQC3JPSIGMgKbIWQMSWXso5RyhpaK2GdKEN08By8mUf3hc01pvjeXSDyYy6egiCHmLPkAA%3D%3D%3D"></iframe>

Cursory, the generated code is not much worse than the had written one, with 43 instructions vs 22 for gcc, and 33 vs 21 for clang.
I think part of the difference is that the compiler has troubles seeing through the abstractions, and going for a safer route for creation and distruction, while having no troubles compiling the lambda in simple ```if not null``` checks.

I can probably get something back by specializing the creation/destruction. 