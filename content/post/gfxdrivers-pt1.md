---
title: "Diving into graphics drivers, Part 1"
date: 2023-07-30T16:02:51+10:00
draft: false
---

**Note:** This is merely a diary of my journey into graphics drivers. I may get some things wrong here ([email me if this is the case](mailto:shamoslover69@gmail.com)), but it's absolutely worth noting here that I am a complete and utter beginner to graphics driver development.

## Motive

Why am I getting my hands dirty with graphics drivers? Throughout my entire software development life (mostly as a hobbyist contributing to open-source), I have always tried to find "the next thing" to challenge myself with. At some point, that "next thing" was learning Vulkan and graphics programming, which I can now say I have succeeded in doing (though of course, I am always learning). Learning Vulkan without a lot of prior exposure to graphics programming and without much guidance was definitely one of the hardest things I've done in terms of software development, but it broke down many mental barriers in terms of levels of difficulty - essentially, nothing in software (or even hardware) is "too hard" for me now: I know I can learn and understand anything if given enough time. I also have a mindset in software to never leave any stone unturned, so while the Vulkan API is nice and all, I'd like to see how my `vkCmdDraw` ends up as voltage in the PCIe line.

So in short, the motivation for this is simply, *to learn*.

## Where do I start?!

Perhaps the biggest challenge when learning something new (and something so vast) is the ubiquitous beginner question: "Where do I even start?". My answer to this is to **just start**. Start anywhere. And start I did - at the Linux kernel.

### Enter DRM

When I looked into graphics drivers, I saw one common underpinning to all of them: `libdrm`. What is this `libdrm` and what does it do? `libdrm` is actually the interface to the DRM drivers in the Linux kernel. See, there's a very clear boundary to where graphics drivers operate. Much of the graphics driver actually operates in user-space, but at some point the driver will definitely need to send packets over PCIe. Now I'm sure there are ways to do this in user-space (vfio?), but really this is the job of the DRM driver running in kernel-space. More obvious is that this task is *heavily* vendor-specific. The layout of the data sent and what kind of data to send is all very specific to the GPU itself. We can confirm this by looking at [`linux/drivers/gpu/drm`](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm). We can see the main desktop vendors here like `amd` and `nouveau`, but also some reverse-engineered ones like `panfrost` for Arm Mali, and in the future I'm sure the `agx` DRM will be upstreamed for Asahi Linux! For the rest of my exploration, I decided to focus exclusively on AMD drivers, for no particular reason.

I won't get into the details of every aspect of the DRM (mostly because I don't know those details...), but I'll dissect a very small slice of it to get a very rough idea of how it works. Let's look at how, for example, a command to copy a GPU buffer works.

Right off the bat, I'm able to find a function named `amdgpu_copy_buffer` in `amdgpu_ttm.c`. Through curiosity, one quick search leads me [here](https://docs.kernel.org/gpu/drm-mm.html), and we find that `ttm` stands for **Translation Table Manager**. That page sums it up in a single sentence nicely, but essentially, TTM will give us a **buffer object** which represents our resource(s), and it will handle all the nitty gritty details like CPU mapping and sending the memory around. Okay, excellent. Now let's look at the implementation of `amdgpu_copy_buffer`:

```c
int amdgpu_copy_buffer(struct amdgpu_ring *ring, uint64_t src_offset,
		       uint64_t dst_offset, uint32_t byte_count,
		       struct dma_resv *resv,
		       struct dma_fence **fence, bool direct_submit,
		       bool vm_needs_flush, bool tmz)
{
	struct amdgpu_device *adev = ring->adev;
	unsigned int num_loops, num_dw;
	struct amdgpu_job *job;
	uint32_t max_bytes;
	unsigned int i;
	int r;

	if (!direct_submit && !ring->sched.ready) {
		DRM_ERROR("Trying to move memory with ring turned off.\n");
		return -EINVAL;
	}

	max_bytes = adev->mman.buffer_funcs->copy_max_bytes;
	num_loops = DIV_ROUND_UP(byte_count, max_bytes);
	num_dw = ALIGN(num_loops * adev->mman.buffer_funcs->copy_num_dw, 8);
	r = amdgpu_ttm_prepare_job(adev, direct_submit, num_dw,
				   resv, vm_needs_flush, &job, false);
	if (r)
		return r;

	for (i = 0; i < num_loops; i++) {
		uint32_t cur_size_in_bytes = min(byte_count, max_bytes);

		amdgpu_emit_copy_buffer(adev, &job->ibs[0], src_offset,
					dst_offset, cur_size_in_bytes, tmz);

		src_offset += cur_size_in_bytes;
		dst_offset += cur_size_in_bytes;
		byte_count -= cur_size_in_bytes;
	}

	amdgpu_ring_pad_ib(ring, &job->ibs[0]);
	WARN_ON(job->ibs[0].length_dw > num_dw);
	if (direct_submit)
		r = amdgpu_job_submit_direct(job, ring, fence);
	else
		*fence = amdgpu_job_submit(job);
	if (r)
		goto error_free;

	return r;

error_free:
	amdgpu_job_free(job);
	DRM_ERROR("Error scheduling IBs (%d)\n", r);
	return r;
}
```

