# Learning Guide: Mojo Programming Language - Fundamentals, Idiomatic Best Practices, Performance, and GPU Programming

Generated: 2026-05-21
Sources: 41 resources analyzed (heavily weighted toward official Modular / mojolang.org docs)
Depth: deep
Target language version: Mojo 1.0.0b1 (2026-05-07) and the nightly stream that follows it

## CRITICAL VERSION CONTEXT - READ FIRST

Mojo is a young, fast-moving language. As of early 2026 it crossed a major threshold: the path to Mojo 1.0. Several long-standing keywords and idioms from pre-2025 tutorials are now DEPRECATED or REMOVED. A coding agent that learned Mojo from 2023-2024 material will write code that warns or fails to compile today.

The official docs moved from `docs.modular.com/mojo/` to `mojolang.org/docs/` (301 redirects in place). Both reflect current content; prefer `mojolang.org`.

Treat the following as the single most important table in this guide. When you see the LEFT column in code or documentation, it is stale - use the RIGHT column.

| Stale (pre-2025, avoid) | Current (Mojo 1.0, use this) | Notes |
|---|---|---|
| `fn` keyword | `def` | `fn` now emits a deprecation warning (slated to become a compile error). Write top-level and method functions as `def`. Exception: nested `@parameter fn` closures still use `fn` today - verify against your toolchain. |
| `owned arg` | `var arg` | The `owned` argument convention was removed. `var` means the function takes ownership. |
| `borrowed arg` | `read arg` (or omit - it is the default) | `borrowed` removed; `read` is the immutable-reference convention and the default. |
| `inout arg` | `mut arg` | `inout` removed; `mut` is the mutable-reference convention. |
| `__copyinit__(inout self, existing)` | `__init__(out self, copy: Self)` (copy constructor) | Synthesized automatically for `Copyable` types. |
| `__moveinit__(inout self, owned existing)` | `__init__(out self, take: Self)` (move constructor) | Synthesized automatically for `Movable` types. |
| `@value` decorator | `@fieldwise_init` (+ derive `Copyable`, `Movable`) | `@value` superseded. |
| `@register_passable` | Conform to traits / `RegisterPassable` trait | Decorator removed. |
| `NDBuffer` | `TileTensor` / `LayoutTensor` | `NDBuffer` removed entirely. The GPU tutorials now show `TileTensor`. |
| `mojo test` CLI command | `TestSuite` struct + `mojo run test_file.mojo` | The `mojo test` command was removed 2025-10-31. |
| `from pkg import x` making `pkg` available | import only binds `x` | Import semantics tightened. |
| `gpu.block.sum(...)` | `gpu.primitives.block.sum(...)` | GPU collective ops moved under `gpu.primitives`. |
| GPU `thread_idx` as `UInt` | `thread_idx` as `Int` | Primitive ID accessors migrated `UInt -> Int` (temporary `*_uint` aliases exist). |
| `Consistency` / `MONOTONIC` | `Ordering` / `RELAXED` (in `std.atomic`) | Atomics reorganized into `std.atomic`. |
| `\xhh` as raw byte | `\xhh` as Unicode code point | String escape semantics now match Python; use `List[Byte]` for raw bytes. |
| negative indexing `lst[-1]` | not supported on stdlib collections | Negative indexing removed; bounds checking on by default on CPU. |

Version timeline (for dating any claim):
- v0.25.6 (2025-09-22), v0.25.7 (2025-11-20), v0.26.1 (2026-01-29), v0.26.2 (2026-03-19), v1.0.0b1 (2026-05-07, first 1.0 beta).
- Mojo 1.0 brings semantic versioning, stabilized-API markers, and backward compatibility across the 1.x series. Breaking changes after 1.0 will land behind an "experimental" flag (Mojo 2.0 model), opt-in per package.

Whenever you assert something that may be unstable, qualify it with a version (e.g., "as of v1.0.0b1"). The standard library outside stabilized-API markers can still change.

## Prerequisites

- Comfort with Python syntax (Mojo is a Python superset in spirit, not byte-for-byte compatible).
- Basic systems-programming concepts: stack vs heap, value vs reference, compile time vs run time.
- For GPU sections: a mental model of data-parallel execution (threads, blocks, grids) helps; CUDA/HIP experience transfers but is not required.
- Toolchain: install via `pixi` (recommended). `pixi` produces `pixi.lock` and a `.pixi/` conda environment. Mojo uses your environment's CPython for Python interop - it does not bundle one.

## TL;DR

