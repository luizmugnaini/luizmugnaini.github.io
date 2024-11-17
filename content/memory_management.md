+++
title = "Memory management strategies"
date = 2024-01-24

[taxonomies]
tags = ["C", "C++", "projecs", "programming"]
+++

I'm building a game engine from scratch in C++ so I decided to document this journey. Since I want
all control over my code, I'm also writing my own alternative to the C++ STL. My goal is to deliver
the programming experience I wanted and libc wasn't able to deliver.

<!-- more -->

Over the last few years C got the fame of being a memory unsafe due to its lack of intrinsic
language constraints that ensure proper object lifetimes, like you have in new trendy languages. You
might ask why bother memory strategies in C if I can simply use my language Z that has a
borrow-checker or a garbage collector: well, language Z is not suited for all use cases - surely not
for me. The borrow-checker is a pain to deal with and most memory strategies in that land, like ECS,
are based in making the compiler shut up by hiding the lifetimes of objects inside of longer
living objects. Do I even need to talk about the garbage collector approach?

# Some points of clarification

- I want to do *everything* from the ground up.
- Performance is our queen.
- Temporary memory allocation should be as cheap as possible.
- Dynamic memory allocation should be avoided.

# First questions to ask when building your own memory system

When writing any API, one should question itself the scope and the audience of the API. In the case
of a memory management system, the main concern is centred in memory safety. How much safety guard-rails
should we build and how much hand-holding does our end-user need?

