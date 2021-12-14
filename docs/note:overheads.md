I'm already going to take for granted:
  * The Generational Hypothesis: most objects will die young, and
  * The Functional Hypothesis: most objects will not be modified after initialization

My additional hypothesis is that most objects will be:
  * small
  * unaligned
  * not pinned
  * have no finalizer

My hypothesis is important for a few reasons:
  * small: so it will fit into a reasonably small block of memory
  * unaligned: so alignment within a reasonably small block of memory will not cause excessive internal fragmentation (and therefore an inefficient early collection)
  * not pinned: so that all survivors can be moved out of a memory block
  * have no finalizer: so that all dead objects in a block can be collected en masse

## Summary (in progress)

Parameters:
  * `BLOCK_SIZE` (also equal to block alignment)
  * `sizeof(gcobj*)`

Small objects are always allocated in a block.
They are aligned on a 16B boundary (on 64-bit systems).
They have a one-word header, which is never zero, that describes the layout and a handful of flags.

  * `NPTRS_DESCBITS = log₂(BLOCK_SIZE/sizeof(gcobj*))`
  * `NBYTES_DESCBITS = log₂(BLOCK_SIZE)`
  * `NELEMS_DESCBITS = log₂(BLOCK_SIZE)`
  * `ALIGN_DESCBITS = log₂(log₂(BLOCK_SIZE/2) - log₂(sizeof(gcobj*)))`
  * `LAYOUT_DESCBITS = NPTRS_DESCBITS + NBYTES_DESCBITS + NELEMS_DESCBITS + ALIGN_DESCBITS`


## Allocate to a Block

This optimization allows two things:
  * allocation is a simple (checked) pointer increment
  * header information common to all objects in the block can be coalesced into a block header

## The Costs of Alignment

Let's say that the allocator operates on 4kiB blocks (the size of a page on amd64).
Objects will naturally be 8B-aligned (so that all header info and pointers to gc objects are aligned.
Since this architecture may support AVX-512, which suggests 64B alignment for optimal loads/stores of 512b SIMD registers, it's not unreasonable to assume 64B alignment requests on managed memory.
The worst-case for memory fragmentation would be allocation of such objects would be allocating single 64B-wide objects at 64B boundaries.
Assuming a header size of 8B, about 42% of the bytes in a block could be wasted (not even used for overhead).
This is, of course, a pathological case, but for completeness:

  * 16B alignment: 8B/32B (25%) maximum internal fragmentation
  * 32B alignment: 24B/64B (37%) maximum internal fragmentation
  * 64B alignment: 54B/128B(42%) maximum internal fragmentation
  * 128B alignment: 120B/256(46%) maximum internal fragmentation

If, however, only 10% of the memory in a block is aligned (and all unaligned fields take up a multiple of 8B), then internal fragmentation will drop by an order of magnitude, and will asymptotically approach 50% as the alignment restriction increases (assuming the object is at least as large as its alignment).
Nevertheless, maximum alignment when allocating into a block will still have to be half the block size (assuming we only support power-of-two alignments.

So it looks like the costs of memory alignment are not too much.
Efficiency (reducing internal fragmentation) could be reduced by separating alignment groups into blocks, but I'm not sure the complexity (of development, and also computing the correct block at runtime) is worth it.

I have two strategies:
  * Blocks with mixed alignment: object alignment stored in object header; maximum alignment up to a tunable parameter up to have the block size parameter, default 64B
  * Blocks with a single alignment: object alignment in block header; user asks allocator for an aligned block only when necessary, and explicitly uses that block to allocate

The former looks a lot better, because the tracer can move all small objects into a single location, and therefore would have only a single tracing queue for small objects.

## The Cost of a New Allocator

I'm thinking small blocks, large blocks, and of course non-blocked objects

If I spawn 10k threads, and a block is 4kiB, that's only ~40MiB of memory reserved .
If a block is 1MiB, that's ~10GiB of memory reserved.
While a modern address space is much larger than 10GiB, I wonder how an OS would actually report that memory usage.