- Use `def` for every function. `fn` is deprecated. Functions are non-raising by default; add `raises` (preferably typed: `raises MyError`) to propagate errors.
- Argument conventions: `read` (default, immutable ref), `mut` (mutable ref), `var` (takes ownership), `ref` (parametric mutability), plus `out` (constructors/named results) and `deinit` (destructors). The old `owned`/`borrowed`/`inout` are gone.
- `struct` is the workhorse type: static, no inheritance, fields declared with `var`, behavior shared via traits (`Copyable`, `Movable`, `Stringable`, `Writable`, ...). Use `@fieldwise_init` instead of the old `@value`.
- Compile-time `[parameters]` vs run-time `(arguments)` is the core of Mojo metaprogramming. `comptime` (values/if/for) and `@parameter` (loop unrolling, parametric closures) drive specialization and zero-cost abstraction.
- Performance comes from SIMD (`SIMD[dtype, size]`), specialization, avoiding copies (ownership + `^` transfer + ASAP destruction), `@always_inline`, and the `algorithm` module (`elementwise`, `vectorize`, `parallelize`).
- GPU programming is first-class and vendor-agnostic (NVIDIA CUDA, AMD HIP, Apple Metal). `DeviceContext` drives allocation/launch; kernels index with `thread_idx`/`block_idx`/`block_dim`/`grid_dim` (now `Int`); `LayoutTensor`/`TileTensor` + tiling + shared memory + `barrier()` are the optimization toolkit.

---

## 1. Language Fundamentals

### 1.1 What Mojo is

Mojo's vision: one language to target CPUs, GPUs, and other accelerators with Python's syntax plus modern systems-programming features (static types, memory safety, compile-time metaprogramming). It is compiled (JIT on `mojo run`, AOT on `mojo build`), built on MLIR, and interoperates natively with Python so you can replace hot functions incrementally.

### 1.2 `def` vs `fn` (current: just use `def`)

There is now ONE function keyword you should write: `def`. `fn` exists only for legacy code and warns. A `def` function:

```mojo
def greet(name: String) -> String:
    return "Hello, " + name
```

- Type annotations on arguments and return are encouraged and often required for performance and for use as parameters.
- Functions are non-raising by default. Mark raising functions with `raises` (see Error Handling).
- The historical distinction ("`fn` is strict/typed, `def` is dynamic/Pythonic") is being collapsed; modern `def` carries the static, checked behavior. Do not teach the old dichotomy.

### 1.3 `var` and variables

Local variables are introduced with `var` (explicit) or by first assignment in `def` bodies. Struct fields MUST use `var` with an explicit type annotation and cannot be initialized at the field declaration site - initialize them in a constructor.

```mojo
struct Point:
    var x: Int
    var y: Int
```

### 1.4 Value semantics, ownership, and argument conventions

Every value has exactly one owner. When the owner's lifetime ends, Mojo destroys the value (ASAP destruction - see 2.3). Argument conventions declare how a function relates to a value:

| Convention | Mutable? | Owns? | Meaning |
|---|---|---|---|
| `read` (default) | No | No | Immutable reference to the caller's value (no copy). |
| `mut` | Yes | No | Mutable reference; mutations are visible to the caller. |
| `var` | Yes | Yes | Function takes ownership. Pair with the `^` transfer sigil at the call site to end the caller's variable. |
| `ref` | parametric | No | Generalizes `read`/`mut`; mutability inferred from origin. Advanced. |
| `out` | (special) | n/a | Uninitialized on entry, must be initialized before return. Used for `__init__` self and named results. |
| `deinit` | (special) | n/a | Initialized on entry, uninitialized on exit. Used for `__del__` self. |

```mojo
def print_list(list: List[Int]):     # read (default): immutable borrow
    print(list.__str__())

def mutate(mut l: List[Int]):        # mutable borrow
    l.append(5)

def take_text(var text: String):     # takes ownership
    text += "!"
    print(text)

def caller():
    var s = String("hi")
    take_text(s^)                    # ^ transfers ownership; s is no longer usable after this
```

Key correctness rules:
- The `^` transfer sigil moves a value into a `var` parameter (or out of scope) and ends the source binding.
- If a type is `Copyable` and you do NOT use `^`, passing to a `var` parameter copies. The compiler applies a "copy-to-move" optimization on a variable's last use, turning the final copy into a move automatically.

### 1.5 Lifetimes, origins, and references

An origin tracks which value a reference points to so the compiler can extend lifetimes and prevent use-after-free. Origins are compile-time parameters the compiler usually infers.

- `ImmutOrigin` / `MutOrigin` (and aliases like `MutAnyOrigin`, `MutExternalOrigin`) specify reference origins, commonly seen on `UnsafePointer` and `LayoutTensor`/`TileTensor` type parameters.
- `ref` arguments and returns carry an origin so a returned reference stays valid as long as its source does.
- You rarely write origins by hand for everyday code; they show up in generic library code and GPU tensor types.

### 1.6 Structs

A `struct` is Mojo's primary user-defined type. Compared to a Python class it is fully static (bound at compile time): no monkey-patching, no dynamic dispatch, no class attributes, and no inheritance. Use traits for shared behavior.

```mojo
@fieldwise_init                  # auto-generates a field-wise __init__
struct MyPair(Copyable, Movable):
    var first: Int
    var second: Int

    def increment_first(mut self):
        self.first += 1

    @staticmethod
    def origin() -> MyPair:
        return MyPair(0, 0)
```