Guard-rails obviously comes with their own performance costs. For instance, if we wanted a very
robust system that disallows for any use-after-free, we wouldn't be returning direct pointers to the
memory to our users. To deal with that, we could use handles and manage their coherence internally
(check [this post](https://floooh.github.io/2018/06/17/handles-vs-pointers.html), by Andre
Weissflog, for more information). This would require every allocation to have a unique ID, every
read request to be checked if ID's match, etc.

In my personal case, as a solo developer of this engine, I don't really need all of this
hand-holding. For the sake of performance, I decided to go for raw pointers and their inherent
unsafe behaviour.

# Stack Allocator

A stack allocator is nothing more than a contiguous memory block which we divide in order to offer
memory space to consumers. In order to avoid memory fragmentation, we only allow the last allocated
block to be freed.

## Headers: storing relevant information

Each memory block allocated by our stack allocator will be accompanied by a header that will carry
some basic information about the memory block it's associated:

```cpp
struct StackAllocHeader {
    size_t padding;
    size_t capacity;
    size_t previous_offset;
};
```

Let me explain what each one of these fields mean:

- `padding`: The offset relative to the *end* of the previously allocated memory block, to the start
  of the current memory block. You may think this is unnecessary, but each allocation has to take
  care of memory alignment! Relax, we'll cover all that in due time.
- `capacity`: The total capacity, in bytes, of the current memory block.
- `previous_offset`: The offset relative to the *start* of the stack allocator memory block. This
  will help us to traverse the stack backwards.

<!--
TODO: Of course you could write a more light weight header, but that is up
      to you (reader).
-->

> Let me flex my horrible ASCII art skills right in front of thy eyes... hold
> on to your seat belts.

In other words, what we have can be summarized by the following diagram:

```md
         previous offset              alignment              |----- capacity -----|
                ^                     ^       ^              ^                    ^
                |                     |       |              |                    |
|previous header|previous memory block|+++++++|current header|current memory block|
                                      ^                      ^
                                      |------- padding ------|
```

> If you want to keep track of the memory by type of usage (string, vector, etc.), you can create a
> memory tag system and add a tag to each header.

## Alignment: rules for memory reads

The CPU memory reads cannot be done willy nilly at any given address. Modern architectures are
optimised to read contiguous memory with a certain alignment - which makes stepping through memory a
regular task (as opposed to jumping around randomly). This alignment is always a power of two and
depends on the size of the memory units (e.g. a struct) we want to read. In C++ you can query the memory alignment
for a given type `T` using `alignof(T)`.

For instance, if we have an array of floats (which has an alignment of 4 bytes), the address of the
`n`th element of the array in memory should be of the form `4 n + c` where `c` is the address of the
first element of the array.

For structs, the compiler may need to add paddings in order to satisfy alignment conditions. A lost
art in programming is the arrangement of struct members for optimal alignment. Let's see this in
practice.

Suppose I have a struct `Foo` has the following memory layout:

```cpp
// Non-optimal packing of members.
struct Foo {
    uint8_t* memory;             // 8 bytes.
    uint32_t allocation_count;   // 4 bytes.
    // ---------------------------> Invisible padding of 4 bytes.
    double   some_metric;        // 8 bytes.
    float    some_other_metric;  // 4 bytes.
    // ---------------------------> Yet another padding of 4 bytes.
};
```

In comments I've put the real memory layout from the point of view of the compiler and, most
importantly, from the CPU. Notice, the compiler has to make sure that the alignment of each member
of `Foo` is valid.

For the sake of contradiction, suppose that the compiler didn't put those padding bytes. If we
wanted to access the member `Foo::some_metric` (which has an alignment of 8 bytes), we would start
to read `Foo` in the same address as the first member, then we would advance by our alignment of 8
bytes at a time until we supposedly reach `Foo::some_metric`. Surprise! We won't be able to reach
our destination. For this exact reason, the compiler has to arrange memory for the worst case
scenario - the largest alignment in the collection of members. In our case, the compiler sees that
the largest alignment is 8 bytes and makes sure that every element has its address in the
form `8n + c` (where `c` is the address of `Foo`).

One thing I didn't explain is why the compiler has to put those 4 bytes of padding after
`Foo::some_other_metric`. If for some reason you have a contiguous array of `Foo` instances, in
order to traverse through the array we would use the alignment of `Foo` (8 bytes) - and if it wasn't
for those last 4 bytes, we wouldn't reach the next `Foo` in the line.

Fixing our bad memory usage is simple, we just rearrange the members taking into account their size:

```cpp
// Optimal alignment of members.
struct Foo {
    uint8_t* memory;             // 8 bytes.
    double   some_metric;        // 8 bytes.
    uint32_t allocation_count;   // 4 bytes.
    float    some_other_metric;  // 4 bytes.
};
```

Analogously, every time we allocate memory in our custom allocators we'll have to make sure to
account for the necessary alignment.

## Stack allocator

On to the stack allocator proper! The basic layout of the allocator looks like
the following diagram:

```md
       previous
        offset
     for header 2
           ^                            current
           |       alignment            offset
           |        ^    ^                 ^
           |        |    |                 |
  |header 1|memory 1|++++|header 2|memory 2|++free space++|
  ^                 ^             ^                       ^
  |                 |---padding---|                       |
memory                            |                      end
  |                            previous                   |
  |                             offset                    |
  |                                                       |
  |--------------------- capacity ------------------------|
```

The stack allocator consists of the following:

- A pointer `memory` to the start of the memory block it's responsible to
  manage.
- A total `capacity` in bytes that its memory can hold.
- An `offset` relative to `memory` to the allocator's free space.
- The `previous_offset` relative to `memory` to the start of the previously
  allocated memory block.

Each block provided by the stack allocator consists of an *alignment* offset
with respect to the end of the previous block, a *header*, and the available
block of *memory* requested by the user.

The *padding* block of memory is preceded by a padding that comprises both the
alignment needed for the memory block and its corresponding header. This header
will store the size of the  padding and the offset to the start the previous
memory block with respect to itself.

The allocator can be either the owner or merely the manager of the memory
pointed by `memory`. If the allocator is the owner, the user should always call
`stack_destroy` in order to free resources. If a block of memory is to be
provided to the allocator for only management, the user may instantiate the
allocator via `stack_init` with a valid pointer to the available memory.

The basic struct will look like the following

```cpp
struct StackAlloc {
    uint8_t* memory = nullptr;
    size_t capacity = 0;
    size_t offset = 0;
    size_t previous_offset = 0;

    StackAlloc(size_t const capacity);
    ~StackAlloc();

    /**
     * @brief Allocates a block of memory of a given size and alignment.
     *
     * @param size      Size, in bytes, of the new memory block.
     * @param alignment The needed alignment of the new memory block. This
     *                  number should always be a power of two.
     *
     * @return If the allocation was successful, a pointer to the new block of
     *         memory is returned. If the stack ran out of memory, a null
     *         pointer will be returned. You should always check the returned
     *         pointer by this function.
     */
    void* alloc(size_t const size, size_t const alignment);

    /**
     * @brief Reallocate a block of memory
     *
     * This method is used to resize blocks of memory. The process of resizing
     * allocates a *new* block of memory and copies the data of the original
     * block to the new one, then returns a pointer to the new block.
     *
     * Resizing blocks of memory will inherently cause memory fragmentation in
     * a stack allocator due to its stack-like nature, so you should avoid
     * resizing if you can.
     *
     * @note The following code-paths may occur according to the given parameters:
     *         - If `mem` is a null pointer, we'll perform an allocation.
     *         - If `new_size` is zero, we'll call `StackAlloc::free` on the
     *           memory block. Notice that this will free all blocks coming after `mem`.
     *         - If `mem` and `new_size` are zero, then a null pointer is returned.
     *
     * @param mem       Pointer to the start of the memory block to be resized.
     * @param new_size  New size, in bytes, of the resized block of memory.
     * @param alignment Memory alignment required for the datatype that shall
     *                  be stored in the memory block. This argument is optional and has
     *                  a default value of `psh::utils::k_default_alignment`.
     *
     *  @return Pointer to the new block of memory. Can return a null pointer
     * if the allocation wasn't successful.
     */
    void* realloc(
        void* const mem, size_t const new_size, size_t const alignment);

    /**
     * Frees the last memory block allocated by the given stack.
     *
     * This function won't panic if the stack is empty, it will simply return
     * false.
     *
     * @return Returns the status of the operation: true if it was successful,
     * false otherwise.
     */
    bool pop();

    /**
     * Free all memory blocks up until the specified block pointed at by `ptr`.
     *
     * @param ptr Pointer to the memory block that should be freed (all blocks above
     *            `ptr` will also be freed). If this pointer is null, outside of the
     *            stack allocator buffer, or already free, the program return false and
     *            won't panic.
     *
     * @return Returns the status of the operation: true if it was successful,
     * false otherwise.
     */
    bool clear_at(void* const ptr);

    /**
     * @brief Frees all blocks of memory.
     *
     * After freeing, the allocator's memory will still be available for later
     * allocations, since this method simply resets the allocator's offsets.
     *
     * @return Returns the status of the operation: true if it was successful,
     * false otherwise.
     */
    bool clear_all();
};
```

> It is to be noted that the pointers returned by the methods associated to
> `StackAlloc` are all raw pointers. This means that if you get a memory block
> via `StackAlloc::alloc` and later free it via a `StackAlloc::free_at`, you'll
> end up with a dangling pointer and use-after-free problems may arise if you
> later read from this pointer. This goes to say that the user should know how
> to correctly handle their memory reads and writes.


### Suggestion: use templates in your favor

Create additional methods that compute the alignment automatically. You could
do something like

```cpp
template <typename T>
T* alloc(size_t const length) {
    return reinterpret_cast<T*>(alloc(length * sizeof(T), alignof(T)));
}
```

as a method of `StackAlloc`. This will give your user a better experience as
they'll only need to pass the number of entities of type `T` that the requested
block should be able to carry.

You can also add a default value for the alignment in case the user doesn't
want to provide one. One option could be `2 * sizeof(void*)`.

# The Memory System

In this engine we'll have a central memory system that will carry out every
allocation and freeing of memory

# Some failures of the system

<!-- TODO: Why you might not want to use this system at all -->

# Further reading material for the nerds

Memory management systems are actually quite fun and interesting. Although they
may seem scary at first, one can build a simple memory system that outperforms
most uses of smart-pointers, garbage-collected languages. It goes without
saying that having a good memory system frees you from the paranoia that
`malloc` will inherently generate.

- [Memory Management Reference](https://www.memorymanagement.org/).
- [Untangling Lifetimes: The Arena Allocator](https://www.rfleury.com/p/untangling-lifetimes-the-arena-allocator),
  by Ryan Fleury.
- Do yourself a favor and read gingerBill's *amazing*
  [Memory Allocation Strategies series](https://www.gingerbill.org/series/memory-allocation-strategies/).
- [Handles are better than pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html), by
  Andre Weissflog, author of the great [sokol](https://github.com/floooh/sokol)
  header-only C libraries.
- Aaron MacDougall's GDC 2016 talk on [Building a Low-Fragmentation Memory System for 64-bit Games](https://gdcvault.com/play/1023309/Building-a-Low-Fragmentation-Memory).
