---
title: "A dumb Vulkan Memory Allocator in plain C"
date: 2021-08-31T12:50:50+02:00
draft: true
---

By reading the github repository of the famous library Vulkan Memory Allocator written by the AMD team, you will see a kinda huge list of games and apps which are actually using it. From "Detroit: Become Human" to "Skia", they are many who trust it. And they're right in a way. It does the job and it does it right. However, when you're learning a subject, you may want to investigate it deeply. You do not want to rely on whatever library is giving you some level of comfort (if it wasn't the case, I would have stayed with OpenGL).

So in this post (the first one of my newly created portfolio), I will explain **how** did I write my *dumb* memory allocator and **why** it's a dumb allocator. I do not ask you to blindly trust me, but I've gained some level of confidence as I heavily use it throughout my projects.

## Basics concepts of memory allocation

First of all, what's a memory allocator? Every resource you use in your program is actually a structured representation of bytes present in your physical memory. This piece of memory is characterized by a size and an address (*how many bytes would you eat?* and *where should I find those bytes?*). When you're writing basic CPU code, you're allocating with *malloc* (or a similar utility) which returns the actual address of the newly created piece of memory. Working with OpenGL, you're creating a resource with some specific size and everything is done magically by your driver. Don't you dare ask where did it put your data, you should not work directly with its address anyway.

Creating a vulkan resource (buffers and images, for example), doesn't mean allocating the required space. It's a separated action you *have to* execute to make the resource usable (creating an image view, binding the buffer as a vertex buffer, etc). That's where your allocator comes in. As you created your image with precise dimensions, your buffer with a precise size, it's easy to guess how your reserved piece of memory should be...

## Guessing the required size

There is no guess! All you need is your physical device with which you can invoke:

```c
VkMemoryRequirements memory_requirements;
vkGetImageMemoryRequirements(device, image, memory_requirements);
// or
vkGetBufferMemoryRequirements(device, image, memory_requirements);
```

Easy easy! `memory_requirements` now contains everything you need to allocate: a size, a memory type and an alignement.

## Allocating for your resource

A computer able to run vulkan code has different types of memory. I won't detail them exhaustively, but it's intuitive to think that depending on the usage of your resource, its place will be different. A mappable buffer and a color attachment texture aren't both requiring the same memory type, for example. In `dumb_alloc` code, I call them *memory region*.

I WANT TO ALLOCATE NOW, WHERE IS THE vkMalloc DUDE??? If you want to rush it, let's just acquire a VkDeviceMemory every time you need to bind memory to your vulkan object.

```c
VkMemoryAllocateInfo allocate_info = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .memoryTypeIndex = your_memory_type,
    .allocationSize = your_size,
};
vkAllocateMemory(device, &allocate_info, NULL, &device_memory)
```

Easy, fast, end of my post. You can leave now. The others who stayed, shame on this guy who wanted to speed run its allocation. It's a horrible practice. The number of memory allocations you can make is, in fact, very variable and limited. It can go from 4.096 to 4.294.970.000 and and and, just read this https://developer.nvidia.com/vulkan-memory-management.

So let's make one big memory allocation for each memory type requested by our engine and divide it for each resource according to their required size. It should be easy, right? Right? Yesn't. You have to write a specialized data structure that fits your needs. With `dumb_alloc`, a memory region is divided into multiple contiguous blocks. Each block is characterized by an enabled boolean, a size and an offset. The `enabled` flag is straightforward, is this block actually used by a resource? `offset` defines the place in a specific VkDeviceMemory where the resource memory begins, while `size` defines the length of the block. 

You want to allocate? In this data structure, you can either:

1. Append a new block at the end.
2. Insert a new block between two blocks.

You do (1) when you didn't find enough free space by iterating through blocks with a `enabled=false`. You do (2) when you find a non-used block with a sufficient size. In this case, you divide the block into two parts: a used block and a non-used block. Before an `alloc` you `join` and `check` your blocks.

* join: contiguous disabled block are merged
* check: all blocks have to be contiguous
* alloc: find a place and return offset

With this *dumb* data structure which is nothing but a dynamic array you constantly resize and insert elements into, your memory is now structured adequately, and it's trivial to find where you can place your resources. The `dumb_alloc` API allows you to do it in one line:

```c
VkMemoryRequirements memory_requirements;
vkGetImageMemoryRequirements(device, image, &memory_requirements);

dumb_allocation_t image_allocation = dumb_allocate(device, &memory_requirements, DUMB_MEMORY_USAGE_HOST_ONLY);
vkBindImageMemory(device, image, image_allocation._memory, image_allocation._offset);
```

## Alignment of offset

All offsets used to bind memory should be a multiple of `VkMemoryRequirements::alignment`. It's the specifications. So our little approach has to be a little modified to be fully conform. On my 1050ti, image offsets have to be a multiple of 1024 and buffer offsets have to be a multiple of 256. The *dumb*est way to work around this vital part is to make a different `vkDeviceMemory` for each different memory alignment (SPOILER: we won't do that). Thankfully, there is `VkPhysicalDeviceLimits::bufferImageGranularity` which is, if you let me paraphrase it, the lowest common multiple between buffer and image alignments. So let's just allocate a block with a modified size. A size multiple of bufferImageGranularity so that each offset is necessarily conform to the memory requirement for both image and buffer.

## Conclusion

`dumb_alloc` is my personal vulkan memory allocator I've written for my vulkan projects. Instead of using an already made library, I chose to make it myself to understand a bit more the memory stuff around this hard API. In this blog post, we saw the underlying data structure of such an allocator and some motivations behind the implementation.
