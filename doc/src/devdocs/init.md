# Initialization of the Julia runtime

How does the Julia runtime execute `julia -e 'println("Hello World!")'` ?

## `main()`

Execution starts at [`main()` in `cli/loader_exe.c`](https://github.com/JuliaLang/julia/blob/master/cli/loader_exe.c),
which calls `jl_load_repl()` in [`cli/loader_lib.c`](https://github.com/JuliaLang/julia/blob/master/cli/loader_lib.c)
which loads a few libraries, eventually calling [`jl_repl_entrypoint()` in `src/jlapi.c`](https://github.com/JuliaLang/julia/blob/master/src/jlapi.c).

`jl_repl_entrypoint()` calls [`libsupport_init()`](https://github.com/JuliaLang/julia/blob/master/src/support/libsupportinit.c)
to set the C library locale and to initialize the "ios" library (see [`ios_init_stdstreams()`](https://github.com/JuliaLang/julia/blob/master/src/support/ios.c)
and [Legacy `ios.c` library](@ref Legacy-ios.c-library)).

Next [`jl_parse_opts()`](https://github.com/JuliaLang/julia/blob/master/src/jloptions.c) is called to process
command line options. Note that `jl_parse_opts()` only deals with options that affect code generation
or early initialization. Other options are handled later by [`exec_options()` in `base/client.jl`](https://github.com/JuliaLang/julia/blob/master/base/client.jl).

`jl_parse_opts()` stores command line options in the [global `jl_options` struct](https://github.com/JuliaLang/julia/blob/master/src/julia.h).

## `julia_init()`

[`julia_init()` in `init.c`](https://github.com/JuliaLang/julia/blob/master/src/init.c) is called
by `main()` and calls [`_julia_init()` in `init.c`](https://github.com/JuliaLang/julia/blob/master/src/init.c).

`_julia_init()` begins by calling `libsupport_init()` again (it does nothing the second time).

[`restore_signals()`](https://github.com/JuliaLang/julia/blob/master/src/signals-unix.c) is called
to zero the signal handler mask.

[`jl_resolve_sysimg_location()`](https://github.com/JuliaLang/julia/blob/master/src/init.c) searches
configured paths for the base system image. See [Building the Julia system image](@ref Building-the-Julia-system-image).

[`jl_gc_init()`](https://github.com/JuliaLang/julia/blob/master/src/gc.c) sets up allocation pools
and lists for weak refs, preserved values and finalization.

[`jl_init_frontend()`](https://github.com/JuliaLang/julia/blob/master/src/ast.c) loads and initializes
a pre-compiled femtolisp image containing the scanner/parser.

[`jl_init_types()`](https://github.com/JuliaLang/julia/blob/master/src/jltypes.c) creates `jl_datatype_t`
type description objects for the [built-in types defined in `julia.h`](https://github.com/JuliaLang/julia/blob/master/src/julia.h).
e.g.

```c
jl_any_type = jl_new_abstracttype(jl_symbol("Any"), core, NULL, jl_emptysvec);
jl_any_type->super = jl_any_type;

jl_type_type = jl_new_abstracttype(jl_symbol("Type"), core, jl_any_type, jl_emptysvec);

jl_int32_type = jl_new_primitivetype(jl_symbol("Int32"), core,
                                     jl_any_type, jl_emptysvec, 32);
```

[`jl_init_tasks()`](https://github.com/JuliaLang/julia/blob/master/src/task.c) creates the `jl_datatype_t* jl_task_type`
object; initializes the global `jl_root_task` struct; and sets `jl_current_task` to the root task.

[`jl_init_codegen()`](https://github.com/JuliaLang/julia/blob/master/src/codegen.cpp) initializes
the [LLVM library](https://llvm.org).

[`jl_init_serializer()`](https://github.com/JuliaLang/julia/blob/master/src/staticdata.c) initializes
8-bit serialization tags for builtin `jl_value_t` values.

If there is no sysimg file (`!jl_options.image_file`) then the `Core` and `Main` modules are
created and `boot.jl` is evaluated:

`jl_core_module = jl_new_module(jl_symbol("Core"), NULL)` creates the Julia `Core` module.

[`jl_init_intrinsic_functions()`](https://github.com/JuliaLang/julia/blob/master/src/intrinsics.cpp)
creates a new Julia module `Intrinsics` containing constant `jl_intrinsic_type` symbols. These define
an integer code for each [intrinsic function](https://github.com/JuliaLang/julia/blob/master/src/intrinsics.cpp).
[`emit_intrinsic()`](https://github.com/JuliaLang/julia/blob/master/src/intrinsics.cpp) translates
these symbols into LLVM instructions during code generation.

[`jl_init_primitives()`](https://github.com/JuliaLang/julia/blob/master/src/builtins.c) hooks C
functions up to Julia function symbols. e.g. the symbol `Core.:(===)()` is bound to C function pointer
`jl_f_is()` by calling `add_builtin_func("===", jl_f_is)`.

[`jl_new_main_module()`](https://github.com/JuliaLang/julia/blob/master/src/toplevel.c) creates
the global "Main" module and sets `jl_current_task->current_module = jl_main_module`.

Note: `_julia_init()` [then sets](https://github.com/JuliaLang/julia/blob/master/src/init.c) `jl_root_task->current_module = jl_core_module`.
`jl_root_task` is an alias of `jl_current_task` at this point, so the `current_module` set by `jl_new_main_module()`
above is overwritten.

[`jl_load("boot.jl", sizeof("boot.jl"))`](https://github.com/JuliaLang/julia/blob/master/src/init.c)
calls [`jl_parse_eval_all`](https://github.com/JuliaLang/julia/blob/master/src/ast.c) which repeatedly
calls [`jl_toplevel_eval_flex()`](https://github.com/JuliaLang/julia/blob/master/src/toplevel.c)
to execute [`boot.jl`](https://github.com/JuliaLang/julia/blob/master/base/boot.jl). <!-- TODO – drill
down into eval? -->

[`jl_get_builtin_hooks()`](https://github.com/JuliaLang/julia/blob/master/src/init.c) initializes
global C pointers to Julia globals defined in `boot.jl`.

[`jl_init_box_caches()`](https://github.com/JuliaLang/julia/blob/master/src/datatype.c) pre-allocates
global boxed integer value objects for values up to 1024. This speeds up allocation of boxed ints
later on. e.g.:

```c
jl_value_t *jl_box_uint8(uint32_t x)
{
    return boxed_uint8_cache[(uint8_t)x];
}
```

[`_julia_init()` iterates](https://github.com/JuliaLang/julia/blob/master/src/init.c) over the
`jl_core_module->bindings.table` looking for `jl_datatype_t` values and sets the type name's module
prefix to `jl_core_module`.

[`jl_add_standard_imports(jl_main_module)`](https://github.com/JuliaLang/julia/blob/master/src/toplevel.c)
does "using Base" in the "Main" module.

Note: `_julia_init()` now reverts to `jl_root_task->current_module = jl_main_module` as it was
before being set to `jl_core_module` above.

Platform specific signal handlers are initialized for `SIGSEGV` (OSX, Linux), and `SIGFPE` (Windows).

Other signals (`SIGINFO, SIGBUS, SIGILL, SIGTERM, SIGABRT, SIGQUIT, SIGSYS` and `SIGPIPE`) are
hooked up to [`sigdie_handler()`](https://github.com/JuliaLang/julia/blob/master/src/signals-unix.c)
which prints a backtrace.

[`jl_init_restored_module()`](https://github.com/JuliaLang/julia/blob/master/src/staticdata.c) calls
[`jl_module_run_initializer()`](https://github.com/JuliaLang/julia/blob/master/src/module.c) for
each deserialized module to run the `__init__()` function.

Finally [`sigint_handler()`](https://github.com/JuliaLang/julia/blob/master/src/signals-unix.c)
is hooked up to `SIGINT` and calls `jl_throw(jl_interrupt_exception)`.

`_julia_init()` then returns [back to `main()` in `cli/loader_exe.c`](https://github.com/JuliaLang/julia/blob/master/cli/loader_exe.c)
and `main()` calls `repl_entrypoint(argc, (char**)argv)`.

!!! sidebar "sysimg"
    If there is a sysimg file, it contains a pre-cooked image of the `Core` and `Main` modules (and
    whatever else is created by `boot.jl`). See [Building the Julia system image](@ref Building-the-Julia-system-image).

    [`jl_restore_system_image()`](https://github.com/JuliaLang/julia/blob/master/src/staticdata.c) deserializes
    the saved sysimg into the current Julia runtime environment and initialization continues after
    `jl_init_box_caches()` below...

    Note: [`jl_restore_system_image()` (and `staticdata.c` in general)](https://github.com/JuliaLang/julia/blob/master/src/staticdata.c)
    uses the [Legacy `ios.c` library](@ref Legacy-ios.c-library).

## `repl_entrypoint()`

[`repl_entrypoint()`](https://github.com/JuliaLang/julia/blob/master/src/jlapi.c) loads the contents of
`argv[]` into [`Base.ARGS`](@ref).

If a `.jl` "program" file was supplied on the command line, then [`exec_program()`](https://github.com/JuliaLang/julia/blob/master/src/jlapi.c)
calls [`jl_load(program,len)`](https://github.com/JuliaLang/julia/blob/master/src/toplevel.c) which
calls [`jl_parse_eval_all`](https://github.com/JuliaLang/julia/blob/master/src/ast.c) which repeatedly
calls [`jl_toplevel_eval_flex()`](https://github.com/JuliaLang/julia/blob/master/src/toplevel.c)
to execute the program.

However, in our example (`julia -e 'println("Hello World!")'`), [`jl_get_global(jl_base_module, jl_symbol("_start"))`](https://github.com/JuliaLang/julia/blob/master/src/module.c)
looks up [`Base._start`](https://github.com/JuliaLang/julia/blob/master/base/client.jl) and [`jl_apply()`](https://github.com/JuliaLang/julia/blob/master/src/julia.h)
executes it.

## `Base._start`

[`Base._start`](https://github.com/JuliaLang/julia/blob/master/base/client.jl) calls [`Base.exec_options`](https://github.com/JuliaLang/julia/blob/master/base/client.jl)
which calls [`jl_parse_input_line("println("Hello World!")")`](https://github.com/JuliaLang/julia/blob/master/src/ast.c)
to create an expression object and [`Core.eval(Main, ex)`](@ref Core.eval) to execute the parsed expression `ex` in the module context of `Main`.

## `Core.eval`

[`Core.eval(Main, ex)`](@ref Core.eval) calls [`jl_toplevel_eval_in(m, ex)`](https://github.com/JuliaLang/julia/blob/master/src/toplevel.c),
which calls [`jl_toplevel_eval_flex`](https://github.com/JuliaLang/julia/blob/master/src/toplevel.c).
`jl_toplevel_eval_flex` implements a simple heuristic to decide whether to compile a given code thunk or run it by interpreter.
When given `println("Hello World!")`, it would usually decide to run the code by interpreter, in which case it calls
[`jl_interpret_toplevel_thunk`](https://github.com/JuliaLang/julia/blob/master/src/interpreter.c), which then calls
[`eval_body`](https://github.com/JuliaLang/julia/blob/master/src/interpreter.c).

The stack dump below shows how the interpreter works its way through various methods of [`Base.println()`](@ref)
and [`Base.print()`](@ref) before arriving at [`write(s::IO, a::Array{T}) where T`](https://github.com/JuliaLang/julia/blob/master/base/stream.jl)
 which does `ccall(jl_uv_write())`.

[`jl_uv_write()`](https://github.com/JuliaLang/julia/blob/master/src/jl_uv.c) calls `uv_write()`
to write "Hello World!" to `JL_STDOUT`. See [Libuv wrappers for stdio](@ref Libuv-wrappers-for-stdio).:

```
Hello World!
```

| Stack frame                    | Source code     | Notes                                                |
|:------------------------------ |:--------------- |:---------------------------------------------------- |
| `jl_uv_write()`                | `jl_uv.c`       | called though [`ccall`](@ref)                        |
| `julia_write_282942`           | `stream.jl`     | function `write!(s::IO, a::Array{T}) where T`        |
| `julia_print_284639`           | `ascii.jl`      | `print(io::IO, s::String) = (write(io, s); nothing)` |
| `jlcall_print_284639`          |                 |                                                      |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_trampoline()`              | `builtins.c`    |                                                      |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_apply_generic()`           | `gf.c`          | `Base.print(Base.TTY, String)`                       |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_trampoline()`              | `builtins.c`    |                                                      |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_apply_generic()`           | `gf.c`          | `Base.print(Base.TTY, String, Char, Char...)`        |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_f_apply()`                 | `builtins.c`    |                                                      |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_trampoline()`              | `builtins.c`    |                                                      |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_apply_generic()`           | `gf.c`          | `Base.println(Base.TTY, String, String...)`          |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_trampoline()`              | `builtins.c`    |                                                      |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `jl_apply_generic()`           | `gf.c`          | `Base.println(String,)`                              |
| `jl_apply()`                   | `julia.h`       |                                                      |
| `do_call()`                    | `interpreter.c` |                                                      |
| `eval_body()`                  | `interpreter.c` |                                                      |
| `jl_interpret_toplevel_thunk`  | `interpreter.c` |                                                      |
| `jl_toplevel_eval_flex`        | `toplevel.c`    |                                                      |
| `jl_toplevel_eval_in`          | `toplevel.c`    |                                                      |
| `Core.eval`                    | `boot.jl`       |                                                      |

Since our example has just one function call, which has done its job of printing "Hello World!",
the stack now rapidly unwinds back to `main()`.

## `jl_atexit_hook()`

`main()` calls [`jl_atexit_hook()`](https://github.com/JuliaLang/julia/blob/master/src/init.c).
This calls `Base._atexit`, then calls [`jl_gc_run_all_finalizers()`](https://github.com/JuliaLang/julia/blob/master/src/gc.c)
and cleans up libuv handles.

## `julia_save()`

Finally, `main()` calls [`julia_save()`](https://github.com/JuliaLang/julia/blob/master/src/init.c),
which if requested on the command line, saves the runtime state to a new system image. See [`jl_compile_all()`](https://github.com/JuliaLang/julia/blob/master/src/gf.c)
and [`jl_save_system_image()`](https://github.com/JuliaLang/julia/blob/master/src/staticdata.c).
