# Zig 0.16 Language Context for LLMs

> **Purpose**: This document provides comprehensive, up-to-date context on the Zig programming language as of **Zig 0.16.0 stable** (released April 2026). Most LLMs were trained on Zig 0.11–0.13 era code, which is now significantly outdated. Use this document to avoid generating broken or deprecated Zig code.

> **Sources**: [0.16.0 release notes](https://ziglang.org/download/0.16.0/release-notes.html), [Ziglings exercises](~/dev/ziglings/exercises/) (0.16-updated, excellent for interfaces/vectors/async/packed patterns), Zig source repository on Codeberg, ziglang.org, zig.guide. Last updated: 2026-04-20.

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

### 1.5 Async/Await Are Back in 0.16 (But `suspend`/`resume` Are Not)

**`async` and `await` are real language features again in Zig 0.16.** They were removed after 0.11 and absent through 0.15, but 0.16 restored them with a completely new implementation built on top of `std.Io`. The usage looks like method-call syntax — `io.async(fn, .{args})` and `future.await(io)` — but `async` and `await` are part of the language, not just method names. Treat them as first-class.

What is **still gone** (and is NOT coming back) is the **0.11-era frame-based system**: `suspend`, `resume`, `nosuspend`, and `@frameSize`. Those were tied to the old stackful-coroutine model and were removed in 0.15. The new async/await is thread-pool / fiber / io_uring based via `std.Io`, not frame-based.

> **TL;DR for LLMs:** `async` + `await` = YES in 0.16. `suspend`/`resume`/`nosuspend`/`@frameSize` = still gone.

The backing runtime dispatches through `std.Io` to platform-specific backends (Grand Central Dispatch on Apple, io_uring on Linux, kqueue on BSDs, thread pool as fallback).

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

**DO generate:** `io.async(fn, .{args})` and `future.await(io)` — `async` and `await` are real 0.16 language features.

**DO NOT generate:** `suspend`, `resume`, `nosuspend`, `@frameSize` — these are the 0.11-era frame-based primitives, removed in 0.15 and NOT coming back. The new async/await does not use them.

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

### 1.10 Containers: `.empty` + Per-Operation Allocator (Unmanaged-by-Default)

The "managed" container variants (that stored an allocator internally) were removed. The remaining containers are all "unmanaged" — they're initialized with the `.empty` constant (no allocator stored) and you pass the allocator into every operation that allocates.

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
try list.appendSlice(allocator, &.{ 10, 20, 30 });
```

This `.empty` + per-op-allocator pattern is the idiom across standard containers in 0.16. The `.Unmanaged` suffix is gone — the base name IS the unmanaged version. Representative list (verify exact signature against `lib/std/` for the container you're using):

| Container | Init | Example op |
|---|---|---|
| `std.ArrayList(T)` | `= .empty` | `list.append(allocator, x)` |
| `std.ArrayListAligned(T, a)` | `= .empty` | `list.append(allocator, x)` |
| `std.AutoHashMap(K, V)` | `= .empty` | `map.put(allocator, k, v)` |
| `std.AutoArrayHashMap(K, V)` | `= .empty` | `map.put(allocator, k, v)` |
| `std.StringHashMap(V)` | `= .empty` | `map.put(allocator, k, v)` |
| `std.StringArrayHashMap(V)` | `= .empty` | `map.put(allocator, k, v)` |
| `std.HashMap(K, V, Ctx, load)` | `= .empty` | `map.put(allocator, k, v)` |
| `std.MultiArrayList(T)` | `= .empty` | `list.append(allocator, x)` |
| `std.SinglyLinkedList` | `= .empty` | `list.prepend(&node)` (no alloc) |
| `std.DoublyLinkedList` | `= .empty` | `list.append(&node)` (no alloc) |
| `std.PriorityQueue(T, Ctx, cmp)` | `= .empty` | `pq.add(allocator, x)` |
| `std.ArrayHashMap(...)` | `= .empty` | `map.put(allocator, k, v)` |

**Pattern recap — three-line lifecycle:**
```zig
var things: std.ArrayList(Thing) = .empty;
defer things.deinit(allocator);
try things.append(allocator, thing);
```

**Don't write `.init(allocator)`** for these — it either won't compile or will call a differently-named constructor that does something else. If a container genuinely needs an init arg (e.g. `PriorityQueue`'s context), the pattern is `= .init(ctx)` with allocator *still* passed per-op.

**Also `.empty` (not `.init`) is the convention for:**
- `std.Io.Group` — `var group: std.Io.Group = .empty;` or similar — see §22
- User-defined containers — if your struct has a sensible zero-state, declare `pub const empty: @This() = .{ ... };` and init with `= .empty`

**Contrast with `.init`:** Reserve `.init` for types that genuinely need construction parameters (`std.heap.ArenaAllocator.init(child)`, `std.Random.DefaultPrng.init(seed)`, `std.heap.DebugAllocator(.{}) = .init`). If no params are needed, `.empty` is preferred.

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
// Old (BROKEN in 0.16 — std.fmt.FormatOptions was renamed to std.fmt.Options,
// and the `comptime fmt`/`anytype writer` shape is no longer used by {})
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

### 1.14 `@Type` Removed — Specialized Type-Creation Builtins (0.16 stable)

The generic `@Type` builtin is **gone** in 0.16. It's replaced by a family of specialized builtins, one per type kind:

**Old (BROKEN):**
```zig
const U10 = @Type(.{ .int = .{ .signedness = .unsigned, .bits = 10 } });
```

**New:**
```zig
const U10 = @Int(.unsigned, 10);
```

New builtins:

| Builtin | Purpose |
|---|---|
| `@Int(signedness, bits)` | Arbitrary-width integer type |
| `@Tuple(field_types)` | Tuple type from a slice of types |
| `@Pointer(size, attrs, Element, sentinel)` | Pointer type |
| `@Struct(layout, BackingInt, names, types, attrs)` | Struct type |
| `@Union(layout, ArgType, names, types, attrs)` | Union type |
| `@Enum(TagInt, mode, names, values)` | Enum type |
| `@Fn(param_types, param_attrs, ReturnType, attrs)` | Function type |
| `@EnumLiteral()` | The `enum literal` type |

No replacement builtins exist for `@Array`, `@Float`, `@Optional`, `@ErrorUnion`, or `@ErrorSet` — use literal syntax (`[N]T`, `f32`, `?T`, `E!T`, `error{...}`) instead.

### 1.15 Packed Unions Require Explicit Backing Integer (0.16 stable)

Packed unions now **must** specify a backing integer type, just like packed enums:

**Old (BROKEN):**
```zig
const U = packed union { x: u16, y: u16 };
```

**New:**
```zig
const U = packed union(u16) { x: u16, y: u16 };
```

Related rules (also new/tightened in 0.16):
- Pointers are **forbidden** inside `packed struct` and `packed union`.
- All fields of a `packed union` must have the same `@bitSizeOf`.
- Packed/enum types with implicit backing integer types are **forbidden in extern contexts** — you must pick an explicit backing int.

Packed unions can also now appear as switch prong items (compared by backing integer):
```zig
const U = packed union(u2) { a: i2, b: u2 };
const u: U = .{ .a = -1 };
switch (u) {
    .{ .b = 3 } => {},  // matches because bit pattern is the same
    else => unreachable,
}
```

**Packed structs are integers in disguise.** They're useful for bitflags, binary protocol headers, and viewing the same data from different perspectives. Fields start at the least significant bit regardless of endianness. You can `@bitCast` between a packed struct and its backing integer:

```zig
// LZ4 frame descriptor FLG byte — real-world protocol header
const FLG = packed struct(u8) {
    dict_id: bool,
    reserved: u1 = 0,
    content_checksum: bool,
    content_size: bool,
    block_checksum: bool,
    block_independence: bool,
    version: u2,
};

// Convert between packed struct and backing integer
const flg: FLG = .{ .version = 1, .block_independence = true, .content_size = true, .content_checksum = false, .dict_id = false };
const byte: u8 = @bitCast(flg);
const back: FLG = @bitCast(byte);

// Packed unions let you view same bits from different perspectives
const Float16 = packed union(u16) {
    value: f16,
    bits: packed struct(u16) {
        mantissa: u10,
        exponent: u5,
        sign: u1,
    },
};

var num: Float16 = .{ .value = 2.34 };
num.bits.sign = 1;  // make it negative by flipping sign bit
// num.value is now -2.34
```

Packed structs support equality comparison (`==`, `!=`) and can be used in `switch` statements (exhaustive matching on all bit patterns). They do NOT support ordering comparisons (`<`, `>`).

### 1.16 Float → Integer via `@round`/`@floor`/`@ceil`/`@trunc` (0.16 stable)

These four builtins now convert **directly** to an integer type when the destination is an integer — no separate `@intFromFloat` needed:

```zig
const x: u8 = @round(12.5);   // 12, result is u8 directly
const y: i32 = @floor(-1.7);  // -2
```

With a float destination they still produce a float (as before).

### 1.17 Small Integer → Float Coercion (0.16 stable)

Integer types whose **every value fits exactly** in a target float will now coerce implicitly:

```zig
var foo_int: u24 = 123;
var foo_float: f32 = foo_int;  // OK — 24 bits fit in f32 mantissa
```

This skips the explicit `@floatFromInt` when the compiler can prove the conversion is lossless.

### 1.18 Pointer Alignment Types Are Distinct (0.16 stable)

`*T` (naturally aligned) and `*align(1) T` (explicitly aligned) are now **distinct types**. They still coerce to each other in most contexts, but `@TypeOf` will reflect the difference, and comparisons like `@TypeOf(a) == @TypeOf(b)` may now return `false` where they previously returned `true`.

### 1.19 Concurrency Primitives Moved to `std.Io` (0.16 stable)

Synchronization primitives migrated off `std.Thread` and onto `std.Io`:

| Old (BROKEN in 0.16) | New |
|---|---|
| `std.Thread.Mutex` | `std.Io.Mutex` |
| `std.Thread.ResetEvent` | `std.Io.Event` |
| `std.Thread.Condition` | `std.Io.Condition` |
| `std.Thread.Semaphore` | `std.Io.Semaphore` |
| `std.Thread.RwLock` | `std.Io.RwLock` |
| `std.Thread.WaitGroup` | `std.Io.Group` |
| `std.Thread.Futex` | `std.Io.Futex` |
| `std.Thread.Mutex.Recursive` | **removed** — rethink the design; recursive locking was a footgun |
| `std.once` | **removed** — avoid global init; use explicit init funcs |
| `std.Thread.Pool` | **removed** — use `io.async` / `io.concurrent` / `std.Io.Group` |
| `std.heap.ThreadSafeAllocator` | **removed** — `std.heap.ArenaAllocator` is now thread-safe + lock-free; `init.gpa` is also thread-safe |

### 1.20 Error Set Renames (0.16 stable)

Several commonly-seen errors were renamed for consistency:

| Old | New |
|---|---|
| `error.RenameAcrossMountPoints` | `error.CrossDevice` |
| `error.NotSameFileSystem` | `error.CrossDevice` |
| `error.SharingViolation` | `error.FileBusy` |
| `error.EnvironmentVariableNotFound` | `error.EnvironmentVariableMissing` |

### 1.21 Process Spawning Simplified (0.16 stable)

**Old pattern:**
```zig
var child = std.process.Child.init(argv, gpa);
try child.spawn(io);
```

**New pattern:**
```zig
var child = try std.process.spawn(io, .{
    .argv = argv,
    .stdin = .pipe,
    .stdout = .pipe,
});
```

`std.process.spawn` replaces the manual `Child.init` + `spawn` two-step.

### 1.22 `std.process` Consolidation (0.16 stable)

A batch of self-exe/cwd/exec helpers moved from `std.fs` → `std.process`, and `Child.run` / `execv` were renamed:

| Old (BROKEN) | New |
|---|---|
| `std.process.Child.init(argv, gpa)` + `.spawn(io)` | `try std.process.spawn(io, .{ .argv = argv, ... })` |
| `std.process.Child.run(...)` | `std.process.run(...)` |
| `std.process.execv(...)` | `std.process.replace(...)` |
| `std.fs.openSelfExe(...)` | `std.process.openExecutable(io, ...)` |
| `std.fs.selfExePath(buf)` | `std.process.executablePath(io, buf)` |
| `std.fs.selfExePathAlloc(allocator)` | `std.process.executablePathAlloc(io, allocator)` |
| `std.fs.selfExeDirPath(buf)` | `std.process.executableDirPath(io, buf)` |
| `std.fs.selfExeDirPathAlloc(allocator)` | `std.process.executableDirPathAlloc(io, allocator)` |
| `std.fs.Dir.setAsCwd()` | `std.process.setCurrentDir(io, dir)` |

Environment variables and process args are also now **non-global** — go through `init.environ_map` / `init.minimal.args` from `std.process.Init`, not a module-level global.

### 1.23 `std.fmt` Renames (0.16 stable)

| Old (BROKEN) | New |
|---|---|
| `std.fmt.FormatOptions` | `std.fmt.Options` |
| `std.fmt.Formatter` | `std.fmt.Alt` |
| `std.fmt.format(writer, ...)` | `std.Io.Writer.print(writer, ...)` |
| `std.fmt.bufPrintZ` | `std.fmt.bufPrintSentinel` |

### 1.24 Removed I/O Shims (0.16 stable)

The following no longer exist — they're subsumed by the non-generic `std.Io.Writer`/`std.Io.Reader`:

- `std.Io.GenericReader`, `std.Io.AnyReader`
- `std.Io.GenericWriter`, `std.Io.AnyWriter`
- `std.Io.FixedBufferStream` — use `std.Io.Writer.fixed(...)` or build on a `Reader` directly
- `std.Io.CountingReader` — parallel removal to `CountingWriter` (use `std.Io.Writer.Discarding` for writers; for readers, track offsets yourself)
- `std.Io.null_writer` — use `std.Io.Writer.Discarding`

### 1.25 File API: Streaming vs Positional, `Reader`-Owned Seeks (0.16 stable)

`std.Io.File` read/write methods split into two families based on whether the operation uses the file's implicit cursor (streaming) or an explicit offset (positional). Seeks moved off `File` onto `File.Reader`:

| Old `std.fs.File` | New `std.Io.File` |
|---|---|
| `file.read(buf)` / `file.readv(iov)` | `file.readStreaming(io, buf)` |
| `file.pread(buf, offset)` / `file.preadv` / `file.preadAll` | `file.readPositional(io, buf, offset)` / `readPositionalAll` |
| `file.write(bytes)` / `file.writev(iov)` / `file.writeAll` | `file.writeStreaming(io, bytes)` / `writeStreamingAll` |
| `file.pwrite(bytes, off)` / `pwritev` / `pwriteAll` | `file.writePositional(io, bytes, off)` / `writePositionalAll` |
| `file.seekTo/seekBy/seekFromEnd(n)` | `file.reader(io, &buf).seekTo(n)` etc. (lives on `Reader`) |
| `file.getPos()` | `file_reader.logicalPos()` |
| `file.getEndPos()` | `file.length(io)` |
| `file.setEndPos(n)` | `file.setLength(io, n)` |
| `file.mode()` | `file.stat(io).permissions.toMode()` |
| `file.chmod(m)` / `file.chown(...)` | `file.setPermissions(io, p)` / `file.setOwner(io, ...)` |
| `file.updateTimes(...)` | `file.setTimestamps(io, ...)` / `setTimestampsNow(io)` |
| `std.fs.File.Mode` / `PermissionsWindows` / `PermissionsUnix` | `std.Io.File.Permissions` (unified) |
| `std.fs.File.default_mode` | `std.Io.File.Permissions.default_file` |

Directory-side renames worth knowing: `Dir.makeDir` → `createDir`, `Dir.makePath` → `createDirPath`, `Dir.makeOpenDir` → `createDirPathOpen`, `Dir.realpath`/`realpathAlloc` → `realPathFile`/`realPathFileAlloc`. Legacy `fs.path`, `fs.max_path_bytes`, `fs.max_name_bytes` are deprecated in favor of `std.Io.Dir.path` / `Dir.max_path_bytes` / `Dir.max_name_bytes`. The platform-suffixed variants (`*Z`, `*W`, `*Wasi`) and the `adaptToNewApi`/`adaptFromNewApi` shims are gone — use the unified `std.Io` APIs.

### 1.26 Vector Restrictions (0.16 stable)

Two breaking vector rules:

1. **Runtime indexing is forbidden.** Use `@shuffle`, `@reduce`, or convert to an array first (`const arr: [N]T = v;`) for runtime-indexed access.
2. **Vectors and arrays no longer in-memory coerce.** You must explicitly convert:
   ```zig
   const v: @Vector(4, f32) = .{ 1, 2, 3, 4 };
   const a: [4]f32 = v;                // OK — assignment coercion still works
   // @as(*[4]f32, &v)                  // NOT OK in 0.16 — pointers don't coerce
   ```
   Assignment between `[N]T` and `@Vector(N, T)` still works, but you cannot reinterpret pointers between them.

### 1.27 Unary Float Builtins Forward Result Types (0.16 stable)

These now participate in result-type inference so the output element/float type is taken from context:

`@sqrt`, `@sin`, `@cos`, `@tan`, `@exp`, `@exp2`, `@log`, `@log2`, `@log10`

```zig
const x: f32 = 2.0;
const y: f64 = @sqrt(x);   // widens to f64 via result-type inference
```

Previously they always returned the same float type as the argument.

### 1.28 Miscellaneous Language Tightenings (0.16 stable)

- **Zero-bit tuple fields are no longer implicitly `comptime`** — if you want compile-time-only, annotate explicitly.
- **You cannot return the address of a trivial local variable** from a function (would dangle).
- **Pointers to comptime-only types are no longer themselves comptime-only.**
- **Switch prong captures may no longer all be discarded** — at least one capture in the prong set must be used. (Other switch wins: packed structs as prong items, decl literals as prong items, union-tag captures allowed on all prongs not just `inline`, and prongs whose value isn't in the error set are allowed if the body is `=> comptime unreachable`.)
- **`@cImport` is DEPRECATED** (not just discouraged) — migrate to `b.addTranslateC(...)` in `build.zig`.

### 1.29 `std.SegmentedList` and `std.meta.declList` Removed (0.16 stable)

- `std.SegmentedList` is gone — use `std.ArrayList` with `.empty`, or a custom chunked structure if you specifically needed stable element addresses.
- `std.meta.declList` is gone — iterate `@typeInfo(T).@"struct".decls` or similar directly.

Other smaller removals: `std.DynLib` no longer supports Windows; `std.math.sign` now returns the smallest int type that fits the input.

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

// Debug allocator (individual alloc/free, leak-checked in debug builds)
// Note: was `GeneralPurposeAllocator` before 0.16; prefer `init.gpa` in main().
var gpa: std.heap.DebugAllocator(.{}) = .init;
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
| `@round(f)` / `@floor(f)` / `@ceil(f)` / `@trunc(f)` | With an integer destination type, converts directly to int (0.16+) |
| `@sqrt` / `@sin` / `@cos` / `@tan` / `@exp` / `@exp2` / `@log` / `@log2` / `@log10` | Unary float; in 0.16 these forward result types (output float type inferred from context) |

### Type-Creation Builtins (0.16+, replace removed `@Type`)

| Builtin | Purpose |
|---|---|
| `@Int(signedness, bits)` | Arbitrary-width integer |
| `@Tuple(field_types)` | Tuple type |
| `@Pointer(size, attrs, Element, sentinel)` | Pointer type |
| `@Struct(layout, BackingInt, names, types, attrs)` | Struct type |
| `@Union(layout, ArgType, names, types, attrs)` | Union type |
| `@Enum(TagInt, mode, names, values)` | Enum type |
| `@Fn(param_types, param_attrs, ReturnType, attrs)` | Function type |
| `@EnumLiteral()` | Enum literal type |

For `@Array`, `@Float`, `@Optional`, `@ErrorUnion`, `@ErrorSet` — use literal syntax (`[N]T`, `f32`, `?T`, `E!T`, `error{...}`).

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

// Assignment coercion between arrays and vectors still works
const arr: [4]i32 = v1;
const vec: @Vector(4, i32) = arr;
```

### Practical SIMD Patterns

**Pairwise operations — replace scalar loops with vector ops:**
```zig
// Scalar (slow):
fn maxPairwiseDiffScalar(a: [4]f32, b: [4]f32) f32 {
    var max: f32 = 0;
    for (a, b) |x, y| {
        const d = @abs(x - y);
        if (d > max) max = d;
    }
    return max;
}

// Vector (fast — compiles to SIMD instructions):
const Vec4 = @Vector(4, f32);
fn maxPairwiseDiffVector(a: Vec4, b: Vec4) f32 {
    return @reduce(.Max, @abs(a - b));
}
```

**Key operations for DSP:**
```zig
const v: @Vector(4, f32) = .{ 1.0, 2.0, 3.0, 4.0 };

// Element-wise math (all compile to SIMD)
const scaled = v * @as(@Vector(4, f32), @splat(0.5));  // multiply by scalar
const sq = @sqrt(v);                                     // per-element sqrt
const sinv = @sin(v);                                    // per-element sin

// Reductions
const sum = @reduce(.Add, v);     // horizontal sum → 10.0
const max = @reduce(.Max, v);     // horizontal max → 4.0
const min = @reduce(.Min, v);     // horizontal min → 1.0
```

### 0.16 Rules (see §1.26)

- **No runtime indexing of vectors.** `v[i]` with runtime `i` is a compile error. Use `@shuffle`, `@reduce`, or convert to an array first (`const a: [N]T = v;` then `a[i]`).
- **No in-memory coercion between vectors and arrays.** Value assignment still coerces, but you cannot reinterpret a `*[N]T` as `*@Vector(N, T)` (or vice versa) — the in-memory layouts are no longer guaranteed to match.
- Unary float builtins (`@sqrt`, `@sin`, `@cos`, `@tan`, `@exp`, `@exp2`, `@log`, `@log2`, `@log10`) now forward result types, so they infer the element type from context when operating on vectors too.

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
std.math.sign(x)              // 0.16+: returns smallest int type that fits
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

Sub-modules: `Io.Reader`, `Io.Writer`, `Io.File`, `Io.Dir`, `Io.Terminal`, `Io.net`, `Io.Mutex`, `Io.RwLock`, `Io.Condition`, `Io.Event`, `Io.Semaphore`, `Io.Futex`, `Io.Group`, `Io.Select`, `Io.Batch`, `Io.Clock`, `Io.Duration`, `Io.Timestamp`, `Io.Timeout`, `Io.Threaded`, `Io.Evented`, `Io.Uring`, `Io.Dispatch`, `Io.Kqueue`

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

// Read into a fixed buffer (returns number of bytes actually read)
var buf: [64]u8 = undefined;
const bytes_read = try reader.readSliceShort(&buf);
const content = buf[0..bytes_read];

// Write to file
var file_writer = new_file.writer(io, &.{});
const writer = &file_writer.interface;
try writer.print("data: {}\n", .{value});
const bytes_written = try writer.write("raw bytes");

// File metadata
const size = try file.length(io);
try file.sync(io);
```

### Directory Operations
```zig
const cwd = std.Io.Dir.cwd();
const subdir = try cwd.openDir(io, "subdir", .{});
defer subdir.close(io);

// createDir takes a mode parameter — .default_dir for platform defaults
cwd.createDir(io, "new_dir", .default_dir) catch |e| switch (e) {
    error.PathAlreadyExists => {},  // idempotent create
    else => return e,
};

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

### C Translation via Build System (0.16+)

`@cImport` inside source files is being phased out in favor of an explicit build-system step. This makes caching and cross-compilation deterministic:

```zig
// In build.zig
const translate_c = b.addTranslateC(.{
    .root_source_file = b.path("src/c.h"),
    .target = target,
    .optimize = optimize,
});
exe.root_module.addImport("c", translate_c.createModule());
```

Then in Zig source: `const c = @import("c");` — no `@cImport` needed.

`@cImport` still works for now, but prefer `addTranslateC` for new code.

### New Compiler/Build Flags (0.16)

- `--error-style <style>` — control error message formatting (e.g., for editor integration)
- `--multiline-errors` — emit multi-line error output
- Unit test timeout support — guard against runaway tests in CI

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

> **0.16 status**: `@cImport` is **deprecated**. Prefer `b.addTranslateC(...)` in `build.zig` and `@import("c")` in source. The inline form still works for now but will likely be removed in a future version.

### Preferred (0.16+): Build-System Translate-C

**build.zig:**
```zig
const translate_c = b.addTranslateC(.{
    .root_source_file = b.path("src/c.h"),
    .target = target,
    .optimize = optimize,
});
exe.root_module.addImport("c", translate_c.createModule());
exe.linkLibC();
exe.linkSystemLibrary("sqlite3");
```

**Source:**
```zig
const c = @import("c");
const result = c.printf("Hello from C!\n");
```

### Legacy (deprecated): Inline `@cImport`

```zig
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("math.h");
});

const result = c.printf("Hello from C!\n");
const angle = c.atan2(y, x);

exe.linkLibC();
exe.linkSystemLibrary("sqlite3");
```

Note: `@cImport` is evaluated at comptime, but caching and cross-compilation are more deterministic via the build-system path.

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

`async` and `await` **are** language features in 0.16 (restored after being absent in 0.12–0.15). They work together with `std.Io`: the runtime that actually schedules/awaits the work is the `std.Io` implementation, while the `async`/`await` surface is part of the language. What's still gone is the 0.11-era `suspend`/`resume`/`nosuspend` frame-based system — that is NOT coming back.

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

### Queue — Bounded Thread-Safe Channel

`std.Io.Queue` is the producer/consumer primitive for passing data between tasks. It's a bounded channel backed by a caller-provided buffer.

```zig
// Create a queue with space for 16 elements
var backing: [16]u32 = undefined;
var queue: std.Io.Queue(u32) = .init(&backing);

// Producer task:
try queue.putOne(io, value);       // blocks if queue is full
queue.putOneUncancelable(io, v) catch return;  // can't be canceled

// Consumer task:
const val = try queue.getOne(io);  // blocks if queue is empty
// Returns error.Closed when queue is drained and closed
// Returns error.Canceled if task is canceled

// Signal no more data:
queue.close(io);
```

**Pattern: producer/consumer with Group:**
```zig
var group: std.Io.Group = .init;
group.async(io, producer, .{ io, &queue });
group.async(io, consumer, .{ io, &queue });
try group.await(io);
```

Consumer drains with a `while (true)` loop, breaking on `error.Closed`:
```zig
fn consumer(io: std.Io, queue: *std.Io.Queue(u32)) void {
    while (true) {
        const value = queue.getOne(io) catch |err| switch (err) {
            error.Closed => break,
            error.Canceled => return,
        };
        // process value
    }
}
```

### Mutex — Protecting Shared State

```zig
var mutex: std.Io.Mutex = .init;

// In a task:
try mutex.lock(io);        // blocks until acquired; cancellation point
defer mutex.unlock(io);    // NOTE: unlock also takes io
// ... critical section ...

// Non-blocking alternative:
if (mutex.tryLock()) {
    defer mutex.unlock(io);
    // ... critical section ...
}
```

**Important:** `mutex.lock(io)` is a cancellation point — it can return `error.Canceled`. And `mutex.unlock(io)` takes `io` as a parameter (unlike some other lock APIs).

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
9. **`Container(T).init(alloc)` for any std container** → Use `= .empty` and pass allocator per-operation. Applies to `ArrayList`, `AutoHashMap`, `StringHashMap`, `MultiArrayList`, `PriorityQueue`, `SinglyLinkedList`, `DoublyLinkedList`, etc. (see §1.10)
10. **`std.os.`** → Use `std.posix.`
11. **`std.rand.`** → Use `std.Random.`
12. **`std.heap.page_allocator` in main** → Prefer `init.gpa` from `std.process.Init`
13. **`std.heap.GeneralPurposeAllocator`** → Renamed to `std.heap.DebugAllocator`. Prefer `init.gpa` in main.
14. **`usingnamespace`** → Permanently removed from the language
15. **`std.io.Writer` / `std.io.Reader`** (lowercase) → Use `std.Io.Writer` / `std.Io.Reader` (capital `I`, vtable-based)
16. **`BufferedWriter`** → Buffering is now built into `std.Io.Writer`
17. **Old `format` signature with `comptime fmt`** → New: `fn format(self: @This(), writer: *std.Io.Writer) std.Io.Writer.Error!void` — invoked via `{f}` not `{}`
18. **`@Type(.{ .int = ... })`** → Use `@Int(.unsigned, 10)` and friends (see Section 1.14). `@Type` is gone in 0.16.
19. **`packed union { ... }` without backing int** → Must be `packed union(u16) { ... }` in 0.16.
20. **`@intFromFloat(@round(x))`** → Just `const y: u8 = @round(x)` — `@round`/`@floor`/`@ceil`/`@trunc` convert to int directly when destination is an int type.
21. **`std.Thread.Mutex` / `std.Thread.ResetEvent` / `std.Thread.Condition` / `std.Thread.Semaphore`** → Moved to `std.Io.Mutex` / `std.Io.Event` / `std.Io.Condition` / `std.Io.Semaphore`
22. **`std.once(...)`** → Removed. Restructure to avoid global lazy init, or use explicit init functions.
23. **`std.Thread.Pool`** → Removed. Use `io.async` / `io.concurrent` / `std.Io.Group` from `std.Io`.
24. **`std.process.Child.init(argv, gpa)` + `.spawn(io)`** → Use `std.process.spawn(io, .{ .argv = argv, ... })`
25. **`error.RenameAcrossMountPoints` / `error.SharingViolation` / `error.EnvironmentVariableNotFound`** → Renamed to `error.CrossDevice` / `error.FileBusy` / `error.EnvironmentVariableMissing`
26. **`@cImport` in `.zig` files** → **Deprecated in 0.16.** Use `b.addTranslateC(.{...})` in `build.zig` and `@import("c")` in source.
27. **`std.fmt.FormatOptions`** → Renamed to `std.fmt.Options`. Also: `std.fmt.Formatter` → `std.fmt.Alt`; `std.fmt.format(w, ...)` → `std.Io.Writer.print(w, ...)`; `bufPrintZ` → `bufPrintSentinel`.
28. **`std.Io.GenericReader` / `AnyReader` / `GenericWriter` / `AnyWriter` / `FixedBufferStream` / `CountingReader` / `null_writer`** → All removed. Use non-generic `std.Io.Reader` / `std.Io.Writer` + `std.Io.Writer.fixed` / `.Discarding` / `.Allocating`.
29. **`std.Thread.WaitGroup` / `Futex` / `RwLock` / `Mutex.Recursive`** → Use `std.Io.Group` / `std.Io.Futex` / `std.Io.RwLock`; recursive mutex is gone (redesign).
30. **`std.heap.ThreadSafeAllocator`** → Removed. `std.heap.ArenaAllocator` is now thread-safe + lock-free on its own; `init.gpa` is also thread-safe.
31. **`std.process.Child.run`** → `std.process.run`. **`std.process.execv`** → `std.process.replace`. **`std.fs.selfExePath*` / `fs.openSelfExe` / `fs.Dir.setAsCwd`** → Moved to `std.process.executablePath*` / `openExecutable` / `setCurrentDir`.
32. **`file.read(buf)` / `file.pread(...)` / `file.seekTo(...)` / `file.getEndPos()`** → `file.readStreaming(io, buf)` / `file.readPositional(io, buf, off)` / `file.reader(io, &buf).seekTo(...)` / `file.length(io)`. Similar split for writes. See §1.25.
33. **Runtime vector indexing (`v[i]` with runtime `i`)** → Forbidden in 0.16. Convert to array first or use `@shuffle`/`@reduce`.
34. **Pointer casts between `*[N]T` and `*@Vector(N, T)`** → No longer allowed; layouts are no longer guaranteed equivalent.
35. **`std.SegmentedList`** → Removed. Use `std.ArrayList = .empty`, or your own chunked container if you needed stable addresses.
36. **`std.meta.declList`** → Removed. Iterate `@typeInfo(T).@"struct".decls` directly.
37. **Implicit `comptime` on zero-bit tuple fields** → Gone; annotate `comptime` explicitly if you need it.
38. **Returning `&local` for a trivial local** → Compile error in 0.16 (would dangle). Return by value, or have the caller pass in a buffer.
39. **`std.DynLib` on Windows** → Windows support removed from that API.
40. **Assuming `std.math.sign(x)` returns `i2` or matches input type** → It now returns the smallest int type that fits; check the actual return type.

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
10. `io.async(fn, .{args})` / `future.await(io)` — `async` and `await` are real 0.16 language features, backed by `std.Io` at runtime
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
| 2025-08 | **0.15.1 stable** | async/await temporarily absent (restored in 0.16 — see timeline below); `std.Io.Writer`/`Reader` vtable-based; non-generic I/O |
| 2025-10 | **0.15.2 stable** | Latest stable release |
| 2025-12 | 0.16.0-dev.1859 | File system I/O integrated with `std.Io` |
| 2026-01 | 0.16.0-dev.1976 | Process API moved to `std.Io`; `main(init)` pattern |
| 2026-01 | 0.16.0-dev.2075 | Randomness API moved to `std.Io` |
| 2026-02 | 0.16.0-dev.2471 | `process.Child.Cwd` added |
| 2026-03 | 0.16.0-dev.2915 | `GeneralPurposeAllocator` renamed to `DebugAllocator` |
| 2026-04 | **0.16.0 stable** | `@Type` removed (→ `@Int`/`@Struct`/`@Union`/`@Enum`/etc.); packed union backing-int required; `@round`/`@floor`/`@ceil`/`@trunc` direct-to-int; small int→float coercion; unary float builtins (`@sqrt`/`@sin`/`@cos`/...) forward result types; **vector runtime indexing forbidden**, vector↔array in-memory coercion removed; `*T` vs `*align(1) T` distinct; `std.Thread.{Mutex,ResetEvent,Condition,Semaphore,RwLock,WaitGroup,Futex}` → `std.Io.*`; `std.once` / `Thread.Pool` / `Thread.Mutex.Recursive` / `ThreadSafeAllocator` removed; `ArenaAllocator` thread-safe/lock-free; `std.fmt.FormatOptions`→`Options`, `Formatter`→`Alt`, `fmt.format`→`Writer.print`, `bufPrintZ`→`bufPrintSentinel`; `Io.{GenericReader,AnyReader,FixedBufferStream,GenericWriter,AnyWriter,null_writer,CountingReader}` removed; `std.process.spawn`/`run`/`replace`/`executablePath*`/`openExecutable`/`setCurrentDir` consolidation; file API split into streaming vs positional, seeks moved to `Reader`, `Mode`→`Permissions`; error renames (`CrossDevice`, `FileBusy`, `EnvironmentVariableMissing`, `DirNotEmpty`); switch prongs accept packed structs & decl literals; `@cImport` **deprecated** (use `b.addTranslateC`); `std.SegmentedList` / `std.meta.declList` removed; `std.math.sign` returns smallest fitting int; `std.DynLib` Windows removed; zero-bit tuple fields no longer implicit-comptime; returning `&trivial_local` forbidden; built-in deflate (faster than zlib); maccatalyst targets added, Solaris/AIX/z/OS dropped |

---

## 24. How to Update This Document

> **For LLMs**: When a user says "update your Zig knowledge" or "refresh zig_context.md", follow this procedure to bring this document up to date with the latest Zig changes.

### Step-by-Step Update Procedure

#### Step 1: Determine the Current Version Target

Check what version of Zig the user is targeting. Look for:
- `build.zig` in their project — search for a version string like `"0.16.0-dev.2471"`
- Or ask: "What version of Zig are you using?" (`zig version`)

The version this document was last updated for: **0.16.0 stable** (April 2026).

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
- **0.16.0 Release Notes** (latest stable, April 2026): https://ziglang.org/download/0.16.0/release-notes.html
- **0.15.1 Release Notes** (prior major breaking changes): https://ziglang.org/download/0.15.1/release-notes.html
- **Ziglings Exercises**: https://codeberg.org/ziglings/exercises
- **Build from Source**: https://codeberg.org/ziglang/zig#building-from-source
- **Zig Devlog**: https://ziglang.org/devlog/2026/