- `__init__` takes `out self` (uninitialized, must be fully initialized before return).
- Instance methods take `self` (immutable) or `mut self` (to mutate) or `var self` (to consume).
- Use `@staticmethod` for methods without `self`.
- `@fieldwise_init` replaces the old `@value` decorator for generating a constructor.

### 1.7 Traits (composition over inheritance)

Traits are contracts (like Rust traits / Java interfaces / C++ concepts). A struct conforms by listing the trait and implementing its requirements.

```mojo
trait Quackable:
    def quack(self): ...           # required method (no body)

trait DefaultQuackable:
    def quack(self):               # default implementation
        print("Quack")

struct Duck(Copyable, Quackable):
    def quack(self):
        print("Quack")

# Trait bound on a generic function
def make_it_quack[T: Quackable](x: T):
    x.quack()

# Shorthand "some" form
def make_it_quack2(x: Some[Quackable]):
    x.quack()

# Composition with & ; and reusable alias
comptime DuckLike = Quackable & Flyable
def go[T: DuckLike](d: T):
    d.quack(); d.fly()

# Refinement (trait extends trait)
trait Animal:
    def make_sound(self): ...
trait Bird(Animal):
    def fly(self): ...
```

Common built-in traits: `Copyable`, `Movable`, `ImplicitlyCopyable`, `Sized` (`__len__`), `Intable`, `Stringable`, `Writable` (`write_to`), `Defaultable`, `Equatable`, `RegisterPassable`. Traits may require `@staticmethod`s and may declare `comptime` members for generic compile-time constants.

You cannot retroactively add a trait to an existing third-party type; design with traits up front.

### 1.8 Parameters vs arguments (compile-time `[]` vs run-time `()`)

This is the defining feature of Mojo metaprogramming. "Parameter" always means a compile-time value; "argument" always means a run-time value. Parameters go in `[]` before the `()` argument list. The compiler generates a specialized instantiation per unique parameter set (monomorphization), which is the basis of zero-cost abstraction.

```mojo
def repeat[count: Int](msg: String):
    @parameter
    for i in range(count):         # unrolled at compile time
        print(msg)

repeat[3]("Hello")                 # count is a parameter; "Hello" is an argument
```

Parameterized struct:

```mojo
struct GenericArray[ElementType: Copyable & Movable]:
    var data: UnsafePointer[Self.ElementType]
    var size: Int
```

Advanced parameter forms:
- Optional parameters with defaults: `def speak[a: Int = 3, msg: String = "woof"]()`.
- Infer-only parameters before `//`: `def f[dtype: DType, //, value: Scalar[dtype]]()` - `dtype` is inferred from `value`.
- Variadic parameters: `struct MyTensor[*dimensions: Int]`.
- Automatic parameterization with `...`: `def take_simd(vec: SIMD[...])` infers `dtype`/`size`.
- Partially-bound aliases: `comptime StringKeyDict = Dict[String, _]`.
- `rebind[T](x)` asserts parametric type equality when the compiler cannot prove it (common in static dispatch on a parameter value).

### 1.9 `comptime` metaprogramming and `@parameter`

Mojo uses the SAME language for compile-time and run-time code. Triggers for compile-time execution: assigning to a `comptime` value, a `comptime if`/`comptime for`, or binding a compile-time parameter.

```mojo
comptime PI = 3.14159265359          # compile-time constant (replaces old `alias`)
comptime AddOne[a: Int]: Int = a + 1 # parameterized comptime value
comptime nine = AddOne[8]

comptime if target_is_gpu():         # only the taken branch compiles
    ...
```

`@parameter` controls compile-time behavior of language constructs:
- `@parameter for ...` fully unrolls a loop whose bounds are compile-time known.
- `@parameter if ...` selects a branch at compile time (legacy spelling; `comptime if` is the modern form).
- `@parameter` on a nested closure makes it a parametric closure usable in compile-time positions and captured correctly (origins of captured values are tracked).

Note: `comptime` is the modern keyword. Older code uses `alias` for compile-time constants - treat `alias` as the predecessor of `comptime` values.

---

## 2. Idiomatic Best Practices

### 2.1 When to use `def`, `mut`, `var`

- Default to `def` and let arguments be `read` (omit the convention) unless you need to mutate or own.
- Use `mut` when a function must modify the caller's value in place (e.g., `append` to a list).
- Use `var` (with caller `^`) when the function consumes/stores the value, to avoid a copy.
- Reach for `ref` only in generic library code that must be mutability-polymorphic.

### 2.2 Error handling (errors as values)

Mojo models errors as alternate return values - no stack unwinding, overhead close to returning and checking a `Bool`. This is also why typed errors work inside GPU kernels (no heap allocation).

```mojo
def read_file(path: String) raises -> String:
    if not path:
        raise "path cannot be empty"     # String shorthand for Error
    return "..."

# Prefer TYPED errors at API boundaries
@fieldwise_init
struct ValidationError(Copyable, Writable):
    var field: String
    var reason: String
    def write_to(self, mut writer: Some[Writer]):
        writer.write("ValidationError(", self.field, "): ", self.reason)

def validate(username: String) raises ValidationError -> String:
    if username.count_codepoints() < 3:
        raise ValidationError(field="username", reason="too short")
    return username

def handle():
    try:
        _ = validate("ab")
    except e:
        print("handled:", e)
    else:
        print("ok")          # only if no error and no early return/break/continue
    finally:
        print("cleanup")     # always runs
```

