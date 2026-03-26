# Zig 0.16 Language Context for LLMs

> **Purpose**: This document provides comprehensive, up-to-date context on the Zig programming language as of **Zig 0.16.0-dev** (March 2026). Most LLMs were trained on Zig 0.11–0.13 era code, which is now significantly outdated. Use this document to avoid generating broken or deprecated Zig code.

> **Sources**: Ziglings exercises (targeting 0.16.0-dev.2915), Zig source repository on Codeberg, ziglang.org, zig.guide. Last updated: 2026-03-26.

---

## Table of Contents

1. [Critical Breaking Changes (READ FIRST)](#1-critical-breaking-changes-read-first)
2. [Basic Syntax](#2-basic-syntax)
3. [Arrays and Strings](#3-arrays-and-strings)
4. [Control Flow](#4-control-flow)
5. [Functions](#5-functions)
6. [Error Handling](#6-error-handling)
7. [Pointers and Memory](#7-pointers-and-memory)
8. [Slices](#8-slices)
9. [Structs](#9-structs)
10. [Enums](#10-enums)
11. [Unions and Interfaces](#11-unions-and-interfaces)
12. [Optionals](#12-optionals)
13. [Type Coercion](#13-type-coercion)
14. [Comptime and Metaprogramming](#14-comptime-and-metaprogramming)
15. [Builtin Functions](#15-builtin-functions)
16. [Sentinels](#16-sentinels)
17. [Vectors / SIMD](#17-vectors--simd)
18. [Standard Library](#18-standard-library)
19. [The New std.Io Interface (0.15/0.16)](#19-the-new-stdio-interface-01516)
20. [Build System](#20-build-system)
21. [C Interoperability](#21-c-interoperability)
22. [Threading and Structured Concurrency](#22-threading-and-structured-concurrency)
23. [Common Pitfalls for LLMs](#23-common-pitfalls-for-llms)
24. [How to Update This Document](#24-how-to-update-this-document)

---

## 1. Critical Breaking Changes (READ FIRST)

These are the changes most likely to cause LLMs to generate broken code. **Read this section before generating any Zig code.**

### 1.1 The `std.Io` Interface (0.15/0.16 — MASSIVE CHANGE)

The entire I/O subsystem was redesigned. Old `std.io` and `std.fs` patterns **will not compile** in 0.16.

**Old (BROKEN in 0.16):**
```zig
pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello!\n", .{});
}
```

**New (0.16):**
```zig
pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var stdout_writer = std.Io.File.stdout().writer(io, &.{});
    const stdout = &stdout_writer.interface;
    try stdout.print("Hello!\n", .{});
}
```

**Exception**: `std.debug.print()` still works without any `io` parameter and `pub fn main() void` or `pub fn main() !void` is still valid for programs that only use debug printing.

### 1.2 Builtin Renames (`@xToY` → `@yFromX`)

All conversion builtins were renamed (0.11+). The old names **do not exist**.

| Old (BROKEN) | New |
|---|---|
| `@enumToInt(e)` | `@intFromEnum(e)` |
| `@intToEnum(T, i)` | `@enumFromInt(i)` — type inferred |
| `@intToFloat(T, i)` | `@floatFromInt(i)` — type inferred |
| `@floatToInt(T, f)` | `@intFromFloat(f)` — type inferred |
| `@ptrToInt(p)` | `@intFromPtr(p)` |
| `@intToPtr(T, i)` | `@ptrFromInt(i)` — type inferred |
| `@errSetCast(T, e)` | `@errorCast(e)` — type inferred |
| `@boolToInt(b)` | `@intFromBool(b)` |

### 1.3 Single-Argument Builtin Casts

Many builtins lost their explicit type parameter. The destination type is **inferred from context**.

**Old (BROKEN):**
```zig
const x = @intCast(u8, some_u32);
const y = @truncate(u16, some_u64);
const z = @ptrCast(*u8, some_ptr);
const w = @bitReverse(u8, val);
```

**New:**
```zig
const x: u8 = @intCast(some_u32);
const y: u16 = @truncate(some_u64);
const z: *u8 = @ptrCast(some_ptr);
const w = @bitReverse(val);
```

### 1.4 `@typeInfo` Field Access Uses Quoted Identifiers

**Old (BROKEN):**
```zig
const fields = @typeInfo(MyStruct).Struct.fields;
```

**New:**
```zig
const fields = @typeInfo(MyStruct).@"struct".fields;
```

Union tag names in `@typeInfo` are now lowercase and since `struct`, `enum`, `union`, `fn` etc. are keywords, they require the `@"..."` quoted identifier syntax.

### 1.5 Async/Await Restored in 0.16

Async/await was removed after 0.11 and the keywords were removed in 0.15, but **async/await has been restored in 0.16** with a completely new implementation backed by `std.Io`. This is NOT the old 0.11-era frame-based `suspend`/`resume`/`nosuspend` system — those are still gone.

The new async works with the `std.Io` interface, which is vtable-based and dispatches to platform-specific backends (Grand Central Dispatch on Apple, io_uring on Linux, kqueue on BSDs).

**Two tiers of concurrency:**

```zig
// io.async — dispatch work, may run inline (eager) or on fiber/thread pool
var future = io.async(myFunction, .{ arg1, arg2 });
const result = future.await(io);

// io.concurrent — MUST use actual parallelism (guaranteed thread pool thread)
// Can fail with ConcurrencyUnavailable
var future2 = try io.concurrent(myFunction, .{ arg1, arg2 });
const result2 = future2.await(io);
```

**Key difference:** `io.async()` may execute the function eagerly/inline (returns `null` future if result is already computed). `io.concurrent()` guarantees a separate unit of concurrency but can fail if the thread pool is exhausted. Use `concurrent` for CPU-bound work where you need true parallelism.

**Cancellation:**
```zig
// Cancel a future (blocks until task finishes, task gets error.Canceled)
const result = future.cancel(io);

// Check for cancellation in long-running CPU work
try io.checkCancel();

// Block cancellation in a critical section
const prev = io.swapCancelProtection(.blocked);
defer _ = io.swapCancelProtection(prev);
```

**See Section 22 for full structured concurrency details (Group, Select, Batch).**

**DO NOT generate:** `suspend`, `resume`, `nosuspend`, `@frameSize` — the old 0.11-era frame-based async primitives are gone. The new async/await in 0.16 is thread-pool-based via `std.Io`.

### 1.6 `std.time`, `std.fs`, and Other APIs Moved to `std.Io`

Many standalone standard library functions **no longer exist** in 0.16. They have been absorbed into the `std.Io` interface and require an `io` handle.

| Old (BROKEN in 0.16) | New |
|---|---|
| `std.time.sleep(ns)` | `io.sleep(std.Io.Duration.fromSeconds(n), .awake)` |
| `std.time.timestamp()` | `std.Io.Timestamp.now(io, .real)` |
| `std.time.nanoTimestamp()` | `std.Io.Timestamp.now(io, .awake).toNanoseconds()` |
| `std.time.Timer` | `std.Io.Timestamp.now(io, .awake)` + `.untilNow()` |
| `std.fs.cwd()` | `std.Io.Dir.cwd()` |
| `std.fs.openFileAbsolute(path)` | `std.Io.Dir.cwd().openFile(io, path, .{})` |
| `std.io.getStdOut()` | `std.Io.File.stdout().writer(io, &.{})` |
| `std.io.getStdErr()` | `std.Io.File.stderr().writer(io, &.{})` |
| `std.io.getStdIn()` | `std.Io.File.stdin().reader(io, &.{})` |
| `std.io.BufferedWriter` | Built into `std.Io.Writer` (no separate type) |
| `std.io.CountingWriter` | `std.Io.Writer.Discarding` (has a count) |
| `std.crypto.random` | `io.random(buffer)` or `try io.randomSecure(buffer)` |

### 1.7 `std.os` → `std.posix`

`std.os` was renamed to `std.posix` in 0.12.

### 1.8 `std.mem.split`/`tokenize` Renamed

| Old (BROKEN) | New |
|---|---|
| `std.mem.split(u8, str, sep)` | `std.mem.splitSequence(u8, str, sep)` |
| `std.mem.tokenize(u8, str, sep)` | `std.mem.tokenizeAny(u8, str, sep)` |
| (new) | `std.mem.splitScalar(u8, str, char)` — for single-char delimiter |
| (new) | `std.mem.tokenizeScalar(u8, str, char)` — for single-char delimiter |

### 1.9 `@fieldParentPtr` Change

**Old (BROKEN):**
```zig
const self = @fieldParentPtr(ZiglingStep, "step", step);
```

**New:**
```zig
const self: *ZiglingStep = @alignCast(@fieldParentPtr("step", step));
```

Type is inferred from the result; `@alignCast` is often needed alongside.

### 1.10 ArrayList API Change

**Old (BROKEN):**
```zig
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();
try list.append(42);
```

**New:**
```zig
var list: std.ArrayList(u32) = .empty;
defer list.deinit(allocator);
try list.append(allocator, 42);
```

The allocator is now passed to each operation rather than stored in the struct. Initialize with `.empty` instead of `.init(allocator)`.

### 1.11 `GeneralPurposeAllocator` → `DebugAllocator` (0.16, March 2026)

**Old (BROKEN):**
```zig
var gpa: std.heap.GeneralPurposeAllocator(.{}) = .init;
```

**New:**
```zig
var gpa: std.heap.DebugAllocator(.{}) = .init;
```

`GeneralPurposeAllocator` has been renamed to `DebugAllocator`. Note: in most programs using `pub fn main(init: std.process.Init)`, prefer `init.gpa` over creating your own allocator.

### 1.12 `usingnamespace` Removed (0.15)

The `usingnamespace` keyword has been **permanently removed** from the language. There is no replacement — restructure code to use explicit imports.

### 1.13 `std.Io.Writer` / `std.Io.Reader` Are Non-Generic (0.15+)

The old `std.io.Writer` and `std.io.Reader` were comptime-generic types. The new `std.Io.Writer` and `std.Io.Reader` (capital `I`) are **vtable-based, non-generic**:

- The buffer lives **in the interface**, not the implementation
- Hot path operates on buffer directly; vtable calls only when buffer is full/empty
- `BufferedWriter` no longer exists — buffering is built into `std.Io.Writer`
- `CountingWriter` no longer exists — use `std.Io.Writer.Discarding` (has a count)
- For allocating output: `std.Io.Writer.Allocating`
- For fixed-buffer output: `std.Io.Writer.fixed`

**Custom `format` function signature changed:**
```zig
// Old (still compiles but {} won't call it)
pub fn format(self: @This(), comptime fmt: []const u8, options: std.fmt.FormatOptions, writer: anytype) !void

// New — use {f} specifier to invoke:
pub fn format(self: @This(), writer: *std.Io.Writer) std.Io.Writer.Error!void
```

**IMPORTANT:** Custom `format` methods are now invoked via `{f}`, NOT `{}`. Using `{}` on a struct prints the default field representation. Example:
```zig
const x = MyType{ .value = 42 };
std.debug.print("{f}", .{x});  // calls MyType.format → "MyType(42)"
std.debug.print("{}", .{x});   // default → ".{ .value = 42 }"
```

---

## 2. Basic Syntax

### Variable Declarations
```zig
const foo: u8 = 20;       // immutable, type explicit
const bar = @as(u8, 20);  // immutable, type via @as
var baz: u8 = 20;         // mutable, type required for var
```

- `const` values can have types inferred; `var` requires explicit types at runtime.
- All declarations must be used or prefixed with `_` to discard.

### Numeric Types
- **Integers**: `u8`, `u16`, `u32`, `u64`, `i8`, `i16`, `i32`, `i64`, `usize`, `isize`
- **Floats**: `f16`, `f32`, `f64`, `f80`, `f128`, `c_longdouble`
- **Comptime**: `comptime_int`, `comptime_float` — arbitrary precision at compile time

### Integer Literals
```zig
const dec: u8 = 255;
const hex: u8 = 0xFF;
const oct: u8 = 0o377;
const bin: u8 = 0b11111111;
const chr: u8 = 'A';              // ASCII code point (65)
const big: u32 = 14_689_520;      // underscores for readability
```

### Float Literals
```zig
const a: f32 = 1.2e+3;            // 1200.0
const b: f16 = 0x2A.F7p+3;        // hex float with 'p' exponent
```

### Imports
```zig
const std = @import("std");
const mymod = @import("mymodule.zig");
```

### The Four Kinds of "No Value"
| Value | Meaning | Type Context |
|---|---|---|
| `undefined` | Not yet initialized; reading is a bug | Any type |
| `null` | Explicit absence of a value | `?T` (optionals) |
| `error` | An error condition | `E!T` (error unions) |
| `void` | Zero-bit type; there will never be a value | `void` |

---

## 3. Arrays and Strings

### Arrays
```zig
const a: [3]u32 = [3]u32{ 42, 108, 5423 };
const b = [_]u32{ 42, 108, 5423 };     // inferred length
const len = a.len;                       // .len property
var c: [3]u32 = .{ 1, 2, 3 };          // anonymous list literal
```

### Comptime-Only Array Operators
```zig
const ab = a ++ b;                      // concatenation
const rep = [_]u8{ 1, 2 } ** 3;        // repetition: 1 2 1 2 1 2
```

### Strings Are `[]const u8`
```zig
const hello = "Hello";  // type: *const [5:0]u8 (pointer to sentinel-terminated array)
```

A string literal is `*const [N:0]u8` and coerces to:
- `[]const u8` — slice (most common parameter type)
- `[:0]const u8` — sentinel-terminated slice
- `[*:0]const u8` — sentinel-terminated many-item pointer
- `[*]const u8` — many-item pointer

### Multi-line Strings
```zig
const text =
    \\Line one
    \\Line two
    \\Line three
;
```

---

## 4. Control Flow

### if/else
```zig
if (condition) {
    // ...
} else if (other) {
    // ...
} else {
    // ...
}

// As expression:
const val: u8 = if (discount) 17 else 20;
```

Zig `if` only accepts `bool`. No implicit integer-to-bool coercion.

### while
```zig
while (n < 1024) { n *= 2; }
while (n < 1000) : (n *= 2) { ... }         // continue expression
while (iter.next()) |item| { ... }           // with payload capture
```

- `continue` → jumps to continue expression, then re-checks condition.
- `break` → exits immediately, continue expression does NOT execute.

### for
```zig
for (items) |item| { ... }                   // iterate slice/array
for (items, 0..) |item, i| { ... }           // with index counter
for (0..10) |i| { ... }                      // range: 0 to 9
for (a, b, c) |x, y, z| { ... }             // multi-object iteration
for (roles, gold, 1..) |role, g, i| { ... }  // mix arrays + counter
```

- `0..` is an open-ended counter. Starting value is arbitrary (`500..`).
- Multi-object for loops iterate multiple arrays/slices in lockstep.

### for/while as Expressions
```zig
const result: ?[]const u8 = for (items) |item| {
    if (matches(item)) break item;
} else null;
```

### switch
```zig
switch (value) {
    1 => doA(),
    2, 3 => doB(),
    4...9 => doC(),
    else => doDefault(),
}

// As expression:
const name = switch (role) {
    .wizard => "Wizard",
    .warrior => "Warrior",
    else => "Unknown",
};
```

- Must be exhaustive (cover all cases or use `else`).
- With enums, no `else` needed if all variants are listed.

### Labeled Switch (NEW in 0.14)
```zig
foo: switch (@as(u8, 1)) {
    1 => continue :foo 2,   // re-enter switch with value 2
    2 => continue :foo 3,
    3 => return,
    else => {},
}
```

State machine primitive — `continue :label value` re-evaluates the switch with a new discriminant.

### Labeled Blocks and Loops
```zig
const result = blk: {
    if (condition) break :blk value_a;
    break :blk value_b;
};

outer: for (matrix) |row| {
    for (row) |cell| {
        if (cell == target) break :outer;
        continue :outer;
    }
};
```

---

## 5. Functions

```zig
fn add(a: u32, b: u32) u32 {
    return a + b;
}

pub fn main() void { ... }     // pub makes it externally visible
```

- Functions are **private** by default.
- Parameters are always **immutable** within the function body.
- Return type is mandatory (use `void` for no return value).

---

## 6. Error Handling

### Error Sets
```zig
const FileError = error{ NotFound, PermissionDenied, OutOfMemory };
```

### Error Unions
```zig
fn openFile(path: []const u8) FileError!File {
    if (notFound) return FileError.NotFound;
    return file;
}
```

### `try` — Propagate Errors
```zig
const file = try openFile("data.txt");  // returns error to caller if error
```

Equivalent to:
```zig
const file = openFile("data.txt") catch |err| return err;
```

### `catch` — Handle or Default
```zig
const val = risky() catch 42;                       // default value
const val = risky() catch |err| handleError(err);   // handle error
```

### `if` with Error Unions
```zig
if (mayFail()) |value| {
    // success path
} else |err| {
    // error path
}
```

### Inferred Error Sets
```zig
pub fn main() !void { ... }  // ! means error type is inferred
```

### `defer` and `errdefer`
```zig
const file = try openFile("data.txt");
defer file.close();          // runs when scope exits (success or error)

const buf = try allocate();
errdefer free(buf);          // runs ONLY if scope exits with error
```

`errdefer` can capture the error:
```zig
errdefer |err| {
    std.debug.print("Failed with: {}\n", .{err});
}
```

---

## 7. Pointers and Memory

### Single-Item Pointers
```zig
var num: u8 = 5;
const ptr: *u8 = &num;     // take address
const val = ptr.*;          // dereference
ptr.* = 10;                // write through pointer
```

### Pointer Mutability
| Declaration | Can change pointer? | Can change pointed-to value? |
|---|---|---|
| `var p: *u8` | Yes | Yes |
| `const p: *u8` | No | Yes |
| `var p: *const u8` | Yes | No |
| `const p: *const u8` | No | No |

### Pass by Reference
```zig
fn increment(x: *u32) void { x.* += 1; }
increment(&my_var);
```

### Struct Pointer Auto-Dereference
```zig
fn move(v: *Vertex) void {
    v.x += 1;   // no need for v.*.x
}
```

### Pointer Type Cheatsheet
| Type | Meaning |
|---|---|
| `*T` | pointer to exactly one `T` |
| `*const T` | pointer to one immutable `T` |
| `[*]T` | many-item pointer (unknown length) |
| `[*:0]T` | sentinel-terminated many-item pointer |
| `*[N]T` | pointer to array of N elements |
| `[]T` | slice (pointer + length) |
| `[:0]T` | sentinel-terminated slice |

### Memory Allocation
```zig
// Arena allocator (frees everything at once)
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();
const allocator = arena.allocator();

const buf = try allocator.alloc(u8, 1024);
defer allocator.free(buf);

// General purpose allocator (individual alloc/free)
var gpa: std.heap.GeneralPurposeAllocator(.{}) = .init;
defer _ = gpa.deinit();
const allocator = gpa.allocator();
```

---

## 8. Slices

```zig
const arr = [_]u8{ 1, 2, 3, 4, 5 };
const slice: []const u8 = arr[1..4];    // elements at index 1, 2, 3
const rest: []const u8 = arr[2..];      // from index 2 to end
const all: []const u8 = &arr;           // whole array as slice

// Slice has .len and .ptr
const length = slice.len;       // 3
const raw_ptr = slice.ptr;      // [*]const u8
```

A slice is a fat pointer: `{ ptr: [*]T, len: usize }`.

---

## 9. Structs

```zig
const Character = struct {
    name: []const u8,
    health: u8 = 100,           // default value
    gold: u32,
    mentor: ?*Character = null, // optional pointer, defaults to null

    // Method (first param is self)
    pub fn takeDamage(self: *Character, amount: u8) void {
        self.health -= amount;
    }

    // Static function (no self)
    pub fn create(name: []const u8) Character {
        return .{ .name = name, .gold = 0 };
    }
};

var hero = Character{ .name = "Glorp", .gold = 10 };
hero.takeDamage(5);
const npc = Character.create("Bob");
```

### Anonymous Struct Literals
```zig
fn printPoint(p: struct { x: i32, y: i32 }) void { ... }
printPoint(.{ .x = 5, .y = 10 });
```

### Tuples (Anonymous Structs Without Field Names)
```zig
const tup = .{ true, @as(i32, 42), "hello" };
// Access: tup[0], tup[1], tup[2]
// Or: tup.@"0", tup.@"1", tup.@"2"
```

The `.{}` in `std.debug.print("x={}\n", .{x})` creates a tuple.

---

## 10. Enums

```zig
const Direction = enum { north, south, east, west };
const dir: Direction = .north;

// With explicit backing type and values
const Color = enum(u32) {
    red = 0xff0000,
    green = 0x00ff00,
    blue = 0x0000ff,
};

const val: u32 = @intFromEnum(Color.red);  // 0xff0000
const color: Color = @enumFromInt(0x00ff00); // .green
```

- In switch/assignments, use shorthand `.variant` instead of `Type.variant`.
- Enums can have methods, just like structs.

---

## 11. Unions and Interfaces

### Tagged Unions
```zig
const Shape = union(enum) {
    circle: f64,          // radius
    rectangle: struct { w: f64, h: f64 },
    none: void,

    pub fn area(self: Shape) f64 {
        return switch (self) {
            .circle => |r| std.math.pi * r * r,
            .rectangle => |rect| rect.w * rect.h,
            .none => 0,
        };
    }
};

const s = Shape{ .circle = 5.0 };
const a = s.area();
```

### `inline else` Pattern for Interfaces (Idiomatic)

This is Zig's approach to polymorphism — no vtables, no OOP inheritance:

```zig
const Animal = union(enum) {
    dog: Dog,
    cat: Cat,
    bird: Bird,

    pub fn speak(self: Animal) void {
        switch (self) {
            inline else => |case| return case.speak(),
        }
    }
};
```

`inline else` unrolls at comptime, calling each variant's `.speak()` method. This is zero-overhead polymorphism.

---

## 12. Optionals

```zig
var x: ?u32 = 42;
x = null;

// Unwrap with default
const val = x orelse 0;

// Unwrap or crash (assert non-null)
const val = x.?;          // equivalent to: x orelse unreachable

// Unwrap in if
if (x) |value| {
    std.debug.print("Got: {}\n", .{value});
} else {
    std.debug.print("Was null\n", .{});
}

// Unwrap in while
while (iterator.next()) |item| { ... }
```

---

## 13. Type Coercion

Key coercion rules:
1. Mutable → immutable (e.g., `*u8` → `*const u8`)
2. Numeric widening (`u8` → `u16`, `f16` → `f32`)
3. `*[N]T` → `[]T` (array pointer to slice)
4. `*[N]T` → `[*]T` (array pointer to many-item pointer)
5. `*T` → `*[1]T`
6. `T` → `?T` (value to optional)
7. `T` → `E!T` (value to error union)
8. `undefined` → any type
9. Comptime numbers → compatible runtime types
10. Tagged union → its tag enum

---

## 14. Comptime and Metaprogramming

### Comptime Variables
```zig
comptime var count: u32 = 0;
count += 1;
const arr: [count]u8 = .{'A'} ** count;
```

### Comptime Parameters (Generics)
```zig
fn Matrix(comptime T: type, comptime rows: usize, comptime cols: usize) type {
    return struct {
        data: [rows][cols]T,

        const Self = @This();

        pub fn init() Self {
            return .{ .data = .{.{0} ** cols} ** rows };
        }
    };
}

const Mat4 = Matrix(f32, 4, 4);
var m = Mat4.init();
```

### `anytype` for Duck Typing
```zig
fn process(item: anytype) void {
    if (@hasDecl(@TypeOf(item), "serialize")) {
        item.serialize();
    }
}
```

### `inline for` / `inline while`
```zig
const fields = @typeInfo(MyStruct).@"struct".fields;
inline for (fields) |field| {
    std.debug.print("Field: {s}\n", .{field.name});
}
```

### Comptime Blocks
```zig
const precomputed = comptime blk: {
    var result: [256]u8 = undefined;
    for (&result, 0..) |*r, i| {
        r.* = @intCast(i * 2);
    }
    break :blk result;
};
```

### Implicit Comptime Contexts
These locations are always evaluated at comptime:
- Container-level scope (outside functions)
- Type annotations of variables, function parameters, struct/union/enum fields
- `@cImport()` expressions
- Inline loop test expressions

---

## 15. Builtin Functions

### Common Builtins
| Builtin | Purpose |
|---|---|
| `@import("std")` | Import a module |
| `@as(T, value)` | Explicit type coercion |
| `@intCast(value)` | Cast between integer types (type inferred) |
| `@floatFromInt(value)` | Integer to float (type inferred) |
| `@intFromFloat(value)` | Float to integer (type inferred) |
| `@intFromEnum(e)` | Enum to integer |
| `@enumFromInt(i)` | Integer to enum (type inferred) |
| `@truncate(value)` | Truncate to smaller integer (type inferred) |
| `@ptrCast(ptr)` | Cast pointer types (type inferred) |
| `@alignCast(ptr)` | Assert pointer alignment |
| `@bitCast(value)` | Reinterpret bits as different type |
| `@bitReverse(value)` | Reverse all bits |
| `@addWithOverflow(a, b)` | Returns `{result, overflow_bit}` |
| `@This()` | Type of innermost struct/enum/union |
| `@TypeOf(value)` | Returns type of a value |
| `@typeInfo(T)` | Returns compile-time type metadata |
| `@typeName(T)` | Returns string name of type |
| `@hasDecl(T, "name")` | Check if type has declaration |
| `@hasField(T, "name")` | Check if type has field |
| `@field(obj, "name")` | Access field by comptime string |
| `@compileLog(...)` | Debug print at compile time |
| `@compileError(msg)` | Emit compile error |
| `@sizeOf(T)` | Size of type in bytes |
| `@alignOf(T)` | Alignment of type |
| `@splat(value)` | Create vector with all elements = value |
| `@reduce(op, vec)` | Reduce vector to scalar |
| `@abs(value)` | Absolute value |

---

## 16. Sentinels

Sentinel values mark the end of a sequence (like C's null-terminated strings).

```zig
// Sentinel-terminated array
const a: [4:0]u32 = .{ 1, 2, 3, 4 };     // element at index 4 is 0

// Sentinel-terminated slice
const b: [:0]const u8 = "hello";

// Inferred-size sentinel array
const c = [_:0]u32{ 1, 2, 3, 4, 5 };

// Many-item pointer with sentinel
const d: [*:0]const u8 = "hello";

// Get sentinel value at comptime
const sentinel = std.meta.sentinel(@TypeOf(ptr));
```

---

## 17. Vectors / SIMD

```zig
const v1: @Vector(4, i32) = .{ 1, 2, 3, 4 };
const v2: @Vector(4, i32) = .{ 10, 20, 30, 40 };

const sum = v1 + v2;              // { 11, 22, 33, 44 }
const prod = v1 * v2;             // element-wise multiply
const abs_v = @abs(v1);           // builtins work on vectors

const broadcast: @Vector(4, i32) = @splat(5);   // { 5, 5, 5, 5 }
const total = @reduce(.Add, v1);                 // 10

// Convert between arrays and vectors
const arr: [4]i32 = v1;
const vec: @Vector(4, i32) = arr;
```

---

## 18. Standard Library

### Debug Output (unchanged, no `io` needed)
```zig
std.debug.print("Name: {s}, Age: {}\n", .{ name, age });
```

### Format Specifiers
| Specifier | Meaning |
|---|---|
| `{}` | Default format |
| `{s}` | String (for `[]const u8`) |
| `{c}` | ASCII character |
| `{u}` | UTF-8 character |
| `{d}` | Decimal integer |
| `{x}` | Hexadecimal |
| `{b}` | Binary |
| `{e}` | Scientific notation float |
| `{f}` | **Custom format** — calls `.format(writer)` on the value (NEW in 0.16) |
| `{any}` | Any type (debug) |
| `{!s}` | Error name as string |

### Format Modifiers
```zig
std.debug.print("{d:>8}", .{42});      // right-align, width 8
std.debug.print("{d:0>4}", .{42});     // zero-pad, width 4: "0042"
std.debug.print("{s:*^20}", .{"hi"});  // center with *, width 20
std.debug.print("{d:.3}", .{3.14159}); // 3 decimal places
```

### Memory Operations (`std.mem`)
```zig
std.mem.eql(u8, slice_a, slice_b)           // equality check
std.mem.indexOf(u8, haystack, needle)        // find substring → ?usize
std.mem.count(u8, haystack, needle)          // count occurrences
std.mem.splitSequence(u8, str, sep)          // split by substring
std.mem.splitScalar(u8, str, char)           // split by single char
std.mem.tokenizeAny(u8, str, delimiters)     // tokenize by any of chars
std.mem.tokenizeScalar(u8, str, char)        // tokenize by single char
std.mem.trim(u8, str, chars_to_trim)         // trim edges
std.mem.startsWith(u8, str, prefix)          // prefix check
std.mem.endsWith(u8, str, suffix)            // suffix check
std.mem.asBytes(&value)                      // view value as byte slice
std.mem.zeroes(T)                            // zero-initialized T
```

### Math (`std.math`)
```zig
std.math.pow(u32, base, exp)
std.math.pi
std.math.maxInt(u8)           // 255
std.math.minInt(i8)           // -128
```

### ASCII (`std.ascii`)
```zig
std.ascii.isAlphabetic(c)
std.ascii.isDigit(c)
std.ascii.isAscii(c)
std.ascii.toLower(c)
std.ascii.toUpper(c)
```

### Sorting (`std.sort`)
```zig
std.sort.insertion(T, slice, context, lessThanFn);
```

### Hashing (`std.hash`)
```zig
const hash = std.hash.Crc32.hash(data);
```

---

## 19. The New std.Io Interface (0.15/0.16)

This section details the new I/O system. **This is the single biggest change in modern Zig.**

### Architecture

`std.Io` is a **vtable-based interface** — just two fields:
```zig
const Io = struct {
    userdata: ?*anyopaque,
    vtable: *const VTable,
};
```

All I/O operations dispatch through the vtable. Platform-specific backends:

| Platform | Backend | Notes |
|---|---|---|
| Apple (macOS, iOS, etc.) | `Io.Dispatch` (Grand Central Dispatch) | Native Apple async |
| Linux | `Io.Uring` (io_uring) | Kernel-level async I/O |
| FreeBSD, NetBSD, OpenBSD, DragonFly | `Io.Kqueue` | BSD event notification |
| Fallback / explicit | `Io.Threaded` | Thread-pool-based blocking I/O |

Sub-modules: `Io.Reader`, `Io.Writer`, `Io.File`, `Io.Dir`, `Io.Terminal`, `Io.net`, `Io.RwLock`, `Io.Semaphore`

### `std.process.Init` ("Juicy Main")

The new `main()` signature receives an `init` struct that provides everything a program needs:

```zig
// Full definition of Init:
pub const Init = struct {
    minimal: Minimal,          // raw args + environ (lightweight)
    arena: *std.heap.ArenaAllocator,  // process-lifetime allocations
    gpa: std.mem.Allocator,    // thread-safe general purpose allocator (with leak checking in debug)
    io: Io,                    // the universal I/O + concurrency handle
    environ_map: *Environ.Map, // environment variables (NOT threadsafe)
    preopens: Preopens,        // named files from parent process (mainly WASI)

    pub const Minimal = struct {
        environ: Environ,
        args: Args,
    };
};
```

**Standard usage:**
```zig
pub fn main(init: std.process.Init) !void {
    const io = init.io;       // threaded I/O handle (thread pool, file I/O, sleep, random, async)
    const gpa = init.gpa;     // thread-safe allocator — prefer over creating your own DebugAllocator
    const arena = init.arena;  // arena allocator for process-lifetime data
}
```

**Lightweight alternative** — use `Init.Minimal` if you only need args/env (no I/O, no allocators):
```zig
pub fn main(init: std.process.Init.Minimal) !void {
    const args = try init.args.toSlice(std.heap.page_allocator);
    // No io, gpa, or arena available
}
```

- `init.io` — A `std.Io` backed by a thread pool (CPU count - 1 threads). Provides file I/O, sleep, randomness, async/concurrent dispatch. On Apple platforms, the async backend uses Grand Central Dispatch.
- `init.gpa` — Thread-safe `std.mem.Allocator` with leak checking in debug mode. Prefer this over creating your own `DebugAllocator` or using `page_allocator`.
- `init.arena` — Arena allocator for allocations that live until program exit.
- `init.environ_map` — Parsed environment variables. **Not threadsafe** — only access from the main thread or protect with a lock.

**Note**: `pub fn main() void` and `pub fn main() !void` still work for simple programs that only use `std.debug.print()`.

### Getting the `io` Handle

**In `main()`:**
```zig
pub fn main(init: std.process.Init) !void {
    const io = init.io;
    // pass io to all I/O operations
}
```

**In build scripts:**
```zig
const io = b.graph.io;
```

**In threads (no init available):**
```zig
var io_instance: std.Io.Threaded = .init_single_threaded;
const io = io_instance.io();
```

### stdout / stderr
```zig
// Writing to stdout
var stdout_writer = std.Io.File.stdout().writer(io, &.{});
const stdout = &stdout_writer.interface;
try stdout.print("Hello {s}!\n", .{name});

// Writing to stderr
var stderr_writer = std.Io.File.stderr().writer(io, &.{});
const stderr = &stderr_writer.interface;
try stderr.print("Error: {}\n", .{err});
```

### File Operations
```zig
const cwd = std.Io.Dir.cwd();

// Open/create files
const file = try cwd.openFile(io, "data.txt", .{});
defer file.close(io);

const new_file = try cwd.createFile(io, "output.txt", .{});
defer new_file.close(io);

// Read from file
var file_reader = file.reader(io, &.{});
const reader = &file_reader.interface;

// Write to file
var file_writer = new_file.writer(io, &.{});
const writer = &file_writer.interface;
try writer.print("data: {}\n", .{value});

// File metadata
const size = try file.length(io);
try file.sync(io);
```

### Directory Operations
```zig
const cwd = std.Io.Dir.cwd();
const subdir = try cwd.openDir(io, "subdir", .{});
defer subdir.close(io);

try cwd.createDir(io, "new_dir", .{});
try cwd.deleteFile(io, "temp.txt");
try cwd.deleteTree(io, "build_output");
```

### Paths
```zig
const sep = std.Io.Dir.path.sep_str;        // "/" or "\\"
const full = std.Io.Dir.path.join(allocator, &.{ "src", "main.zig" });
```

### Clock, Duration, Timestamp, and Timeout

All time operations go through `std.Io`. There is no standalone `std.time` module.

**Clocks:**
```zig
const Clock = std.Io.Clock;

// Available clocks:
// .real       — wall-clock time since Unix epoch (affected by NTP)
// .awake      — monotonic, excludes suspend time (best for measuring elapsed time)
// .boot       — monotonic, includes suspend time
// .cpu_process — CPU time used by this process (user + kernel)
// .cpu_thread  — CPU time used by this thread (user + kernel)
```

**Duration (clock-agnostic):**
```zig
const Duration = std.Io.Duration;

const d1 = Duration.fromSeconds(2);
const d2 = Duration.fromMilliseconds(500);
const d3 = Duration.fromMicroseconds(1000);
const d4 = Duration.fromNanoseconds(44100);  // accepts i96

const ms = d1.toMilliseconds();  // i64
const ns = d1.toNanoseconds();   // i96
```

**Sleep:**
```zig
// Sleep using raw Duration + clock
try io.sleep(Duration.fromSeconds(2), .awake);

// Sleep using clock-tagged duration
const tagged_dur = Clock.Duration{ .raw = Duration.fromMilliseconds(100), .clock = .awake };
try tagged_dur.sleep(io);
```

**Timestamps (getting current time):**
```zig
// Raw timestamp
const now = std.Io.Timestamp.now(io, .awake);

// Clock-tagged timestamp
const tagged_now = Clock.Timestamp.now(io, .real);

// Measure elapsed time
const start = std.Io.Timestamp.now(io, .awake);
// ... do work ...
const elapsed = start.untilNow(io, .awake);  // returns Duration

// Wait until a specific time
const deadline = Clock.Timestamp.fromNow(io, .{ .raw = Duration.fromSeconds(5), .clock = .awake });
try deadline.wait(io);
```

**Timeout (for operations with deadlines):**
```zig
const Timeout = std.Io.Timeout;

// No timeout
const no_limit: Timeout = .none;

// Duration-based timeout
const dur_timeout: Timeout = .{ .duration = .{ .raw = Duration.fromSeconds(5), .clock = .awake } };

// Deadline-based timeout
const deadline_timeout: Timeout = .{ .deadline = tagged_timestamp };

// Used with operations:
try batch.awaitConcurrent(io, dur_timeout);
```

**Clock resolution:**
```zig
const res = try Clock.awake.resolution(io);  // returns Duration
```

### Randomness
```zig
var seed: u64 = undefined;
io.random(std.mem.asBytes(&seed));

// Cryptographically secure random
try io.randomSecure(buffer);

var prng = std.Random.DefaultPrng.init(seed);
const rnd = prng.random();
```

### ANSI Escape Code Detection
```zig
const stderr = std.Io.File.stderr();
if (try stderr.supportsAnsiEscapeCodes(io)) {
    // Terminal supports colors
} else if (builtin.os.tag == .windows) {
    if (stderr.enableAnsiEscapeCodes(io)) |_| {
        // Enabled on Windows
    }
}
```

---

## 20. Build System

### Minimal `build.zig`
```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the application");
    run_step.dependOn(&run_cmd.step);
}
```

### Key Build API types
- `std.Build` — the build runner
- `std.Build.Step` — a build step
- `std.Build.LazyPath` — a path that may not exist yet (was `FileSource`)
- `b.path("src/main.zig")` — create a `LazyPath` from a relative path
- `b.graph.io` — I/O interface for build scripts
- `b.graph.zig_exe` — path to the Zig compiler

### Adding Dependencies
```zig
const dep = b.dependency("mylib", .{ .target = target, .optimize = optimize });
exe.root_module.addImport("mylib", dep.module("mylib"));
```

### Adding Tests
```zig
const tests = b.addTest(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});
const run_tests = b.addRunArtifact(tests);
const test_step = b.step("test", "Run unit tests");
test_step.dependOn(&run_tests.step);
```

### `build.zig.zon` (Package Manifest)
```zig
.{
    .name = .myproject,
    .version = "0.1.0",
    .dependencies = .{
        .mylib = .{
            .url = "https://example.com/mylib-0.1.0.tar.gz",
            .hash = "...",
        },
    },
    .paths = .{ "build.zig", "build.zig.zon", "src" },
}
```

---

## 21. C Interoperability

```zig
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("math.h");
});

// Call C functions directly
const result = c.printf("Hello from C!\n");
const angle = c.atan2(y, x);

// Link C library in build.zig
exe.linkLibC();
exe.linkSystemLibrary("sqlite3");
```

Note: `@cImport` is evaluated at comptime.

---

## 22. Threading and Structured Concurrency

### Basic Threading
```zig
const std = @import("std");

fn worker(id: usize) void {
    std.debug.print("Thread {} running\n", .{id});
}

pub fn main() void {
    const handle = std.Thread.spawn(.{}, worker, .{1}) catch unreachable;
    defer handle.join();
}
```

For I/O in threads (where no `init` is available):
```zig
fn threadFn() !void {
    var io_instance: std.Io.Threaded = .init_single_threaded;
    const io = io_instance.io();
    try io.sleep(std.Io.Duration.fromSeconds(1), .awake);
}
```

### Async / Concurrent (via `std.Io`)

There are **no language-level async/await keywords**. All async is library-level on `std.Io`.

**`io.async`** — dispatch work, may run inline (eager execution allowed):
```zig
fn computeFFT(io: std.Io, data: []f32) []f32 {
    // ... expensive work ...
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    // Dispatch async — may run on thread pool or inline
    var future = io.async(computeFFT, .{ io, my_data });
    // ... do other work while it runs ...
    const result = future.await(io);  // block until done
}
```

**`io.concurrent`** — guaranteed parallel execution (requires thread pool thread):
```zig
// Must use actual parallelism — can fail if pool exhausted
var future = try io.concurrent(computeFFT, .{ io, my_data });
const result = future.await(io);
```

**Key difference:** `async` may execute eagerly (good for I/O-bound work). `concurrent` guarantees a separate thread (good for CPU-bound work like DSP). Use `concurrent` when you need true parallelism.

### Future API
```zig
// Await result (blocks until ready, idempotent, NOT threadsafe)
const result = future.await(io);

// Cancel (requests cancellation, blocks until task finishes)
// Task receives error.Canceled at next cancellation point
const result = future.cancel(io);
```

### Group — Unordered Task Set
```zig
var group: std.Io.Group = .init;

// Functions used with Group MUST return Cancelable!void (i.e. error{Canceled}!void)
fn taskA(x: *u32) std.Io.Cancelable!void { x.* += 1; }

// Spawn multiple tasks into the group
group.async(io, taskA, .{&val_a});
group.async(io, taskA, .{&val_b});
try group.concurrent(io, taskA, .{&val_c});

// Wait for ALL tasks to complete
try group.await(io);

// Or cancel all tasks
group.cancel(io);
```

Resources for each task are released when the individual task returns, not when the group completes. Tasks can only be awaited/cancelled as a whole.

### Select — Structured Concurrency with Tagged Results
```zig
// Await the FIRST completed result from multiple async tasks of different types
const ResultUnion = union(enum) {
    download: []u8,
    compute: f64,
    timeout: void,
};

var select = std.Io.Select(ResultUnion).init(io, &result_buffer);
select.async(.download, downloadFn, .{url});
select.async(.compute, computeFn, .{data});
select.async(.timeout, timeoutFn, .{duration});

// Get first completed result
const first = try select.await();
switch (first) {
    .download => |bytes| { ... },
    .compute => |val| { ... },
    .timeout => { ... },
}

// Clean up remaining tasks
select.cancelDiscard();
```

### Cancellation
```zig
// Check for cancellation in long-running CPU-bound code
try io.checkCancel();  // returns error.Canceled if cancelled

// Block cancellation in a critical section
const prev = io.swapCancelProtection(.blocked);
defer _ = io.swapCancelProtection(prev);

// Re-arm cancellation (so NEXT cancellation point also gets error.Canceled)
io.recancel();
```

Every `Io` operation that can return `error.Canceled` is a "cancellation point". Use `checkCancel()` in tight loops to allow cancellation of CPU-bound work.

### Batch — Low-Level Operation Batching
```zig
var batch = std.Io.Batch.init(&storage);
_ = batch.add(operation1);
_ = batch.add(operation2);

// Execute all operations, await results
try batch.awaitAsync(io);

// Or with timeout
try batch.awaitConcurrent(io, timeout);
```

---

## 23. Common Pitfalls for LLMs

### DO NOT generate:
1. **`std.io.getStdOut()`** → Use `std.Io.File.stdout().writer(io, &.{})` + `.interface`
2. **`std.fs.cwd()`** → Use `std.Io.Dir.cwd()`
3. **`std.time.sleep()`** → Use `io.sleep(std.Io.Duration.fromSeconds(n), .awake)`
4. **`@intCast(T, value)`** → Use `const x: T = @intCast(value)` (single arg, type inferred)
5. **`@enumToInt()`** → Use `@intFromEnum()`
6. **`@typeInfo(T).Struct`** → Use `@typeInfo(T).@"struct"`
7. **`suspend`, `resume`, `nosuspend`, `@frameSize`** → Old 0.11-era frame-based async is gone. Use `io.async(fn, .{args})` / `future.await(io)` — async/await restored in 0.16 via `std.Io`
8. **`std.mem.split()`** → Use `std.mem.splitSequence()` or `std.mem.splitScalar()`
9. **`std.ArrayList(T).init(alloc)`** → Use `std.ArrayList(T) = .empty`, pass allocator per-operation
10. **`std.os.`** → Use `std.posix.`
11. **`std.rand.`** → Use `std.Random.`
12. **`std.heap.page_allocator` in main** → Prefer `init.gpa` from `std.process.Init`
13. **`std.heap.GeneralPurposeAllocator`** → Renamed to `std.heap.DebugAllocator`. Prefer `init.gpa` in main.
14. **`usingnamespace`** → Permanently removed from the language
15. **`std.io.Writer` / `std.io.Reader`** (lowercase) → Use `std.Io.Writer` / `std.Io.Reader` (capital `I`, vtable-based)
16. **`BufferedWriter`** → Buffering is now built into `std.Io.Writer`
17. **Old `format` signature with `comptime fmt`** → New: `fn format(self: @This(), writer: *std.Io.Writer) std.Io.Writer.Error!void` — invoked via `{f}` not `{}`

### DO generate:
1. `std.debug.print()` for quick output — it still works with no I/O setup
2. `pub fn main() void` for simple programs that only use `std.debug.print()`
3. `pub fn main(init: std.process.Init) !void` for programs that need I/O
4. `pub fn main(init: std.process.Init.Minimal) !void` for lightweight programs (args/env only)
5. Single-argument builtins with type inferred from context
6. `inline else` for union dispatch (interface pattern)
7. Multi-object `for` loops for parallel iteration
8. `.{}` anonymous struct/tuple syntax
9. `0..` open-ended ranges in for loops
10. `io.async(fn, .{args})` / `future.await(io)` for async dispatch (library-level, not keywords)
11. `try io.concurrent(fn, .{args})` for guaranteed parallel execution (CPU-bound work)
12. `std.Io.Group` / `std.Io.Select` for structured concurrency

### Data-Oriented Design

Zig encourages **Struct of Arrays** (SoA) over **Array of Structs** (AoS) for cache performance. Multi-object for loops make this ergonomic:

```zig
// Preferred: Struct of Arrays
const x_coords = [_]f32{ 1.0, 2.0, 3.0 };
const y_coords = [_]f32{ 4.0, 5.0, 6.0 };
for (x_coords, y_coords) |x, y| { ... }

// Less preferred: Array of Structs
const points = [_]Point{ .{1,4}, .{2,5}, .{3,6} };
for (points) |p| { ... }
```

---

## Version Migration Timeline

| Date | Version | Key Change |
|------|---------|-----------|
| 2024-06 | 0.14.0-dev.42 | `std.mem.split`/`tokenize` renamed |
| 2024-09 | 0.14.0-dev.1573 | Labeled switch introduced |
| 2025-07 | 0.15.0-dev.1092 | New `std.Io` interface (first wave); `usingnamespace` removed |
| 2025-08 | 0.15.0-dev.1519 | ArrayList API changes |
| 2025-08 | **0.15.1 stable** | async/await keywords removed (later restored in 0.16); `std.Io.Writer`/`Reader` vtable-based; non-generic I/O |
| 2025-10 | **0.15.2 stable** | Latest stable release |
| 2025-12 | 0.16.0-dev.1859 | File system I/O integrated with `std.Io` |
| 2026-01 | 0.16.0-dev.1976 | Process API moved to `std.Io`; `main(init)` pattern |
| 2026-01 | 0.16.0-dev.2075 | Randomness API moved to `std.Io` |
| 2026-02 | 0.16.0-dev.2471 | `process.Child.Cwd` added |
| 2026-03 | 0.16.0-dev.2915 | `GeneralPurposeAllocator` renamed to `DebugAllocator` |

---

## 24. How to Update This Document

> **For LLMs**: When a user says "update your Zig knowledge" or "refresh zig_context.md", follow this procedure to bring this document up to date with the latest Zig changes.

### Step-by-Step Update Procedure

#### Step 1: Determine the Current Version Target

Check what version of Zig the user is targeting. Look for:
- `build.zig` in their project — search for a version string like `"0.16.0-dev.2471"`
- Or ask: "What version of Zig are you using?" (`zig version`)

The version this document was last updated for: **0.16.0-dev.2915** (March 2026).

#### Step 2: Check Primary Sources (in priority order)

**Source 1 — Ziglings Exercises Changelog (FASTEST, most useful)**
- Changelog/README: `https://codeberg.org/ziglings/exercises/raw/branch/main/README.md`
- Browse exercises: `https://codeberg.org/ziglings/exercises/src/branch/main/exercises/`
- Read individual exercises at: `https://codeberg.org/ziglings/exercises/raw/branch/main/exercises/001_hello.zig` (change filename as needed)
- The README contains a detailed **version changelog** listing every API break and which dev version introduced it
- This is the single best source because it tells you exactly what broke and when

**Source 2 — Zig Standard Library Source (GROUND TRUTH, for verifying specific APIs)**
- URL: `https://codeberg.org/ziglang/zig/src/branch/master/lib/std/`
- Key directories to check for changes:
  - `lib/std/Io/` — the new I/O interface (Writer.zig, Reader.zig, File.zig, Dir.zig)
  - `lib/std/process.zig` — process init, child process
  - `lib/std/Build.zig` — build system
  - `lib/std/array_list.zig` — ArrayList API
  - `lib/std/mem.zig` — memory utilities
  - `lib/std/debug.zig` — debug printing
- Look at recent commits to `lib/std/` to spot new changes
- The actual source code is the definitive answer for "does this API exist?"

**Source 3 — Zig Release Notes (for stable releases)**
- URL: `https://ziglang.org/download/` — links to release notes for each version
- Check if a new stable release (0.15.x, 0.16.0, etc.) has shipped since the last update
- Release notes summarize all breaking changes in one place

**Source 4 — zig.guide (supplementary)**
- URL: `https://zig.guide`
- Good for beginner-friendly explanations but tends to lag behind dev builds
- Useful for verifying idioms and patterns, less useful for bleeding-edge changes

#### Step 3: Identify What Changed

Compare the version the user is on vs. what this document covers. Focus on:

1. **New builtins added or renamed** — check `lib/std/builtin.zig`
2. **std.Io changes** — this module is still evolving rapidly
3. **Build system changes** — `std.Build` API modifications
4. **New language features** — labeled switch was added in 0.14; look for similar additions
5. **Removed features** — anything newly deprecated or deleted
6. **Standard library reorganization** — modules moving, renaming, API signature changes

#### Step 4: Update This Document

For each change found:

1. **If it's a breaking change**: Add it to **Section 1 (Critical Breaking Changes)** with a "Old (BROKEN)" / "New" code comparison
2. **If it's a new feature**: Add it to the appropriate topic section (e.g., new control flow → Section 4)
3. **If it's an API rename**: Update the relevant section AND add to Section 23 (Common Pitfalls)
4. **Update the Version Migration Timeline** table at the bottom
5. **Update the version number** in the document header (line 3)

#### Step 5: Validate

If the user has a Zig compiler available, test key patterns:
```bash
# Quick smoke test — does a minimal program compile?
echo 'const std = @import("std"); pub fn main() void { std.debug.print("ok\n", .{}); }' > /tmp/test.zig && zig run /tmp/test.zig

# Test the std.Io pattern
echo 'const std = @import("std"); pub fn main(init: std.process.Init) !void { var w = std.Io.File.stdout().writer(init.io, &.{}); try (&w.interface).print("ok\n", .{}); }' > /tmp/test_io.zig && zig run /tmp/test_io.zig
```

### What NOT to Do When Updating

- **Don't remove working patterns** — if old syntax still compiles, keep it documented
- **Don't guess** — if you can't verify a change from source, flag it as unverified
- **Don't trust your training data over the source** — your training data is likely outdated for Zig. Always defer to the actual source code and ziglings exercises
- **Don't rewrite the whole document** — make targeted updates to affected sections

### Quick Reference: Key Files to Check in Zig Source

| What Changed? | Check This File |
|---|---|
| I/O interface (core) | `lib/std/Io.zig` (Duration, Clock, Timestamp, Timeout, Future, Group, Select, Batch) |
| I/O Reader/Writer | `lib/std/Io/Writer.zig`, `lib/std/Io/Reader.zig` |
| File I/O | `lib/std/Io/File.zig` |
| File system / dirs | `lib/std/Io/Dir.zig` |
| Networking | `lib/std/Io/net.zig` |
| Terminal | `lib/std/Io/Terminal.zig` |
| Threaded backend | `lib/std/Io/Threaded.zig` |
| Apple GCD backend | `lib/std/Io/Dispatch.zig` |
| Linux io_uring backend | `lib/std/Io/Uring.zig` |
| BSD kqueue backend | `lib/std/Io/Kqueue.zig` |
| Fiber / coroutine | `lib/std/Io/fiber.zig` |
| Concurrency primitives | `lib/std/Io/RwLock.zig`, `lib/std/Io/Semaphore.zig` |
| Process / main() | `lib/std/process.zig` |
| Build system | `lib/std/Build.zig`, `lib/std/Build/Step.zig` |
| ArrayList | `lib/std/array_list.zig` |
| Memory utils | `lib/std/mem.zig` |
| Formatting | `lib/std/fmt.zig`, `lib/std/Io/Writer.zig` |
| Builtins | `lib/std/builtin.zig` |
| Random | `lib/std/Random.zig` |
| Allocators | `lib/std/heap.zig`, `lib/std/heap/DebugAllocator.zig` |

---

## Useful Resources

- **Zig Language Reference**: https://ziglang.org/documentation/master/
- **Zig Standard Library Docs**: https://ziglang.org/documentation/master/std/
- **Zig Guide (0.15.2)**: https://zig.guide
- **Zig Source (Codeberg)**: https://codeberg.org/ziglang/zig
- **Zig Standard Library Source**: https://codeberg.org/ziglang/zig/src/branch/master/lib/std/
- **0.15.1 Release Notes** (major breaking changes): https://ziglang.org/download/0.15.1/release-notes.html
- **Ziglings Exercises**: https://codeberg.org/ziglings/exercises
- **Build from Source**: https://codeberg.org/ziglang/zig#building-from-source
- **Zig Devlog**: https://ziglang.org/devlog/2026/
