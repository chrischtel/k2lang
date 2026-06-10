# K2 Compiler

K2 is an experimental systems programming language written in Zig. It targets
low-level explicitness — no hidden allocations, explicit dynamic dispatch,
lexical ownership zones — while providing generics, fallible functions, and a
growing standard library.

The compiler and language are still evolving. K2 is not ready for production
use, and some accepted constructs do not yet have complete backend semantics.

## Design Principles

- No hidden allocations.
- Dynamic dispatch is explicit through `*Interface` values.
- Static polymorphism uses monomorphized generics.
- Allocation zones give lexical ownership without a garbage collector.
- Debug builds trap common undefined behavior close to its source.
- Features should have complete semantics before more syntax is added.

## Building

The frontend builds and tests without LLVM:

```powershell
zig build
zig build test
```

LLVM code generation requires an LLVM installation (tested against
`Y:/SDK/LLVM`):

```powershell
zig build -Dllvm-path=Y:/SDK/LLVM
zig build test -Dllvm-path=Y:/SDK/LLVM
```

The standard-library root defaults to the sibling `k2-modules` directory:

```powershell
zig build -Dstdlib-root=Y:/path/to/k2-modules
```

Compiler commands:

```text
k2 check <file>     Parse and type-check a source file
k2 ir <file>        Print K2 IR
k2 object <file>    Emit an object file
k2 build <file>     Build an executable (Windows)
```

## Language Example

```k2
#import std.io.{ Writer, Stdout, println };

Greeting :: struct {
    name: []const u8,
}

Greeting as Writer {
    write :: fn(self: *Self, data: []const u8) -> usize ! IoError {
        return write_stdout(data);
    }
    flush :: fn(self: *Self) -> void ! IoError {}
}

main :: fn() -> i32 {
    g := Greeting { name = "world" };
    println("hello");

    zone scratch: Arena {
        buf := scratch.new_slice(u8, 64);
        buf[0] = 'h';
    }

    return 0;
}
```

A more complete example lives in `k2son/` — a JSON serializer written in K2
itself, using interfaces, fallible functions, zones, and several stdlib modules.

## Implementation Status

| Area | Status | Notes |
| --- | --- | --- |
| Lexer, parser, AST, diagnostics | Implemented | Source spans; diagnostic tests. |
| Semantic analysis and typed IR | Implemented | IR validation and optimization passes (const-fold, branch, DCE). |
| LLVM object/executable generation | Implemented | Core lowering, module verification, debug checks, Windows executable integration. |
| Structs, packed structs, enums | Implemented | Enums support payloads and pattern matching. |
| Integer types | Implemented | i8–i128, u1–u128, sub-byte types in packed structs. |
| Float types | Implemented | f32 and f64; arithmetic and comparisons lower to correct LLVM FP instructions. |
| Pointers, arrays, slices, optionals | Implemented | Debug builds check null dereferences, bounds, and invalid unwraps. |
| Type casts | Implemented | Integer widening/narrowing (signed and unsigned), float/int conversions, and casts to underlying types. |
| Distinct types | Implemented | Newtype wrappers over any type; casts to/from underlying type work; IR correctly lowers to the underlying representation in all positions (params, returns, locals, struct fields, generics). |
| Opaque types | Implemented | `Foo :: opaque;` declares a forward-declared type; used only as `*Foo`; lowers to an opaque pointer (`ptr`). |
| Atomic types | Partial | `atomic T` field qualifier strips to `T` in struct layout. `atomic_load` and `atomic_store` builtins work with acquire/release ordering. `compare_exchange` and fetch-add/sub are not yet implemented. |
| Functions and generics | Implemented | Generic functions and structs are monomorphized. |
| Control flow | Implemented | `if`, `while`, range/slice `for`, `break`, `continue`, `defer`. |
| Integer and enum `match` | Implemented | Single and grouped cases; exhaustiveness checked. |
| Compile-time execution | Partial | `#if` and `#run` evaluate inside functions; `sizeof` is accurate. Float/uint ops, array indexing, and `for` loops work at comptime. Struct construction and reflection are still incomplete. |
| Errors and fallible functions | Implemented | `-> T!E` ABI, `fail`, `?` propagation, `catch`, and fallible entry points lower correctly. |
| Zones | Implemented | `Arena` provides zero-initialized lexical allocation, non-escape checking, and deterministic cleanup. |
| Interfaces | Partial | Dynamic `*Interface` dispatch works end to end; fallible methods and `.ok`/`.err` access work. Static constraints, const correctness, generic methods, composition, and default methods are missing. **Known bug:** a method dispatched through `*InterfaceA` that itself takes a `*InterfaceB` argument and calls a method on it causes an LLVM verification error. Avoid nesting interface calls until this is fixed. |
| Runtime | Implemented on Windows; source-valid on Linux | Embedded runtime: output, exit/abort, panic, assertions. Windows linking is end-to-end; Linux native entry-point and macOS are incomplete. |
| Modules, imports, visibility | Implemented | Private by default; `pub` exports; selective imports; import-scoped extension methods. |
| Standard library | Partial | `std.io`, `std.fmt`, `std.mem`, `std.fs`, `std.process`, `std.ptr`, `std.bits` are implemented. Package namespaces, manifests, and most other modules are still needed. |
| Attributes | Partial | `#extern #packed #inline #noinline #noreturn #naked #entry #export #deprecated #align(N)` implemented. `#require #ensure #link #section #callconv #test` are not. |
| Tooling | Partial | Check, IR, object, and build commands. No formatter, LSP, package manager, or test runner. |

