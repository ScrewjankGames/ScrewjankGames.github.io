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

I've recently started working on a new [game engine](https://github.com/ScrewjankGames/ScrewjankEngine), and I decided to start by writing custom memory allocators. This post will go over some of the basics of how C++ programs can handle memory, and provide a look at how I implemented some of the Screwjank Engine's memory allocators. 

The source code can be found [here](https://github.com/ScrewjankGames/ScrewjankEngine/tree/master/Engine/Source/Runtime/system/include/system/allocators) and [here](https://github.com/ScrewjankGames/ScrewjankEngine/tree/master/Engine/Source/Runtime/system/src/allocators).
{: .text-center .notice--info}

<figure style="width: 300px" class="align-center">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/system32.jpg' | absolute_url }}" alt="Pick it up.">
  <figcaption><a href="https://www.instagram.com/system32comics/">Photo from System 32 Comics</a></figcaption>
</figure> 

# The Allocator Interface
Every allocator in the Screwjank Engine implements the following interface:

```cpp
class Allocator
{
public:
  virtual void* Allocate(size_t size, size_t alignment) = 0;

  virtual void Free(void* memory) = 0;
};
```

# The Unmanaged Allocator
This was the first allocator I implemented. It's a wrapper around `malloc()`, with an allocation counter for leak detection:

```cpp
class UnmanagedAllocator : public Allocator
{
public:
  ~UnmanagedAllocator()
  {
    SJ_ASSERT(m_ActiveAllocationCount == 0) // Hey look, memory leak detection
  }

  void* Allocate(size_t size, size_t alignment = alignof(max_align_t))
  {
    m_ActiveAllocationCount++;
    return malloc(size);
  }

  void Free(void* memory)
  {
    m_ActiveAllocationCount--;
    free(memory);
  }

private:
  size_t m_ActiveAllocationCount;
};
```

# Memory Alignment
For the remaining allocators, it's important to understand the importance of memory alignment.  Jonathan Rentzsch from IBM has an amazing rundown of [everything you need to know](https://developer.ibm.com/technologies/systems/articles/pa-dalign/). I highly suggest you give it a glance.

The short of it is that your CPU accesses memory in chunks, so the memory addresses for your data structures and their members need to begin at those boundaries. 

If you don't, depending on your hardware:
* Your game will run slower.
* Or your game will crash, possibly catastrophically.

To get the alignment requirement for a type in C++ you can use the `alignof()` operator. When allocating memory for an instance of that type, the memory address needs to be evenly divisible by the alignment requirement.

Another way to express the alignment requirement is:

Let m be the memory location in question
Let a be the alignment requirement

An address is aligned iff (if and only if):


$$
\begin{equation}
\mathbf{X}_{n,p} = \mathbf{A}_{n,k} \mathbf{B}_{k,p}    \label{test}
\end{equation}
$$ 

 m & a = 0

```cpp
uintptr_t GetAlignmentOffset(size_t align_of, const void* const ptr)
{
    return (uintptr_t)(ptr) & (align_of - 1);
}
```

To get the next aligned address:

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
I like to pretend this new engine's going to run on a Playstation one day. So, with that in mind, carefully managing when and how memory is acquired is a must. As studios like [Naughty Dog (see 36:27)](https://www.gdcvault.com/play/1022186/Parallelizing-the-Naughty-Dog-Engine) and [SCE London](https://www.gdcvault.com/play/1023005/Building-a-Low-Fragmentation-Memory) can demonstrate: memory allocators are a critical piece of engine architecture. 
Here's some useful shit:

* https://www.gamedev.net/tutorials/programming/general-and-gameplay-programming/c-custom-memory-allocation-r3010/
* http://bitsquid.blogspot.com/2010/09/custom-memory-allocation-in-c.html