Idioms and advanced features:
- Prefer `raises MyError` (typed) over bare `raises` for compile-time safety; each function declares exactly one error type.
- `raise e^` to re-raise by transferring ownership.
- `-> Never` return type expresses "always raises / never returns" (e.g., `panic`); `raises Never` means cannot raise.
- Parametric raises: `def run[E: AnyType](action: def() thin raises E -> Int) raises E -> Int` propagates the callee's error type.
- Use a `Variant[...]` error or an enumerated-variant struct when multiple error shapes are needed.
- Context managers: `with open(path) as f:` and custom `__enter__`/`__exit__` for deterministic cleanup; `__exit__(self, e: Error) -> Bool` returning `True` suppresses the error.
- Python interop tends to raise more, so interop functions usually need `raises`.

### 2.3 Memory management and the value lifecycle

Mojo has no garbage collector and no reference counting for value types. Memory safety comes from ownership + ASAP destruction: a value is destroyed immediately after its last use (the compiler inserts `__del__` calls after each sub-expression where possible), not at end of scope.

Lifecycle methods (CURRENT names):

| Event | Method | Notes |
|---|---|---|
| Construct | `__init__(out self, ...)` | No default constructor is synthesized. |
| Copy construct | `__init__(out self, copy: Self)` | Auto-synthesized if you derive `Copyable`. |
| Move construct | `__init__(out self, take: Self)` | Auto-synthesized if you derive `Movable`. |
| Destroy | `__del__(deinit self)` | Trivial no-op destructor added if you define none. |

- Structs are NOT copyable or movable by default. Derive `Movable`, `Copyable`, or `ImplicitlyCopyable` (implies both; signals copies are cheap/implicit) as appropriate.
- The compiler synthesizes `__init__(copy=)`, `__init__(take=)`, and `copy()` for conforming types that do not define them.
- `__moveinit__`/`__copyinit__` are legacy - migrate to the unified `__init__`.
- Never call lifecycle dunders directly; the compiler invokes them.

### 2.4 Struct design

- Keep fields minimal and typed; initialize all fields in `__init__`.
- Derive the smallest trait set you need; only derive `ImplicitlyCopyable` when copies are genuinely cheap.
- Provide `Writable` (`write_to`) instead of ad hoc string building for printable types; provide `Equatable` for value comparison.
- Prefer named-result `out` returns for types that are expensive or impossible to move.

### 2.5 Traits over inheritance

There is no class inheritance. Model "is-a" relationships with trait refinement and "has-a" with composition. Use generic functions bounded by traits (`def f[T: SomeTrait](x: T)`) for static polymorphism with zero dispatch overhead.

### 2.6 Pythonic interop

```mojo
from std.python import Python

def main() raises:
    np = Python.import_module("numpy")        # whole module only; no `from numpy import array`
    a = np.array(Python.list(1, 2, 3))
    print(a)
```

- All Python values are `PythonObject`. Conversions to/from Mojo types happen at the boundary.
- Imports must be inside functions, not at module top level.
- `mojo build` does not bundle Python packages; deploy the same `pixi` environment.
- Crossing into Python is a performance boundary - keep hot loops in Mojo and call Python for glue/IO only.

### 2.7 Naming, project layout, testing

- Follow Python-style naming: `snake_case` for functions/variables, `PascalCase` for structs/traits, `SCREAMING_SNAKE_CASE` or `comptime PascalCase` for compile-time constants.
- Use `pixi` to scaffold and lock projects (`pixi.lock`, `.pixi/`). Store conda recipes at `conda.recipe/recipe.yaml` for packaging with `rattler-build`.
- Testing (current): the `mojo test` CLI was removed (2025-10-31). Use the `TestSuite` struct:

```mojo
from std.testing import assert_equal, assert_raises, TestSuite

def test_inc_one() raises:                 # name starts with test_, no args, returns None, raises on failure
    assert_equal(inc(1), 2)

def test_bad_input() raises:
    with assert_raises(contains="required"):
        raise Error("missing required argument")

def main() raises:
    TestSuite.discover_tests[__functions_in_module()]().run()
```

Run with `mojo run test_file.mojo`. Assertions: `assert_true/false`, `assert_equal/not_equal`, `assert_almost_equal` (floats), `assert_raises`.

### 2.8 Common pitfalls (idiomatic)