## Standard Library

The standard library lives in the sibling `k2-modules/std` directory.

| Module | What it provides |
| --- | --- |
| `std.io` | `Writer`/`Reader` interfaces; `Stdout`, `Stderr`, `FixedBuf`, `NullWriter`; numeric formatters; convenience `print`/`println`. |
| `std.fmt` | Width-justified output, left/right padding, integer columns, joined slices — built on top of `*Writer`. |
| `std.mem` | Typed-slice helpers: `eql`, `copy`, `fill`, `index_of`, `contains`, byte-level variants. |
| `std.fs` | `File` implementing `Reader` + `Writer`; `open`, `create`, `append`, `delete`, `exists` (Windows, ANSI paths). |
| `std.process` | PID, raw command line, env-var access, child process spawn/wait/kill (Windows). |
| `std.ptr` | Pointer/address conversions and alignment arithmetic (`to_addr`, `from_addr`, `add_bytes`, `is_aligned`). |
| `std.bits` | Bit-twiddling for u32/u64: population count, leading/trailing zeros, rotate, power-of-two test. |

Example:

```k2
#import std.io.{ println, print_u64 };
#import std.mem.{ copy, eql_bytes };
#import std.fmt.{ write_padded_left };

main :: fn() -> i32 {
    println("hello from K2");
    print_u64(42u64);
    return 0;
}
```

## Interfaces

Interfaces define required methods. Any type can implement them:

```k2
Writer :: interface {
    write :: fn(*Self, []const u8) -> usize ! IoError;
    flush :: fn(*Self) -> void ! IoError;
}

FileHandle :: struct { fd: i32 }

FileHandle as Writer {
    write :: fn(self: *FileHandle, data: []const u8) -> usize ! IoError {
        n := sys_write(self.fd, data);
        if n == 0usize { fail .write_failed; }
        return n;
    }
    flush :: fn(self: *FileHandle) -> void ! IoError {}
}

main :: fn() {
    file := FileHandle { fd = 1 };
    w: *Writer = &file;   // conformance checked at compile time
    w.write("hello\n") catch err {};
}
```

`*Writer` holds a data pointer and a generated vtable. Coercions are checked
at compile time; method calls go through indirect dispatch.

**What works:** `*Interface` coercions in assignments, arguments, and returns;
vtable generation; fallible interface methods; `.ok`/`.err` field access.

**What does not work yet:**
- Static constraints (`where T: Writer` style generics)
- `*InterfaceA` method calling a `*InterfaceB` argument — triggers an LLVM
  verification error (known bug, under investigation)
- Const-correct interface values
- Generic interface methods
- Interface composition, inheritance, owned interface objects

## Zones

`zone name: Arena { ... }` creates a lexical allocation arena:

```k2
fill :: fn(data: borrow []u8) {
    data[0] = 42u8;
}

work :: fn() {
    zone scratch: Arena {
        data := scratch.new_slice(u8, 4);
        fill(data);
        // arena freed here
    }
}
```

Key rules:
- Zone-owned values may not outlive their zone (compiler-enforced).
- `borrow *T` / `borrow []T` parameters may temporarily receive zone-owned
  values but cannot store, return, or forward them to ordinary parameters.
- `defer`, `return`, `fail`, `break`, and `continue` all trigger cleanup.
- `Arena.free(value)` validates ownership but defers reclamation to zone exit.
- The current backend uses the Windows process heap.

## Safety And Undefined Behavior

Debug builds trap common invalid operations through the shared runtime panic
path, with `file:line:column` reported.

