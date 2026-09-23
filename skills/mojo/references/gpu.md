# GPU kernels

Read this when code targets an accelerator. Since Mojo 1.0 the GPU APIs ship in the MAX package, and in 1.1 `std.gpu` is private. Docs: `max.modular.com/gpu/` and the API reference under `max.modular.com/api/mojo/max/gpu/`. Checked against those pages and the Mojo 1.0 and 1.1 changelogs.

## Imports

```mojo
from max.gpu import thread_idx, block_idx, block_dim, global_idx, barrier, WARP_SIZE
from max.gpu.host import DeviceContext
from max.gpu.memory import async_copy_commit_group, async_copy_wait_all
from max.gpu.primitives import warp
from layout import TileTensor, stack_allocation
from layout.tile_layout import row_major
from std.sys import has_accelerator
```

`layout` is bundled with MAX, not Mojo. MAX and Mojo versions should come from the same release channel; mismatches show up as kernel compile failures.

## Launching

- Guard host code with `comptime if not has_accelerator(): ...`. `has_*` functions ask about the host; `is_nvidia_gpu()`, `is_amd_gpu()`, `is_apple_gpu()` (from `std.sys`) ask about the compilation target and belong inside GPU code.
- `ctx.enqueue_function[kernel](args..., grid_dim=..., block_dim=...)` takes the kernel once (since 1.0.0b2; the two-argument form is deprecated) and type-checks the arguments at compile time.
- `Int` and `UInt` are not `DevicePassable` since 1.0: pass fixed-width types such as `Int32` or `UInt32` to kernels, because host and device can disagree on the width of `Int`.
- The grid is usually rounded up, so bounds-check: `if global_idx.x < n:`.

## TileTensor

- A function that subscripts a `TileTensor` needs proof of its rank: `comptime assert tensor.flat_rank == 2` in the body, or a trailing `where tensor.flat_rank == 2`. Without it indexing fails with "lacking evidence to prove correctness". Tensors derived with `.tile()`, `.vectorize()` or `.distribute()` have a new layout, so assert again before indexing them.
- Shared memory: `stack_allocation[...]` from `layout` returns a `TileTensor` in shared address space (a static allocation despite the name). `AddressSpace` lives in `std.memory.address_space`.
- Async staging to shared memory: a copier such as `GenericToSharedAsyncTileCopier[thread_layout]().copy(dst, src)` on vectorized tiles, then `async_copy_commit_group()`, `async_copy_wait_all()` and `barrier()` before reading. `TileTensor.copy_from()` is the synchronous copy; on GPU, `distribute()` the tensors across threads first.
- When operands come from tensors with different layouts, element types may not unify; `rebind[Scalar[dtype]](tensor[i])` converts each operand.

## Warps and blocks

- `WARP_SIZE` depends on hardware (32 on NVIDIA and RDNA, 64 on CDNA); use the constant.
- `warp.shuffle_down(val, offset)` and `shuffle_xor` take `offset: UInt32`; the masked overloads take `mask: UInt`. Values from lanes past the end are undefined. Reductions such as `warp.sum()` and `warp.max()` give every lane the result; shuffles do not.
- `barrier()` synchronizes a block; calling it inside a branch that not every thread reaches deadlocks.
- `Atomic` is parameterized on a value type in 1.1 (`Atomic[Float32]`, not `Atomic[DType.float32]`).

## Performance habits

Coalesce global memory access (adjacent threads touch adjacent addresses), stage reused tiles through shared memory, and aim for enough occupancy to hide latency rather than the maximum. Profile before and after; a high cache hit rate can hide uncoalesced access.