| Pitfall | Why it happens | How to avoid |
|---|---|---|
| Writing `fn` | Old tutorials | Use `def`; `fn` warns and will error. |
| Using `owned`/`borrowed`/`inout` | Pre-2025 docs | Use `var`/`read`/`mut`. |
| Defining `__copyinit__`/`__moveinit__` | Legacy lifecycle | Derive `Copyable`/`Movable`; rely on synthesized `__init__(copy=)`/`__init__(take=)`. |
| Negative indexing `lst[-1]` | Python habit | Removed on stdlib collections; index from the front or compute `len-1`. |
| Expecting copies to be free | Value semantics | Use `^` to move, `read`/`ref` to borrow; profile copy hotspots. |
| `from pkg import x` and using `pkg.y` | Old import behavior | Import what you use; `pkg` is not bound by a `from` import. |
| Treating struct like a Python class | Dynamic expectations | No monkey-patching/inheritance/dynamic attrs; use traits and parameters. |
| Assuming `mojo test` works | Removed | Use `TestSuite`. |

---

## 3. Performance

Mojo's performance model: write clear algorithms, then guide the compiler with types, parameters, SIMD, and the `algorithm` module. Specialization (monomorphization over `[parameters]`) plus inlining produces machine code competitive with C/C++/CUDA.

### 3.1 SIMD and vectorization

`SIMD[dtype, size]` maps directly to hardware vector registers (AVX, NEON, etc.); `size` must be a power of two. Every scalar numeric type is an alias: `Float32 == SIMD[DType.float32, 1] == Scalar[DType.float32]`.

```mojo
var a = SIMD[DType.float32, 8](1, 2, 3, 4, 5, 6, 7, 8)
var b = SIMD[DType.float32, 8](2.0)        # splat
var c = a * b + 1.0                        # element-wise, single op
var d = a.fma(b, c)                        # fused multiply-add: a*b + c
var total = a.reduce_add()                 # horizontal reduce
var ints = a.cast[DType.int32]()           # float->int floors
var mask = a.gt(4.0)
var sel = mask.select(a * 2, a)            # branchless select
```

Guidance:
- Choose `size` near the hardware native width; query it with `sys.simd_width_of[DType.float32]()` (older spelling `simdwidthof`). Avoid vectors more than ~2x the native register size.
- Express arithmetic on whole vectors; let the compiler fuse and schedule.

### 3.2 The `algorithm` module: `vectorize`, `parallelize`, `elementwise`

The `algorithm` package provides higher-order parallel/vector primitives.

- `vectorize[func, simd_width](size)` maps a SIMD-width function across `[0, size)`, handling the `size % simd_width` remainder.
- `parallelize[func](num_work_items, num_workers)` distributes work across CPU threads.
- `elementwise[func, simd_width, rank](shape)` executes a width/rank-parameterized function across an N-D shape, possibly as parallel sub-tasks - the modern, layout-aware workhorse.

```mojo
from algorithm import vectorize, parallelize

comptime W = sys.simd_width_of[DType.float32]()

def saxpy(n: Int, a: Float32, x: UnsafePointer[Float32], y: UnsafePointer[Float32]):
    @parameter
    fn body[w: Int](i: Int):
        var xi = x.load[width=w](i)
        var yi = y.load[width=w](i)
        (x + i).store(a * xi + yi)
    vectorize[body, W](n)
```

The inner `@parameter fn body` is intentional: nested parametric closures still use `fn` (the `def`-not-`fn` rule targets top-level and method functions). The outer `saxpy` is `def`.

Note (as of v1.0.0b1): `parallelize()` and `elementwise()` accept an optional trailing `ctx: Optional[DeviceContext]` forwarded to `sync_parallelize()`, unifying CPU/GPU dispatch. Some pre-1.0 `vectorize`/`parallelize` call shapes have shifted; verify signatures against the installed version.

### 3.3 Inlining and specialization

- `@always_inline` forces inlining of a function - essential for tiny kernels (SIMD bodies, accessors) so the compiler can fuse and eliminate call overhead. `@always_inline("nodebug")` also hides the frame.
- Parameterize on `DType`, sizes, and layouts so the compiler specializes and unrolls (`@parameter for`).
- Use `comptime`/`alias` to precompute constants and tables at compile time.

### 3.4 Memory layout and copy avoidance

- Prefer contiguous, well-aligned layouts; use `LayoutTensor`/`Layout.row_major(...)` to make layout explicit and enable tiling/vectorization without copies.
- Avoid unnecessary copies: borrow with `read`/`ref`, move with `^`, take ownership with `var`. Rely on copy-to-move on last use, but verify hot paths.
- Use `UnsafePointer` for raw buffers in performance-critical code. Note: `UnsafePointer` is now non-null by design (v1.0.0b1); use `Optional[UnsafePointer[...]]` for nullable pointers with zero-overhead layout.

### 3.5 Benchmarking with the `benchmark` module

```mojo
from benchmark import Bench, BenchConfig, Bencher, BenchId, keep, run

def main():
    var report = run[my_workload]()       # auto-tunes iteration counts
    report.print()                        # mean/min/max, iterations
```

- `run[f]()` decides batch sizes from observed timing and reports averages.
- Use `keep(value)` to prevent the optimizer from deleting work whose result is unused (dead-code elimination would otherwise invalidate the measurement).
- `Bench` + `Bencher` + `BenchConfig` orchestrate multiple named benchmarks for comparison.
- MAX ships `kbench` (a Python tool) to benchmark, autotune, and analyze MAX kernel performance.