Hmm, lots of stuff going on here. Let's take it step-by-step. I can pick out the parameters that immediately make sense: `src_offset`, `dst_offset`, and `byte_count` are painfully obvious and I assume they pretty much directly map to the same parameters higher up (e.g., [`VkBufferCopy`](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkBufferCopy.html)). Now `amdgpu_ring`, I can see that `amdgpu_device` is taken from `ring->adev`, so what is `amdgpu_ring` exactly? Some more poking around into `amdgpu_ring.h` and fingers crossed there's a comment...

```c
/* provided by hw blocks that expose a ring buffer for commands */
struct amdgpu_ring_funcs {
	/* ... */
```

Ahh. Well, there it is; a ring buffer for commands.

Now `dma_fence`, this is definitely an output parameter, and I'm going to take a wild guess and say it's a fence primitive for sync.

```c
/**
 * struct dma_fence - software synchronization primitive
	...
 */
struct dma_fence {
	/* ... */
```

Wow! Am I a genius or what? :)

Now important distinction to make here is that I don't think this is *directly* related to the GPU fences we all know and love (e.g., `VkFence`). This fence is actually agnostic to any kind of driver in the Linux kernel and seems to be a part of the DMA system. Now very well this could be a part of the implementation for those GPU fences ([and probably is?](https://www.kernel.org/doc/html/v5.9/driver-api/dma-buf.html#dma-fences) We'll find out later I'm sure)

Moving on from that, we see a `dma_resv` parameter. Let's investigate:

```c
/**
 * struct dma_resv - a reservation object manages fences for a buffer
 *
 * This is a container for dma_fence objects which needs to handle multiple use
 * cases.
 *
 * One use is to synchronize cross-driver access to a struct dma_buf, either for
 * dynamic buffer management or just to handle implicit synchronization between
 * different users of the buffer in userspace. See &dma_buf.resv for a more
 * in-depth discussion.
 *
 * The other major use is to manage access and locking within a driver in a
 * buffer based memory manager. struct ttm_buffer_object is the canonical
 * example here, since this is where reservation objects originated from. But
 * use in drivers is spreading and some drivers also manage struct
 * drm_gem_object with the same scheme.
 */
struct dma_resv {
	/* ... */
```

Interesting. Again, part of the DMA system and so driver-agnostic, but essentially this manages access to the buffer objects mentioned earlier. Makes sense! So if we need to access a buffer, we need to lock it, or "reserve" it, before doing so. Looking at the body of the `amdgpu_copy_buffer` function we can see it's used in one place only, `amdgpu_ttm_prepare_job` (defined right above `amdgpu_copy_buffer` conveniently enough).

```c
static int amdgpu_ttm_prepare_job(struct amdgpu_device *adev,
				  bool direct_submit,
				  unsigned int num_dw,
				  struct dma_resv *resv,
				  bool vm_needs_flush,
				  struct amdgpu_job **job)
{
	enum amdgpu_ib_pool_type pool = direct_submit ?
		AMDGPU_IB_POOL_DIRECT :
		AMDGPU_IB_POOL_DELAYED;
	int r;

	r = amdgpu_job_alloc_with_ib(adev, &adev->mman.entity,
				     AMDGPU_FENCE_OWNER_UNDEFINED,
				     num_dw * 4, pool, job);
	if (r)
		return r;

	if (vm_needs_flush) {
		(*job)->vm_pd_addr = amdgpu_gmc_pd_addr(adev->gmc.pdb0_bo ?
							adev->gmc.pdb0_bo :
							adev->gart.bo);
		(*job)->vm_needs_flush = true;
	}
	if (!resv)
		return 0;

	return drm_sched_job_add_resv_dependencies(&(*job)->base, resv,
						   DMA_RESV_USAGE_BOOKKEEP);
}
```

And `drm_sched_job_add_resv_dependencies` is the only place where `resv` is used. I don't think the implementation of `drm_sched_job_add_resv_dependencies` is super important here - we really more care about what it ends up doing, but essentially the documentation states:

