# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a central knowledge base -- a collection of detailed reference documents that provide Claude Code with accurate, up-to-date domain knowledge for working across multiple projects on this machine. There is no buildable code here; the repo contains only markdown context files.

When opened, Claude should be ready to assist with any of the user's projects using the knowledge stored here.

## Context Documents

### Project-Specific
- **pitchfreq.md** -- Full project context for pitchfreq: architecture, build order, processing pipeline, quality targets, Zig 0.16 quick-reference, and FFT library decision. **Read this before working on pitchfreq.**

### Language & SDK References
- **zig.md** -- Zig 0.16.0 language reference. Covers breaking changes from 0.11-0.13 era (builtin renames, `std.Io` rewrite, single-arg casts), build system, C interop, and standard library. Critical for generating correct modern Zig code.
- **ptsl.md** -- Zig/Rust FFI wrapper for Avid Pro Tools' gRPC scripting API (PTSL). Covers the architecture (Zig -> libptsl.dylib Rust layer -> Pro Tools gRPC), all 159 commands, memory model (arena allocators), and build setup.
- **reaper.md** -- REAPER C/C++ extension plugin SDK. Covers plugin lifecycle, `plugin_register` system, action/menu/SWELL APIs, and cross-platform gotchas (especially macOS submenu insertion via `InsertMenu` + `MF_POPUP`).

### DSP Algorithm References
- **pitch-shifting-algorithms.md** -- Implementation reference for phase vocoder, WSOLA, and hybrid time-stretch/pitch-shift algorithms. Covers FFT library choices (pffft for commercial use), phase locking, transient detection, and resampling.
- **phase-vocoder-deep-dive.md** -- Detailed pseudocode, formulas, and complete Python reference implementation of a phase-locked vocoder with transient detection.

## Key Projects

- **pitchfreq** (`~/dev/pitchfreq`) -- Pro-quality pitch shifting audio plugin in Zig 0.16. Long-term goal: ARA plugin for Pro Tools replacing Pitch 'n Time. Currently building core DSP engine as a CLI tool. Build/run: `zig build`, `zig build run`, `zig build test`.

## Shell Environment

- `grep` is aliased to `rg` (ripgrep). Use ripgrep flags/syntax when running grep commands (e.g. `grep -rn 'pattern'` won't work the same -- use `grep 'pattern' path/` or `rg 'pattern' path/`).

## Working With These Files

- These documents are meant to be **read and referenced**, not compiled or tested.
- When editing, preserve the markdown structure and technical accuracy -- these files are fed directly to LLMs as context windows.
- The zig.md file is large (~26k tokens). Read specific sections rather than the whole file.
- Primary platform is **macOS**. Windows/Linux are secondary.
- Audio code targets commercial plugin distribution -- GPL libraries are not viable without commercial licenses.