### 3.6 Autotuning

Autotuning explores parameter spaces (tile sizes, block dims, unroll factors) to pick the best for the target hardware. In practice today this is done by parameterizing kernels over `[comptime]` choices and sweeping them with `benchmark`/`kbench`, rather than a single magic decorator. Keep tunable knobs as parameters so they specialize cleanly.

---

## 4. GPU Programming

Mojo makes GPU programming first-class and vendor-agnostic: the same language and largely the same code target NVIDIA (CUDA), AMD (HIP), and Apple Silicon (Metal). No vendor C++ toolchain is required.

### 4.1 Execution model

- A grid contains thread blocks; each block contains threads; blocks/threads are 1-3 dimensional.
- The hardware schedules blocks onto streaming multiprocessors (SMs) and runs threads in warps (32 on NVIDIA, 64 on AMD wavefronts, SIMD-groups on Metal). Use `gpu.WARP_SIZE`, never a hardcoded 32.
- Kernels: non-raising functions whose arguments conform to `DevicePassable`; they cannot return values - write results into passed buffers/tensors.

### 4.2 Indexing (now `Int`)

Inside a kernel:
- `thread_idx.{x,y,z}` - thread position within its block.
- `block_idx.{x,y,z}` - block position within the grid.
- `block_dim.{x,y,z}` - threads per block.
- `grid_dim.{x,y,z}` - blocks per grid.
- `global_idx` - convenience global position; `global_idx.x = block_dim.x * block_idx.x + thread_idx.x`.

These are `Int` as of recent releases (migrated from `UInt`; temporary `*_uint` aliases exist). Always bounds-check `if tid < n:` because grid size is usually rounded up.

### 4.3 DeviceContext: allocate, copy, launch, synchronize

`DeviceContext` is a single stream of execution on one accelerator. Operations are enqueued asynchronously and execute in order; `synchronize()` blocks the host until the queue drains.

```mojo
from gpu.host import DeviceContext
from math import ceildiv

def main() raises:
    var ctx = DeviceContext()                                   # optional device_id, api
    comptime n = 1024
    var host = ctx.enqueue_create_host_buffer[DType.float32](n)
    var dev  = ctx.enqueue_create_buffer[DType.float32](n)
    # fill host ...
    ctx.enqueue_copy(dst_buf=dev, src_buf=host)

    comptime block = 256
    var grid = ceildiv(n, block)
    ctx.enqueue_function[kernel, kernel](dev, n, grid_dim=grid, block_dim=block)

    ctx.enqueue_copy(dst_buf=host, src_buf=dev)
    ctx.synchronize()
```

- `compile_function[...]` precompiles a kernel for repeated launches; `enqueue_function[...]` compiles-and-launches in one call.
- Detect hardware: `has_nvidia_gpu_accelerator()`, `has_amd_gpu_accelerator()`, `has_apple_gpu_accelerator()`.
- CPU `DeviceContext` now supports stream-ordered execution (`enqueue_cpu_function()`, `enqueue_cpu_range()`), so the same dispatch shape can run on CPU.

### 4.4 A complete kernel (vector add, TileTensor)

```mojo
from gpu.id import thread_idx, block_idx, block_dim

def vector_addition(
    lhs: TileTensor[float_dtype, type_of(layout), MutAnyOrigin],
    rhs: TileTensor[float_dtype, type_of(layout), MutAnyOrigin],
    out_t: TileTensor[float_dtype, type_of(layout), MutAnyOrigin],
):
    var tid = block_idx.x * block_dim.x + thread_idx.x
    if tid < vector_size:                       # always guard
        out_t[tid] = lhs[tid] + rhs[tid]
```

The intro tutorial wraps device buffers in `TileTensor(buffer, layout)` and launches with `grid_dim=ceildiv(n, block)`, `block_dim=block`. `TileTensor` is the successor to the removed `NDBuffer`; the manual's `LayoutTensor` API is the same family of layout-aware tensor view.

### 4.5 Memory hierarchy

| Space | Speed | Scope | Mojo usage |
|---|---|---|---|
| Registers | fastest | per-thread | Local `var`s; warp shuffles exchange register values. |
| Shared | fast on-chip | per-block | `LayoutTensor[..., address_space=AddressSpace.SHARED].stack_allocation()` or `stack_allocation[..., address_space=AddressSpace.SHARED]`. |
| Global | high latency | whole grid | Device buffers; coalesce accesses. |

### 4.6 LayoutTensor / TileTensor and tiling

A layout tensor is a non-owning view = layout + `DType` + pointer. It enables tiling, vectorized loads/stores, and thread-distribution without copies.

```mojo
comptime layout = Layout.row_major(8, 16)
var t = LayoutTensor[DType.float32, layout](storage)

# Tile for shared-memory blocking
var global_tile = t.tile[16, 16](Int(block_idx.y), Int(block_idx.x))
var shared = LayoutTensor[
    DType.float32, tile_layout, MutAnyOrigin, address_space=AddressSpace.SHARED
].stack_allocation()
shared_fragment.copy_from_async(global_fragment)   # async global->shared
barrier()                                          # all threads see loaded tile

# Vectorize a view; distribute fragments across threads
var vt = t.vectorize[1, 4]()
var frag = tile.vectorize[1, 4]().distribute[thread_layout](lane_id())
```

