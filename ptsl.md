# PTSL Library Context

> **Purpose**: This document describes the `ptsl` library — a Rust+Zig FFI wrapper around Avid Pro Tools' gRPC scripting API (PTSL). Use this when writing Zig code that communicates with Pro Tools.

---

## Architecture

```
┌──────────────┐     JSON strings     ┌──────────────┐     gRPC      ┌───────────┐
│  Your Zig    │ ◄──────────────────► │  libptsl.dylib│ ◄──────────► │ Pro Tools  │
│  Program     │   ptsl_send/free     │  (Rust/Tonic) │  localhost    │           │
└──────────────┘                      └──────────────┘   :31416      └───────────┘
```

- **Rust layer** (`libptsl.dylib`): Manages gRPC connection, auto-connects on first send, auto-reconnects on error. Exposes 3 C FFI functions. Takes a command ID + JSON string in, returns JSON string out. That's it.
- **Zig layer** (`proto/ptsl.zig`): Provides `CommandId` enum, all proto message structs, typed send wrappers, and JSON serialization/deserialization. This is what you import and use.

---

## Quick Start (Zig)

```zig
const std = @import("std");
const ptsl = @import("ptsl");

pub fn main() !void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const allocator = arena.allocator();

    // Connect (optional — send auto-connects)
    try ptsl.connect();

    // Typed API — commands with no request body
    const name = try ptsl.getSessionName(allocator);
    std.debug.print("Session: {s}\n", .{name.value.session_name orelse "none"});

    // Typed API — commands with a request body
    const tracks = try ptsl.getTrackList(allocator, .{
        .page_limit = 100,
    });
    if (tracks.value.track_list) |list| {
        for (list) |track| {
            std.debug.print("Track: {s}\n", .{track.name orelse "(unnamed)"});
        }
    }

    // Raw API — send JSON directly, get JSON back
    const raw = try ptsl.send(allocator, .GetSessionName, "{}");
    std.debug.print("{s}\n", .{raw});
}
```

---

## API Reference

### Connection

```zig
try ptsl.connect();  // error{ConnectionFailed}
```

Explicitly opens gRPC connection to `localhost:31416` and registers. Optional — all send functions auto-connect on first call and auto-reconnect on transport error.

### Raw Send

```zig
const json = try ptsl.send(allocator, cmd, req_json);
// json is allocator-owned []u8, caller frees (or use arena)
```

- `cmd`: `ptsl.CommandId` enum value
- `req_json`: `[]const u8` JSON request body (use `"{}"` for commands with no body)
- Returns: JSON response string
- Errors: `SendError{ SendFailed, OutOfMemory }`

### Typed Send (Low-Level)

```zig
const result = try ptsl.sendTyped(allocator, .GetSessionName, ptsl.Empty{}, ptsl.GetSessionNameResponseBody);
// result is std.json.Parsed(ResponseType)
// result.value has the deserialized struct
```

### Per-Command Wrappers (Preferred)

Every PTSL command has a convenience wrapper. There are 159 commands in 4 categories:

**Category A — request body + response body:**
```zig
const resp = try ptsl.getTrackList(allocator, .{ .page_limit = 50 });
// resp.value is GetTrackListResponseBody
```

**Category B — request body only (response is empty):**
```zig
const resp = try ptsl.renameTargetTrack(allocator, .{ .new_name = "My Track" });
```

**Category C — response body only (no request needed):**
```zig
const resp = try ptsl.getSessionName(allocator);
// resp.value.session_name is ?[]const u8
```

**Category D — neither (fire-and-forget):**
```zig
const resp = try ptsl.cut(allocator);
```

All wrappers return `std.json.Parsed(ResponseType)`.

---

## Memory Model

**Always use an arena allocator.** The typed API returns `std.json.Parsed` which creates its own internal arena, but parsed string values are zero-copy references into the JSON response buffer. The response buffer lives in the allocator you provide. An arena ensures everything is cleaned up together.

```zig
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();
const allocator = arena.allocator();

// All ptsl calls use this allocator — arena.deinit() frees everything
const name = try ptsl.getSessionName(allocator);
const tracks = try ptsl.getTrackList(allocator, .{});
```

If you need per-call cleanup (rare), call `.deinit()` on the `Parsed` result — but note this won't free the underlying JSON buffer, only the parsed arena. Prefer a top-level arena.

---

## Important: Proto Enum Fields Are Strings

Pro Tools serializes protobuf enum values as their **string names**, not integers. For example:

```json
{"sample_rate": "SR_48000"}
{"current_setting": "WAV"}
{"current_setting": "Bars_Beats"}
```

