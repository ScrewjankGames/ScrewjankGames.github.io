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

I've recently started working on a new [game engine](https://github.com/ScrewjankGames/ScrewjankEngine) and, since I have no restraint, task zero is writing custom memory allocators. `malloc()` and `new` are great tools for getting a hold of raw memory, but they don't leave any room for customization or optimization based on how that memory is utilized. Custom allocators perform the same job in a context-aware way, which provides the opportunity to take shortcuts `malloc()` and `new` never could.

<figure style="width: 300px" class="align-center">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/system32.jpg' | absolute_url }}" alt="Pick it up.">
  <figcaption><a href="https://www.instagram.com/system32comics/">Photo from System 32 Comics</a></figcaption>
</figure> 

You can see my source code [here](https://github.com/ScrewjankGames/ScrewjankEngine/blob/master/Engine/Source/Runtime/core/include/core/MemorySystem.hpp) and [here](https://github.com/ScrewjankGames/ScrewjankEngine/tree/master/Engine/Source/Runtime/system). If you're looking for a library that can give you all this and more (like being STL compatible, and thoroughly tested), check out [foonathan's memory allocators](https://github.com/foonathan/memory).

# The Allocator Interface
As with many things in life, I had no idea what I was doing. So what better place to start than looking at how other people went about writing memory allocators, and swiping their good ideas. The fine folks at [Bitsquid](http://bitsquid.blogspot.com/2010/09/custom-memory-allocation-in-c.html), [Gamedev.net](https://www.gamedev.net/tutorials/programming/general-and-gameplay-programming/c-custom-memory-allocation-r3010/), [Sony Computer Entertainment London](https://www.gdcvault.com/play/1023005/Building-a-Low-Fragmentation-Memory), and [Naughty Dog (See 36:27)](https://www.gdcvault.com/play/1022186/Parallelizing-the-Naughty-Dog-Engine) provide some great insights.

Distilling the above down, I ended up with the following interface for my Allocators:

```cpp
class Allocator
{
public:
  virtual void* Allocate(size_t size, size_t alignment) = 0;

  virtual void Free(void* memory) = 0;

  template<class T>
  void* AllocateType();
};
```

The controlling ideas here are that users interact with allocators through instances, as opposed to the STL's preference ([but not requirement!](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2554.pdf)) for stateless allocator types. Using instances for allocator access provides more flexibility when chaining or changing allocators out at runtime. In addition, allocator instances can be supplied to custom containers without changing the container's type.

In addition to this interface, I laid out some golden rules that each allocator must follow:
<figure style="width: 700px" class="align-center">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/golden-rules.jpg' | absolute_url }}" alt="1. Allocators only reserve and release raw memory. 2. Every allocation and header is properly aligned. 3. All allocators track allocation data. 4. All allocators ASSERT(false) if a memory leak is detected">
</figure> 

# The Unmanaged Allocator
This allocator is about as simple as possible. It's a thin wrapper over `malloc()` that keeps an allocation count for leak detection.

```cpp
class UnmanagedAllocator : public Allocator
{
public:
  ~UnmanagedAllocator()
  {
    SJ_ASSERT(m_ActiveAllocationCount == 0) // Memory leak detection
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

In case you were wondering `malloc()` allocates virtual memory. You could ask it for more memory than there is ram in your machine.
{: .notice--success}

# Allocator Tools

With the first allocator done, we can also start building tools that provide access to the system.

### The Memory Sytem
Here's a simple structure that provides access to the unmanaged allocator across the engine:
```cpp
class MemorySystem
{
public:
  static Allocator* GetUnmanagedAllocator()
  {
    static UnmanagedAllocator s_Allocator;
    return &s_Allocator;
  }
};
```

Now any caller can include `MemorySystem.hpp` and get access to a memory allocator. Later, we can extend this class to allow the engine to initialize more more advanced allocators, and reserve a configurable amount of memory at startup. For now though this Unmanaged Allocator will be valid for the duration of the program's life, and will be destroyed automatically after your program terminates.

### Overriding new and delete
Now that we have an allocator and a means of accessing it, the next step in getting a handle on your system's memory allocations is to override the global `new` and `delete` operators.

You can do so by adding the following in **any** .cpp file. At link time, the linker will find these definitions and use them to replace the defaults everywhere in your program.
```cpp
[[nodiscard]] void* operator new(size_t num_bytes) noexcept(false)
{
    return MemorySystem::GetUnmanagedAllocator()->Allocate(num_bytes);
}

void operator delete(void* memory) noexcept
{
    MemorySystem::GetUnmanagedAllocator()->Free(memory);
}
```

This means calls to `operator new()` and `operator delete()` will now be forwarded to our allocator, with the added bonus of imploding your program if the allocator detects a memory leak at the end of execution.

# Memory Alignment
For the remaining allocators, it's critical to understand the importance of memory alignment. Jonathan Rentzsch from IBM has an amazing rundown of [everything you need to know](https://developer.ibm.com/technologies/systems/articles/pa-dalign/). I highly suggest you give it a glance.

The short of it is that your CPU reads memory in chunks. If you don't respect that behavior, depending on your hardware:
<figure style="width: 700px" class="align-center">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/alignment-threats.jpg' | absolute_url }}" alt="1. Your game will run slower. 2. Your game will crash. 3. Your game will crash your operating system">
</figure> 

By aligning your allocations, all your data can be read with the minimum number of memory accesses. Unaligned data can straddle two memory accesses, and require the processor to do some extra work to stitch your data together. It's always slower than getting the alignment right in the first place, and some processors just won't do it.

### Calculating alignment
To get the alignment requirement for a type in C++ you can use the `alignof()` operator.

Here are the rules for what that operator returns:

> Every data object has an alignment-requirement. The alignment-requirement for all data except structures, unions, and arrays is either the size of the object or the current packing size [...]. For structures, unions, and arrays, the alignment-requirement is the largest alignment-requirement of its members. Every object is allocated an offset so that:
>
> <p class="text-center">offset % alignment-requirement == 0</p>
> <footer><strong><a href="https://docs.microsoft.com/en-us/cpp/c-language/padding-and-alignment-of-structure-members?view=vs-2019">Microsoft Docs</a></strong> &mdash; Padding and Alignment of Structure Members</footer>

Given the alignment requirement of the type and the memory address, you can calculate how far that address is from being aligned like so:
```cpp
uintptr_t GetAlignmentOffset(size_t align_of, const void* const ptr)
{
    // Note: m % n === m & (n-1) if n is a power of two
    // Alignment requirements are always a power of two
    return (uintptr_t)(ptr) & (align_of - 1);
}
```

For more information on how structures are padded to enforce alignment requirements, [see here](https://docs.microsoft.com/en-us/cpp/cpp/alignment-cpp-declarations?view=vs-2019).

To get the next aligned address for a type given some starting memory address, we can calculate an adjustment value like so:
```cpp
uintptr_t GetAlignmentAdjustment(size_t align_of, const void* const ptr)
{
    auto offset = GetAlignmentOffset(align_of, ptr);
    if (offset == 0) {
        return 0;
    }

    auto adjustment = align_of - offset;
    return adjustment;
}
```

Adding that adjustment value to your memory address yields the next aligned address.

# Linear Allocators

Linear allocators are extremely simple, performant, and memory efficient. They operate by maintaining a pointer to the start of their buffer, and a pointer to the first free memory address. Allocations bump the free space pointer forward, and individual Frees are disallowed. Instead, to reclaim member the free space pointer is simply moved back to the start of the buffer.

### Allocations
Allocations bump the free pointer forward by exactly `allocation_size + alignment_padding` bytes. The allocator returns a pointer the first byte after padding.

<figure style="width: 700px" class="align-center">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/linear-allocator-allocate.jpg' | absolute_url }}" alt="">
</figure> 


### Frees
To reclaim memory from a linear allocator you can call `Reset()`. Reset moves the free pointer back to the start of the buffer, allowing the old data to be overwritten by new allocations. It doesn't actually release the memory back to the operating system until the allocator is destroyed.

Linear Allocators cannot free individual allocations (and in the Screwjank Engine they'll `assert(false)` if you try).
{: .text-center .notice--danger}

### Implementation
**[LinearAllocator.hpp](https://github.com/ScrewjankGames/ScrewjankEngine/blob/master/Engine/Source/Runtime/system/include/system/allocators/LinearAllocator.hpp)**

**[LinearAllocator.cpp](https://github.com/ScrewjankGames/ScrewjankEngine/blob/master/Engine/Source/Runtime/system/src/allocators/LinearAllocator.cpp)**

# Stack Allocators
Like linear allocators, stack allocators trade flexibility for speed &mdash; albeit in a less extreme way. Stack allocators treat allocations and deallocations as push and pop operations, which implies that allocations **must** be released in a LIFO (last-in first-out) order. 

Internally, stack allocators maintain:
- A pointer to the start of their buffer
- A pointer to the first free byte in their buffer
- A pointer to the most recent allocation header

### Allocations
For every allocation stack allocators move an offset pointer into their buffer forward by the number of bytes requested, plus some extra amount to account for the allocation header and padding.

The allocation headers contain the metadata necessary to unwind the stack:
```cpp
struct StackAllocatorHeader
{
  /** Pointer to the previous allocation */
  StackAllocatorHeader* PreviousHeader;

  /** Number of padding bytes inserted before the header for alignment */
  size_t HeaderOffset;
}
```

When allocating with header we need to ensure:
1. The header and user payload are right next to each other.
2. Both data segments reside at an address that satisfies their alignment requirements.

We can find an address that satisfies these conditions by:
1. Assuming the earliest possible payload address is: <br/> 
  `m_Offset + sizeof(StackAllocationHeader)`

1. Determining the strictest alignment requirement between the header and the user data: <br/>
  `max(requested_alignment, alignof(StackAllocatorHeader))`

1. Finding how many bytes we need to adjust the payload address from step 1 by to satisfy the alignment requirement from step 2: <br/>
  `GetAlignmentAdjustment(strict_alignment, first_payload_address)`

This means the allocation header can be placed at `uintptr_t(m_Offset) + adjustment`, with the user data at`uintptr_t(m_Offset) + adjustment + sizeof(StackAllocatorHeader)`.

Now that we know where everything needs to go &mdash; assuming there's enough space in the buffer &mdash; we can:
1. Place the header in the buffer and mark it as the current allocation header.
2. Update the stack allocator's offset to point to the first byte after the user's data
3. Return a pointer to the user's block

<figure style="width: 700px" class="align-center">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/stack-allocator-allocate.jpg' | absolute_url }}" alt="">
</figure> 

### Frees
Freeing data from the stack allocator is also fairly simple. Since this most recent allocation header is tracked by the allocator the free operation is simply:

1. Roll back `m_Offset` to the fist free byte after the last allocation: <br/>
  This is the address of the current header minus the offset that header has stored.

1. Update the allocator's current header to: </br>
    `current_header->PreviousHeader`

If the user instead called `Free()` with a memory address instead of pop, you simply need to verify that address is the top block.

<figure style="width: 700px" class="align-center">
  <img src="{{ '/images/posts/writing-your-own-memory-allocators/stack-allocator-free.jpg' | absolute_url }}" alt="">
</figure> 

# Pool Allocators
Pool allocators are general purpose allocators capable of allocating and freeing data in any order. 

Internally pool allocators maintain:
- A pointer to the start of their buffer
- A linked list of free blocks, constructed in the buffer
- A pointer the the head of that linked list

On construction pool allocators split their data buffer into fixed-sized blocks, with and construct a linked list with nodes placed at the beginning of every block. When a block is allocated, the header removed from the linked list and it's data is overwritten. The header is re-inserted and re-added on free.

In the Screwjank Engine pool allocators use a template parameter to define their blocks size. Doing so allows compile-time validation that the blocks are large enough to hold the free list nodes. 
```cpp
template <size_t t_BlockSize>
class PoolAllocator
{
  PoolAllocator(size_t num_blocks, Allocator* backing_allocator);
  ~PoolAllocator();

  ...
};
```

### Initialization
After construction every block in the pool allocator contains a `FreeBlock` structure:
```cpp
struct FreeBlock
{
  FreeBlock* Next;
};
```

The struct is a stylistic choice to make others less sad when they read your code. You can instead chain `void` pointers together to construct the list if you hate your fellow man.

[There's no such thing as a zero-cost abstraction](https://www.youtube.com/watch?v=rHIkrotSwcc). Since PoolAllocator is templated and FreeBlock is a private type defined inside it, every specialization of PoolAllocator actually results in another type `PoolAllocator<T>::FreeBlock` being compiled. The cost here is negligible but not non-existent. 
{: .notice--danger}


After acquiring a data buffer it can be split like into blocks like so:

```cpp
PoolAllocator<T>::PoolAllocator(size_t num_blocks, Allocator* backing_allocator)
{
  ...
  // Create first free block at start of buffer
  m_FreeListHead = new (m_BufferStart) FreeBlock {nullptr};
  FreeBlock* curr_block = m_FreeListHead;
  uintptr_t curr_block_address = (uintptr_t)m_BufferStart;

  // Build free list in the buffer
  for (size_t i = 1; i < num_blocks; i++) {
      // Calculate the block address of the next block
      curr_block_address += t_BlockSize;

      // Create new free-list block
      curr_block->Next = new ((void*)curr_block_address) FreeBlock();

      // Move the curr block forward
      curr_block = curr_block->Next;
  }
}
```


### Allocation 
Allocation is a O(1) operation that simply returns the pointer of the first free block, and removes it from the list:

```cpp
template <size_t t_BlockSize>
inline void* PoolAllocator<t_BlockSize>::Allocate(const size_t size, 
                                                  const size_t alignment)
{
  ...

  // Take the first available free block
  auto free_block = m_FreeListHead;

  ...

  // Remove the free block from the free list
  m_FreeListHead = m_FreeListHead->Next;

  ...

  return (void*)(free_block);
}
```

It's not necessary to account for padding with pool allocators, since (with the exception of over-aligned types) `alignof(T) <= sizeof(T)`. The memory address of each free block an alignment requirement of `alignof(t_BlockSize)`, so any data that fits in the block will have an equal or weaker alignment requirement. 
{: .notice--info}

### Frees
The free operation simply places a `FreeBlock` back into the buffer, and places the block at the head of the free list:

```cpp
template <size_t t_BlockSize>
inline void PoolAllocator<t_BlockSize>::Free(void* memory)
{
  ...

  // Place a free-list node into the block
  auto new_block = new (block_address) FreeBlock {m_FreeListHead};

  // Push block onto head of free list
  m_FreeListHead = new_block;

  ...
}
```

# Free List Allocators
Like a block allocator with dynamically sized blocks

# Proxy Allocator
Proxy allocators are adapters for other allocators. They simply forward all of their operations to a backing allocator, and maintain their own tracking data. They're super useful for situations in which you want multiple systems to share an allocator instance, but keep separate tabs on each system's memory usage.

# Moving Forward
What extensions can or should be made?

- Better utilization of virtual memory
- Remove fixed-size buffer limitations on some allocators
- Improved data tracking
- Security

# References
I like to pretend this new engine's going to run on a Playstation one day. So, with that in mind, carefully managing when and how memory is acquired is a must. As studios like [Naughty Dog (see 36:27)](https://www.gdcvault.com/play/1022186/Parallelizing-the-Naughty-Dog-Engine) and [SCE London](https://www.gdcvault.com/play/1023005/Building-a-Low-Fragmentation-Memory) can demonstrate: memory allocators are a critical piece of engine architecture. 
Here's some useful shit:

* https://www.gamedev.net/tutorials/programming/general-and-gameplay-programming/c-custom-memory-allocation-r3010/
* http://bitsquid.blogspot.com/2010/09/custom-memory-allocation-in-c.html