Tiling pattern for matmul: cooperatively load A/B tiles into shared memory, `barrier()`, compute, `barrier()`, advance. 2-D block tiling (each thread computes a TMxTN output tile) raises arithmetic intensity; threads can also stage 8-wide vectors in registers.

### 4.7 Synchronization, block, and warp operations

- `barrier()` - block-wide execution barrier AND memory fence. Every thread in the block MUST reach it (never put `barrier()` inside a divergent branch -> deadlock). Pattern: load -> `barrier()` -> compute -> `barrier()`.
- Block collectives (in `gpu.primitives.block`): `sum`, `max`, `min`, `broadcast(val, src_thread=0)`, `prefix_sum[exclusive=False]`. They handle synchronization internally.
- Warp ops (faster than block reductions; register-to-register): `shuffle_up/down/xor/idx`, `broadcast`, `warp.sum/max/min`, `warp.prefix_sum`. `syncwarp()` handles divergence reconvergence; it is NOT needed before shuffles/reductions (they sync implicitly).
- Portability: AMD wavefronts run lock-step (barriers can be no-ops); NVIDIA Volta+ has independent thread scheduling; Metal uses SIMD-group sync. Mojo abstracts these - rely on the primitives.

### 4.8 Occupancy and performance tuning

Occupancy = active warps / max warps per SM; it lets the GPU hide latency by switching warps.

```
Theoretical Occupancy = min(
    RegsPerSM / (RegsPerThread * ThreadsPerBlock),
    SharedPerSM / SharedPerBlock,
    MaxBlocksPerSM
) * ThreadsPerBlock / MaxThreadsPerSM
```

Rules of thumb:
- 25-50% occupancy is usually enough for latency hiding; beyond ~50-70% returns diminish for memory-bound work. Do not chase 100%.
- For memory-bound kernels, memory bandwidth and access pattern dominate occupancy. Measured example: a strided kernel (`i += 512`) hit ~6 GB/s with 99% L2 hits, while a coalesced kernel reached ~310 GB/s.
- Sweep block sizes (256/512/1024); larger blocks may hit register/shared limits but a thread-per-SM cap often dominates.
- Watch register and shared-memory pressure (e.g., 19 vs 25 vs 40 registers/thread changes how many blocks fit).

### 4.9 GPU performance patterns and pitfalls

| Pattern / Pitfall | Guidance |
|---|---|
| Memory coalescing | Make consecutive threads touch consecutive addresses; non-coalesced strides destroy bandwidth. |
| High cache hit rate | Can be a RED FLAG - re-reading slow paths cheaply, not a win. Profile bandwidth, not just hit rate. |
| Shared-memory bank conflicts | Different threads hitting different addresses in the same bank serialize; pad layouts / use conflict-free access patterns. |
| Overusing shared memory / barriers | Prefer warp-level ops within a warp; use `barrier()` + shared memory between warps. |
| Async copy overlap | Use `copy_from_async` to overlap global->shared transfers with compute. |
| Divergent barriers | Never call `barrier()` inside an `if` that not all threads take. |
| Hardcoded warp size | Use `gpu.WARP_SIZE`. |

### 4.10 MAX integration

MAX (the Modular Platform) is built in Mojo and ships the world's largest open-source library of CPU/GPU kernels (`modular/modular`, `/max/kernels`). You can:
- Write hardware-agnostic custom ops/kernels in Mojo and plug them into MAX model pipelines (e.g., custom PyTorch ops, custom matmul).
- Use MAX tooling (`kbench`) to benchmark and autotune kernels.
- Rely on MAX for unified compute across NVIDIA/AMD/Apple. MAX 26.x version numbers track alongside Mojo (MAX 26.3 ~ Mojo 1.0.0b1, 2026-05-07).

---

## Best Practices Checklist (apply directly when writing Mojo)

Language and syntax
- [ ] Use `def`, never `fn`.
- [ ] Use argument conventions `read` (default) / `mut` / `var` / `ref`; never `owned`/`borrowed`/`inout`.
- [ ] Declare struct fields with `var` + explicit type; initialize all fields in `__init__`.
- [ ] Use `@fieldwise_init`, not `@value`. Do not use `@register_passable`.
- [ ] Implement lifecycle via derived `Copyable`/`Movable` and synthesized `__init__(copy=)`/`__init__(take=)`; never write `__copyinit__`/`__moveinit__`.
- [ ] Share behavior via traits + `Some[Trait]`/`[T: Trait]` bounds; no inheritance.
- [ ] Use `comptime` for compile-time constants/values; put compile-time inputs in `[]`, run-time in `()`.