| Operation | Debug behavior |
| --- | --- |
| Integer overflow | Traps signed/unsigned add, subtract, multiply, negate, signed division overflow. |
| Division by zero | Traps integer division and remainder by zero. |
| Invalid shifts | Traps shift amounts ≥ operand width. |
| Out-of-bounds indexing | Guards slice and array indexing. |
| Null pointer dereference | Guards dereference, pointer field access, and pointer indexing. |
| Invalid optional unwrap `!!` | Always panics through the runtime. |
| Use after zone cleanup | Safe code rejects escaping zone values; unsafe pointer misuse is caller responsibility. |
| Data races | Caller responsibility. |

Release-mode behavior and explicit wrapping arithmetic (`+%`, `-%`, `*%`) need
a final language-level decision before implementation.

## Modules And Visibility

Every source file is a module. Declarations are private unless marked `pub`:

```k2
pub add :: fn(a: i32, b: i32) -> i32 { return a + b; }
helper :: fn() {}  // private
```

Selective imports are preferred:

```k2
#import math.{ add, multiply };
#import std.io.{ println };
```

Any visible top-level function whose first parameter is named `self` is also
reachable through dot-call syntax in importing files:

```k2
// math.k2
pub doubled :: fn(self: i32) -> i32 { return self * 2; }

// main.k2
#import math.{ doubled };
x := 21.doubled();   // same as doubled(21)
```

Extension methods do not access private fields and are scoped to the importing
file. Generic extension methods currently require explicit type arguments
(`values.fill(i32, 0)`).

**Current limitation:** top-level names must be unique across the whole
compilation. Package namespaces and symbol mangling must be added before
third-party packages can coexist.

## Runtime

Real program builds embed one runtime automatically:

```text
write_stdout(data: []const u8) -> usize
write_stderr(data: []const u8) -> usize
exit(code: u32)
abort()
@panic(msg: []const u8)
assert(cond: bool)
assert_msg(cond: bool, msg: []const u8)
```

`@panic` writes a prefixed message to stderr and terminates. Compiler-generated
safety traps use the same path. Windows is end-to-end; Linux source compiles
and checks but native entry-point generation and linking are incomplete; macOS
has no runtime yet.

## Roadmap

### Now: Fix What Blocks Real Programs

- **Interface-through-interface dispatch** — a method on `*InterfaceA` that
  calls a method on a `*InterfaceB` argument causes an LLVM verification error.
  This is the top compiler bug and is blocking `std.fmt.Display`.
- **Symbol mangling / package namespaces** — top-level names must be globally
  unique, which prevents multi-package projects and third-party code.
- **Wrapping arithmetic** — decide and implement `+%`/`-%`/`*%` and the
  release-build overflow policy.

### Soon: Complete The Language

- Static interface constraints (`where T: Writer`-style).
- Const-correct interface values and receiver mutability rules.
- `#test` runner and test attribute.
- `#callconv`, `#link`, `#section` attributes.
- Linux native entry-point generation and linking; macOS runtime.
- Continue bootstrapping `std.*` (string slicing, sorting, hash maps).

### Later: Tooling And Ergonomics

- Formatter.
- Language server (LSP).
- Package manifest and dependency management.
- Improved diagnostics for generic instantiation and interface conformance.

### Future Language Expansion (After Stable Foundation)

- Runtime type identity, reflection, `Any`.
- SIMD and vector operations.
- Contracts (`#require`, `#ensure`).
- Interface composition, owned dynamic objects, downcasting.
- User-defined attributes.

## Open Language Decisions

| Decision | Question |
| --- | --- |
| Plain arithmetic | Does overflow trap, wrap, or become UB in release builds? |
| Debug safety | Which checks are mandatory vs. disabled inside `unsafe`? |
| Error ABI | How are fallible returns propagated across modules and FFI? |
| Borrow scope | Should `borrow` expand to fields, returns, or extern contracts? |
| Static constraints | What is the `where T: Trait` syntax? Coherence rules? |
| Mutability | How do `const`, receiver mutability, pointers, and interface coercions interact? |
| `Any` / reflection | Borrowed or owned? What defines stable runtime type identity? |
| Packages | How are packages named, versioned, and represented in symbols? |

## Testing

```powershell
zig build test                              # frontend and IR tests
zig build test -Dllvm-path=Y:/SDK/LLVM     # full suite including LLVM lowering
```

New features should include parser, semantic, IR, and LLVM tests where
applicable. Safety work should include executable tests that verify the
expected debug trap fires.
