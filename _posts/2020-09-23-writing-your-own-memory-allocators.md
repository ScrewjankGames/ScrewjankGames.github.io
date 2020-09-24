---
title: "Writing Your Own Memory Allocators in C++"
date: 2020-09-23T21:11:00-04:00
categories:
  - Engine Programming
tags:
  - C++
  - Memory Management
  - Over-Engineering
last_modified_at: 2020-09-23T21:11:00-04:00
---

Because nobody's around to tell you not to.

{% include toc %}

# Introduction
<figure style="width: 330px" class="align-right">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/system32.jpg' | absolute_url }}" alt="Pick it up.">
  <figcaption><a href="https://www.instagram.com/system32comics/">Photo from System 32 Comics</a></figcaption>
</figure> I've recently started working on a new [game engine](https://github.com/ScrewjankGames/ScrewjankEngine), and I decided to start by tackling a problem that I've completely ignored in all of my game projects to date: Custom Memory Allocation. This post will go over some of the basics of how C++ programs can request and manage RAM, and provide an in-depth look at how I implemented some of Screwjank's memory allocators.

Though unnecessary for small PC games, wrangling your memory in a deliberate way is not optional when making games that require every ounce of performance the hardware has to give. As studios like [Naughty Dog (see 36:27)](https://www.gdcvault.com/play/1022186/Parallelizing-the-Naughty-Dog-Engine) and [SCE London](https://www.gdcvault.com/play/1023005/Building-a-Low-Fragmentation-Memory) can demonstrate, memory allocators are a critical piece of engine architecture in the big leagues. Plus they're fun to write.

# How does memory work
When your program runs it gets some memory from your operating system when it starts, and can ask for more memory after the program starts running. As far as C++ is concerned, your program gets it's grubby little mitts on memory in one of three ways:
  * Statically
    * Fixed-size allocations performed before your program's `main()` function is called
  * Automatically
    * Fixed-size allocations performed automatically when your program enters functions
  * Dynamically
    * Arbitrarily sized allocations performed by `malloc()` or `new()`

The method you use to request memory effects the [Storage Duration](https://en.cppreference.com/w/cpp/language/storage_duration#Storage_duration) and [Lifetime](https://en.cppreference.com/w/cpp/language/lifetime) of the object that uses that memory. In other words:
  * Storage duration is a property of an object's memory that defines the time in which it is accessible to your program. The time starts when the memory is allocated and ends when it's de-allocated.
  * Lifetime is a property of an object or reference that defines the time in which is is usable as an instance of that type. The lifetime is less than or equal to, and cannot exceed, the storage duration of the memory it uses. 
    * For primitive types (`bool`, `int`, `float`, `double`, etc.) the object's lifetime is the same as the storage duration. 
    * For `struct` and `class` types the object's lifetime is is typically the time between the calls to it's constructor and destructor. 

| Syntax      | Description |
| ----------- | ----------- |
| Header      | Title       |
| Paragraph   | Text        |

### A note on Physical vs. Virtual memory
Modern operating systems have the ability to map more memory than they have RAM. They do this using virtual memory addresses, which are mapped to and swapping in and out of physical memory addresses. Your programs probable get virtual memory addresses from `new()` and `malloc()`, which are translated to actual addresses behind the scenes. 

This has implications in more advanced custom memory allocation strategies, but won't be covered in further detail in this post.

# Memory Allocators
Why write you own allocators? What does it take? What do you need to know? Introduce the different types of allocators.

# The Allocator Interface
Describe the basic interface for all the allocators in the Screwjank Engine
Allocate
Free

HELPER FUNCTIONS:
Allocator.New
Allocator.Delete
Allocator.AllocateType<T>

### MemoryStats
Total
Active

# Unmanaged Allocators
Wrangling the global new/delete operators
Wrapping malloc and free
Leak Detection!

# Stack Allocators
How do they work?

# Pool Allocators
How do they work?

# Free List Allocators
How do they work?

# Proxy Allocator
wtf is it

# Moving Forward
What extensions can or should be made?

# References
Here's some useful shit:
* https://developer.ibm.com/technologies/systems/articles/pa-dalign/
* https://www.gamedev.net/tutorials/programming/general-and-gameplay-programming/c-custom-memory-allocation-r3010/
* http://bitsquid.blogspot.com/2010/09/custom-memory-allocation-in-c.html