Correctness and safety
- [ ] Mark raising functions `raises`; prefer typed `raises MyError`.
- [ ] Use `try/except/else/finally`; re-raise with `raise e^`.
- [ ] No negative indexing on stdlib collections; respect default CPU bounds checking.
- [ ] Use `Optional[UnsafePointer[...]]` for nullable pointers (UnsafePointer is non-null).
- [ ] Move with `^`, borrow with `read`/`ref`, own with `var` to avoid accidental copies.

Performance
- [ ] Vectorize with `SIMD[dtype, size]`; choose size from `sys.simd_width_of`.
- [ ] Use `algorithm.elementwise`/`vectorize`/`parallelize` for data-parallel CPU work.
- [ ] `@always_inline` tiny hot functions; parameterize over `DType`/sizes for specialization.
- [ ] Make memory layout explicit with `LayoutTensor`/`Layout.row_major`; tile for locality.
- [ ] Benchmark with `benchmark.run[...]` and guard results with `keep(...)`.

GPU
- [ ] Bounds-check `if tid < n:` in every kernel.
- [ ] Drive allocation/launch/sync through `DeviceContext`; `enqueue_*` then `synchronize()`.
- [ ] Coalesce global-memory accesses; profile bandwidth, distrust high cache-hit rates.
- [ ] Stage tiles in shared memory; `barrier()` only block-wide (never in divergent branches).
- [ ] Prefer warp ops within a warp; use `gpu.WARP_SIZE`, not 32.
- [ ] Target sufficient occupancy (25-50%), not maximum; sweep block sizes and watch register/shared pressure.
- [ ] Treat `thread_idx`/`block_idx`/etc. as `Int`.

Version hygiene
- [ ] Target Mojo 1.0.x; pin via `pixi`. Re-verify any stdlib API outside stabilized-API markers against the installed version.
- [ ] Use `TestSuite` + `mojo run` for tests (the `mojo test` command is gone).
- [ ] Import only what you need; `from pkg import x` does not bind `pkg`.

---

## Further Reading

| Resource | Type | Why recommended |
|---|---|---|
| [Mojo Manual](https://mojolang.org/docs/manual/) | Official docs | Canonical, current language reference. |
| [Mojo releases / changelog](https://mojolang.org/releases/) | Official changelog | Authoritative version-by-version breaking changes. |
| [Mojo v1.0.0b1 release notes](https://mojolang.org/releases/v1.0.0b1/) | Official changelog | The 1.0 beta migration list (fn->def, conventions, NDBuffer removal). |
| [Ownership](https://mojolang.org/docs/manual/values/ownership/) | Official docs | Current argument conventions (`read`/`mut`/`var`/`ref`). |
| [Value lifecycle](https://mojolang.org/docs/manual/lifecycle/) | Official docs | Constructors/destructors and synthesized copy/move. |
| [Parameterization](https://mojolang.org/docs/manual/parameters/) | Official docs | Compile-time metaprogramming, `comptime`, inference. |
| [Functions](https://mojolang.org/docs/manual/functions/) | Official docs | `def`, `raises`, conventions, named results. |
| [Errors and context managers](https://mojolang.org/docs/manual/errors/) | Official docs | Typed errors, `Never`, parametric raises. |
| [Traits](https://mojolang.org/docs/manual/traits/) | Official docs | Composition model, built-in traits. |
| [SIMD](https://mojolang.org/docs/stdlib/builtin/simd/) | Official stdlib docs | Vector type, ops, reductions, casting. |
| [GPU fundamentals](https://mojolang.org/docs/manual/gpu/fundamentals/) | Official docs | Execution model, DeviceContext, indexing. |
| [GPU intro tutorial](https://mojolang.org/docs/manual/gpu/intro-tutorial/) | Official tutorial | End-to-end kernel with TileTensor. |
| [GPU block and warp ops](https://mojolang.org/docs/manual/gpu/block-and-warp/) | Official docs | barrier(), shuffles, warp/block reductions. |
| [Using LayoutTensor](https://mojolang.org/docs/manual/layout/tensors/) | Official docs | Layouts, tiling, vectorize, distribute. |
| [Mojo GPU Puzzles](https://puzzles.modular.com/) | Official interactive | Hands-on coalescing, occupancy, bank conflicts. |
| [modular/modular](https://github.com/modular/modular) | Source | Stdlib + MAX kernels; ground truth for APIs. |
| [The path to Mojo 1.0](https://www.modular.com/blog/the-path-to-mojo-1-0) | Official blog | Stability guarantees and post-1.0 plans. |
| [Testing](https://mojolang.org/docs/tools/testing/) | Official docs | TestSuite-based testing. |
| [Calling Python from Mojo](https://mojolang.org/docs/manual/python/python-from-mojo/) | Official docs | Interop patterns and constraints. |

---

This guide was synthesized from 41 sources, weighted toward official Modular / mojolang.org documentation current to Mojo v1.0.0b1 (2026-05-07). See `resources/mojo-best-practices-performance-gpu-sources.json` for the full source list with confidence and recency ratings. Because Mojo is pre-1.0-stable in places, re-verify any stdlib API outside stabilized-API markers against your installed version.
