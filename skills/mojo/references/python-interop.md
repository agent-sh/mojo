# Python interop

Read this when code crosses the Python boundary. Checked against the Mojo 1.1 manual ("Calling Mojo from Python", "Python integration") and the 1.0 and 1.1 changelogs.

## Python from Mojo

- `from std.python import Python, PythonObject`; `Python.import_module("numpy")`, `Python.evaluate("...")` for expressions, including a Python `lambda` when you need a Python callable.
- Converting a `PythonObject` to a Mojo value uses the `py=` keyword: `Int(py=obj)`, `Float64(py=obj)`, `String(py=obj)`, `Bool(py=obj)`.
- `Python.dict(...)` infers one value type for all keyword arguments; wrap mixed values in `PythonObject(...)`.
- Every `PythonObject` operation goes through CPython. 1.0 made operators about 12x faster, but keep hot loops in Mojo and convert once at the boundary.
- NumPy: `std.python.numpy` has `copy_to_numpy_array()` / `from_numpy_array()` (1-D) and, in 1.1, `copy_to_numpy_tensor()` / `from_numpy_tensor()` for N-D C-contiguous arrays (`from_numpy_*` borrow the buffer without copying).

## Mojo from Python (extension modules)

```mojo
from std.python import PythonObject
from std.python.bindings import PythonModuleBuilder
from std.os import abort

@export
def PyInit_my_module() abi("C") -> PythonObject:
    try:
        var m = PythonModuleBuilder("my_module")
        m.def_function[factorial]("factorial", docstring="Compute n!")
        return m.finalize()
    except e:
        abort(String("error creating module:", e))
```

- The function is `PyInit_<name>` where `<name>` matches the `.mojo` file, exported with `abi("C")`. An `abi("C")` function cannot raise, so catch and `abort` inside it. Registered functions (`def_function`, `def_method`, `def_py_init`) need no `@export` and may raise; raised errors become Python exceptions.
- Types: `mb.add_type[T]("T")` returns a `PythonTypeBuilder` for `def_py_init`, `def_method`, `def_staticmethod`. Bound types must be `Writable`; custom initializers need `Movable`. Methods take `py_self: PythonObject` or a typed `self_ptr: Pointer[mut=True, Self]`. Wrap a Mojo value for Python with `PythonObject(alloc=value^)` and get it back with `obj.downcast_value_ptr[T]()`. 1.1 removed the fixed limit on positional arguments.
- Build with `mojo build --emit shared-lib`, or `import mojo.importer` in Python to compile `.mojo` files on import (cached in `__mojocache__/`, or the Modular cache dir when the source dir is read-only). The shared library must not define `main()`.
- A Mojo shared library called from C or C++ (no Mojo `main`) must call `std.runtime.initialize_runtime()` before using `parallelize` or other runtime APIs.