The Zig structs use `?[]const u8` for these fields, not `?i32`. When sending requests that take enum values, send the string name.

---

## Command List (All 159)

The `CommandId` enum has values 0–159 (value 60 is reserved/skipped):

| ID | Command | Category |
|----|---------|----------|
| 0 | CreateSession | B |
| 1 | OpenSession | B |
| 2 | Import | A |
| 3 | GetTrackList | A |
| 12 | GetTaskStatus | A |
| 17 | CloseSession | D |
| 18 | SaveSession | D |
| 20–23 | Cut/Copy/Paste/Clear | D |
| 28 | ExportMix | B |
| 34–45 | GetSession* (AudioFormat, SampleRate, BitDepth, Name, Path, etc.) | C |
| 46–54 | SetSession* | B |
| 55 | GetPtslVersion | C |
| 56–59 | GetPlaybackMode/RecordMode/TransportArmed/TransportState | C |
| 69 | GetMemoryLocations | A |
| 70 | RegisterConnection | (internal, handled by connect()) |
| 72 | CreateNewTracks | A |
| 82 | GetTimelineSelection | C |
| 97 | GetSessionIDs | C |
| 104–108 | Undo/Redo/UndoAll/RedoAll/ClearUndoQueue | D |
| 118–122 | Time math commands | A |
| 125 | GetClipList | A |
| 131 | GetEditSelection | C |
| 148 | GetTrackControlInfo | A |
| 154 | GetTrackPlaylists | A |
| 157 | DeleteTracks | A |
| 158 | GetPlaylistElements | A |

See `proto/ptsl.zig` for the full `CommandId` enum and all struct definitions.

---

## Build Setup (Zig)

### Prerequisites

1. Build the Rust library first:
   ```sh
   cd /path/to/ptsl
   cargo build --release
   ```
   This produces `target/release/libptsl.dylib` (macOS).

2. Pro Tools must be running with PTSL enabled.

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const ptsl_mod = b.createModule(.{
        .root_source_file = b.path("path/to/proto/ptsl.zig"),
        .target = target,
        .optimize = optimize,
    });

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
            .imports = &.{
                .{ .name = "ptsl", .module = ptsl_mod },
            },
        }),
    });

    // Link the Rust dylib
    exe.root_module.addLibraryPath(b.path("path/to/ptsl/target/release"));
    exe.root_module.linkSystemLibrary("ptsl", .{});
    exe.root_module.addRPath(b.path("path/to/ptsl/target/release"));

    b.installArtifact(exe);
}
```

---

## Common Struct Shapes

### Track
```zig
pub const Track = struct {
    name: ?[]const u8 = null,
    id: ?[]const u8 = null,
    index: ?i32 = null,
    color: ?[]const u8 = null,
    track_attributes: ?TrackAttributes = null,
    format: ?i32 = null,
    timebase: ?i32 = null,
};
```

### MemoryLocation
```zig
pub const MemoryLocation = struct {
    number: ?i32 = null,
    name: ?[]const u8 = null,
    start_time: ?[]const u8 = null,
    end_time: ?[]const u8 = null,
    time_properties: ?i32 = null,
    reference: ?i32 = null,
    general_properties: ?i32 = null,
    comments: ?[]const u8 = null,
    color_index: ?i32 = null,
    location: ?i32 = null,
};
```

### All fields are optional

Every struct field is `?T = null`. Pro Tools only populates fields it has data for. Always use `orelse` or `if` to handle nulls:

```zig
const name = track.name orelse "(unnamed)";
if (track.index) |idx| {
    // use idx
}
```

---

## File Locations

```
ptsl/
├── proto/
│   ├── PTSL.proto       ← Avid's proto definition (read-only reference, 13126 lines)
│   └── ptsl.zig         ← Zig bindings: CommandId, all structs, send functions
├── src/
│   └── lib.rs           ← Rust: Session, gRPC, FFI exports
├── target/
│   └── release/
│       └── libptsl.dylib
├── Cargo.toml
└── zig/                  ← Example/test program
    ├── build.zig
    ├── build.zig.zon
    └── src/
        └── main.zig
```

---

## Zig Version

This library targets **Zig 0.16.0-dev**. Key 0.16 APIs used:
- `std.Io.Writer.Allocating` for JSON serialization buffers
- `std.json.Stringify.value()` for struct-to-JSON
- `std.json.parseFromSlice()` / `std.json.Parsed` for JSON-to-struct
- `std.heap.ArenaAllocator` with `.init` / `.allocator()` / `.deinit()`

Do NOT use old Zig patterns (pre-0.14 ArrayList, old io, etc.). See `zig_context.md` for full 0.16 migration guide.
