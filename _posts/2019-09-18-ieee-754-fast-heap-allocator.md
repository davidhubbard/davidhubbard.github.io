---
layout: post
---

# IEEE 754 Fast Heap Allocator

Usually when [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) is mentioned,
it's talking about numbers, especially fractions.

But there's an interesting way to organize a heap allocator that leads to
fast and cache-efficient memory allocation:

### Setup

1. Decide on the minimum size of an allocation - say 32 bytes
2. Divide the contiguous memory to be used into blocks starting with the
   largest power-of-2 size that will fit, and then compute the largest
   power-of-2 size in the remainder, thus creating at most 1 block in
   each power-of-2 size.

### Request to allocate memory

1. Convert the size to an IEEE 754 float. (Also do some sanity checks:
   no allocation can have a fractional size. If it's less than 1 byte, fail.
   If it's less than the minimum size, increase it to the minimum.)
2. Copy the exponent only. If the exponent shows it's larger than the largest
   power-of-2 size, stop.
3. Pop the first available block from the power-of-2 size, if one exists.
   Finish by handing this block to the requester.
4. At this point, the request can only be served by finding the next largest
   power-of-2 size. Instead of a for() loop, use bitmasks to find the next
   set bit. Then pop the first available block from the first available
   power-of-2 size, and possibly update the bitmasks if it was the only block
   in that power-of-2 size.
5. Since this block is larger than the requested size, divide it from one
   contiguous block into smaller blocks. The largest of the smaller blocks
   gets pushed back into their power-of-2 list, leading to smaller and smaller
   blocks until the right size is reached.
6. When the block at the right size is all that remains, hand this block to
   the requester.

### Request to deallocate memory.

This is the non-obvious improvement: instead of an O(n) search along a list
of free blocks to find adjacent free blocks and combine them into one
larger block, this accomplishes the same thing, but in O(1) memory
accesses:

1. Retrieve the size of the block in the request from a pointer hidden just
   before the block's address in memory.
2. Update that hidden pointer to a different format that indicates it is
   now a free block.
3. Access the memory just beyond the end of the block's addresses. Check if
   this will cause a page fault, and skip step 4 instead.
4. If the hidden pointer just beyond the end of the block is in the format
   indicating it is a free block also, get its size and decide if combining
   the two would make a power-of-2 size. If the combination is power-of-2 in
   size, pop the one "just beyond" from its respective power-of-2 list.
   Update the block in the request to reflect it is now the combination of
   the two and push it into a larger power-of-2 list of free blocks. Then
   go back to step 3 to try merging with any other large blocks beyond that.
5. At this point, merging with free blocks "just beyond" has ended. Each
   free block is also set up with a marker at the end of the block, on a
   fixed alignment, that points to the start of the block - allowing a check
   for a free block "just before" the request in memory.

   Since this marker could be confused with normal data stored in a block,
   only follow the marker after walking the power-of-2 list of free blocks
   it supposedly represents. This is faster than walking all free blocks
   because it only checks blocks of an exact known size.
6. If a valid free block sits just before the request in memory, use the
   same process as in step 4 to merge free blocks.