```c
/**
 * drm_sched_job_add_resv_dependencies - add all fences from the resv to the job
 * @job: scheduler job to add the dependencies to
 * @resv: the dma_resv object to get the fences from
 * @usage: the dma_resv_usage to use to filter the fences
 *
 * This adds all fences matching the given usage from @resv to @job.
 * Must be called with the @resv lock held.
 *
 * Returns:
 * 0 on success, or an error on failing to expand the array.
 */
int drm_sched_job_add_resv_dependencies(struct drm_sched_job *job,
					struct dma_resv *resv,
					enum dma_resv_usage usage)
{
	/* ... */
```

Okay, now this thing is starting to make sense. We have a `drm_sched_job` which actually does the job at hand (the buffer copy), and jobs have a list of dependencies (fences) they must wait on before executing. So the `dma_resv` is just a list of all the dependencies for a given buffer object that e.g., a job accessing that buffer object must wait on.

The last three parameters of `amdgpu_copy_buffer` are `direct_submit`, `vm_needs_flush`, and `tmz`. `vm_needs_flush` is fairly straight-forward, basically signalling whether we need to flush the GPU virtual memory (GPUVM) (actally [here's](https://docs.kernel.org/gpu/amdgpu/amdgpu-glossary.html) a big glossary of all the acronyms that are popping up).

`direct_submit` is the difference between submitting our job via `amdgpu_job_submit_direct` if true, or `amdgpu_job_submit` if false. These two functions are also conveniently right next to each other so here they are:

```c
struct dma_fence *amdgpu_job_submit(struct amdgpu_job *job)
{
	struct dma_fence *f;

	drm_sched_job_arm(&job->base);
	f = dma_fence_get(&job->base.s_fence->finished);
	amdgpu_job_free_resources(job);
	drm_sched_entity_push_job(&job->base);

	return f;
}

int amdgpu_job_submit_direct(struct amdgpu_job *job, struct amdgpu_ring *ring,
			     struct dma_fence **fence)
{
	int r;

	job->base.sched = &ring->sched;
	r = amdgpu_ib_schedule(ring, job->num_ibs, job->ibs, job, fence);

	if (r)
		return r;

	amdgpu_job_free(job);
	return 0;
}
```

It seems that `amdgpu_job_submit_direct` inserts the job directly into the ring buffer, whereas `amdgpu_job_submit` schedules it with the DRM scheduler (driver-agnostic). I'm sort of inspecting this entire function tree in a vacuum so I don't know why you'd use one over the other, but I think looking at the user-space drivers would give us more information regarding that.

Finally, the `tmz` parameter... well, you'll see.

With all that (mostly) figured out, I can actually simplify the body of the `amdgpu_copy_buffer` function:

```c
int amdgpu_copy_buffer(struct amdgpu_ring *ring, uint64_t src_offset,
		       uint64_t dst_offset, uint32_t byte_count,
		       struct dma_resv *resv,
		       struct dma_fence **fence, bool direct_submit,
		       bool vm_needs_flush, bool tmz)
{
	/* initialize some variables */

	/* return an error if the ring isn't "ready" - required to submit our job */

	max_bytes = adev->mman.buffer_funcs->copy_max_bytes;
	num_loops = DIV_ROUND_UP(byte_count, max_bytes);
	num_dw = ALIGN(num_loops * adev->mman.buffer_funcs->copy_num_dw, 8);
	r = amdgpu_ttm_prepare_job(adev, direct_submit, num_dw,
				   resv, vm_needs_flush, &job, false);
	if (r)
		return r;

	for (i = 0; i < num_loops; i++) {
		uint32_t cur_size_in_bytes = min(byte_count, max_bytes);

		amdgpu_emit_copy_buffer(adev, &job->ibs[0], src_offset,
					dst_offset, cur_size_in_bytes, tmz);

		src_offset += cur_size_in_bytes;
		dst_offset += cur_size_in_bytes;
		byte_count -= cur_size_in_bytes;
	}

	/* submit our job */
}
```

Now I really have the core of this function. First thing I notice is that we copy in a loop. In fact, this emits `num_loop` buffer copy commands. Why? `num_loops = DIV_ROUND_UP(byte_count, max_bytes);` Right, so there's actually a limit to how many bytes we can copy at a time (from `adev->mman.buffer_funcs->copy_max_bytes`). Whether this is a limit of the GPU itself, the TTM, or PCIe, or whatever, I don't know (yet), but there's a limit. As such, we need to split up our copy up into chunks of `max_bytes`.

`num_dw` is the number of dwords our job will need to store all our commands. As you can imagine, it is `num_loop` times the number of dwords for a single buffer copy command (`adev->mman.buffer_funcs->copy_num_dw`). We'll actually see precisely what that number is soon. If we look back at `amdgpu_ttm_prepare_job` we can see that it's being passed to `amdgpu_job_alloc_with_ib` as `num_dw * 4`, thus `amdgpu_job_alloc_with_ib` will allocate a job with an IB (I'll get into that later) with `num_dw * 4` bytes.

Then we get to the loop where we emit the actual buffer copy commands, with our offsets incrementing as we go through each `max_bytes` "chunk" of the copy. Let's look into the `amdgpu_emit_copy_buffer` function now. Which, uhh, isn't a function? It's a macro that invokes a function pointer. Well, I might have an idea of why this is. AMD might change their encoding of copy buffer commands so that function may change depending on the specific version/model of GPU. I'll just look at one *possible* implementation, in `sdma_v2_4.c`:

```c
static void sdma_v2_4_emit_copy_buffer(struct amdgpu_ib *ib,
				       uint64_t src_offset,
				       uint64_t dst_offset,
				       uint32_t byte_count,
				       bool tmz)
{
	ib->ptr[ib->length_dw++] = SDMA_PKT_HEADER_OP(SDMA_OP_COPY) |
		SDMA_PKT_HEADER_SUB_OP(SDMA_SUBOP_COPY_LINEAR);
	ib->ptr[ib->length_dw++] = byte_count;
	ib->ptr[ib->length_dw++] = 0; /* src/dst endian swap */
	ib->ptr[ib->length_dw++] = lower_32_bits(src_offset);
	ib->ptr[ib->length_dw++] = upper_32_bits(src_offset);
	ib->ptr[ib->length_dw++] = lower_32_bits(dst_offset);
	ib->ptr[ib->length_dw++] = upper_32_bits(dst_offset);
}
```

There it is. The fruits of our labor. The exact format of the data that gets sent to the GPU (I think). So first off, we can see that this entire command takes 7 dwords which we can confirm further up in `sdma_v2_4.c`: `.copy_num_dw = 7`. Here we can also see that mysterious `tmz` parameter and that it does... nothing. Hah. I'm sure it had some now-obsolete purpose or is used in other implementations of `*_emit_copy_buffer`.

We send the header specifying what kind of command it is - `SDMA_OP_COPY | SDMA_SUBOP_COPY_LINEAR` - `SDMA_OP_COPY` looks to be an overarching copy operation, and `SDMA_SUBOP_COPY_LINEAR` specifies it is for linear memory. So I wonder if for optimal-layout image copies it would use `SDMA_OP_COPY | SDMA_SUBOP_COPY_TILED`? Regardless, everything else is as expected, sending the `byte_count` and the upper/lower dwords of the offsets. Also a dword set to 0 kindly annotated to explain that it is for swapping endianness (which I guess it doesn't need to do here!)

Along the way it increments an internal dword offset in `amdgpu_ib`, which explains why I didn't see anything to specify the byte offset for each command in the loop in `amdgpu_copy_buffer`.

Okay, one final struct to investigate: `amdgpu_ib`. IB stands for **indirect buffer** and is explained here:

```c
/*
 * IB
 * IBs (Indirect Buffers) and areas of GPU accessible memory where
 * commands are stored.  You can put a pointer to the IB in the
 * command ring and the hw will fetch the commands from the IB
 * and execute them.  Generally userspace acceleration drivers
 * produce command buffers which are send to the kernel and
 * put in IBs for execution by the requested ring.
 */
static int amdgpu_debugfs_sa_init(struct amdgpu_device *adev);
```

So quite simple really: It's just a buffer where we write our commands. Each job (optionally?) has an IB allocated along with it (see `amdgpu_job_alloc_with_ib` above). When we put our job into the `amdgpu_ring`, the hardware (the GPU) will read commands from that IB and execute it. In the future I'd like to dig deeper into what these indirect buffers *really are*.

### Exit DRM

With that, I think that concludes this first part of my journey! We started with a high-level command (`amdgpu_copy_buffer`) and worked out how it is initialized, where the command actually gets stored, how the command is written (and in the case of SDMA v2.4, the encoding of the command), and finally how the command is submitted and scheduled. There are plenty of loose ends still here that I haven't explored, but for now we'll leave DRM to another day. In part two, I'd like to look at the user-space driver and how it invokes `amdgpu_copy_buffer`.
