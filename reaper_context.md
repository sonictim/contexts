# REAPER Extension SDK Context for LLMs

> **Purpose**: Comprehensive reference for building REAPER C/C++ extension plugins. Covers the plugin lifecycle, registration system, menu hooks, action system, SWELL (cross-platform Win32 on macOS/Linux), and common patterns. Use this document to understand how REAPER extensions work under the hood.

> **Sources**: [reaper-sdk](https://github.com/justinfrankel/reaper-sdk), [reaper_plugin.h](https://www.reaper.fm/sdk/plugin/reaper_plugin.h), [reaper_plugin_functions.h](https://www.reaper.fm/sdk/plugin/reaper_plugin_functions.h), [SWS Extension source](https://github.com/reaper-oss/sws), [REAPER Extensions SDK page](https://www.reaper.fm/sdk/plugin/plugin.php), [X-Raym ReaScript docs](https://www.extremraym.com/cloud/reascript-doc/).

---

## Table of Contents

1. [Plugin Lifecycle](#1-plugin-lifecycle)
2. [Registration System (plugin_register)](#2-registration-system-plugin_register)
3. [Action System](#3-action-system)
4. [Menu System](#4-menu-system)
5. [SWELL (macOS/Linux GUI)](#5-swell-macoslinux-gui)
6. [Extended State (Persistent Storage)](#6-extended-state-persistent-storage)
7. [Common REAPER API Functions](#7-common-reaper-api-functions)
8. [Opaque Types](#8-opaque-types)
9. [Key Struct Definitions](#9-key-struct-definitions)
10. [REAPER Info Value Strings](#10-reaper-info-value-strings)
11. [Patterns and Gotchas](#11-patterns-and-gotchas)

---

## 1. Plugin Lifecycle

### Entry Point

Every REAPER extension DLL/dylib must export:

```c
int ReaperPluginEntry(HINSTANCE hInstance, reaper_plugin_info_t *rec);
```

- **Loading**: REAPER calls this with a valid `rec` pointer. Return `1` to stay loaded, `0` to unload.
- **Unloading**: REAPER calls this again with `rec == NULL`. Clean up resources and return `0`.
- **Version check**: Compare `rec->caller_version` against `REAPER_PLUGIN_VERSION` (currently `0x20E`). If they don't match, return `0` — the plugin is incompatible.

### Initialization Order

A typical extension entry point should:

1. Check `rec != NULL` and version compatibility
2. Import API functions via `rec->GetFunc("FunctionName")`
3. Register command hooks (`hookcommand2`)
4. Register toggle action callback (`toggleaction`)
5. Register custom actions (`custom_action`)
6. Register menu hook (`hookcustommenu`)
7. Call `AddExtensionsMainMenu()` to trigger menu creation
8. Return `1`

### reaper_plugin_info_t

```c
typedef struct reaper_plugin_info_t {
    int caller_version;  // REAPER_PLUGIN_VERSION
    HWND hwnd_main;      // Main REAPER window handle
    int (*Register)(const char *name, void *infostruct);
    void *(*GetFunc)(const char *name);
} reaper_plugin_info_t;
```

- `Register()` — register hooks, actions, surfaces, etc. Returns command ID for `custom_action`, or 1/0 for success/failure for other types.
- `GetFunc()` — retrieve any REAPER API function pointer by name. Returns `NULL` if not found.

### Importing API Functions

```c
// Manual import
ShowConsoleMsg = (void(*)(const char*))rec->GetFunc("ShowConsoleMsg");

// Or using the auto-import macro pattern from the SDK:
IMPAPI(ShowConsoleMsg);
```

In the Zig extension, this is done via comptime reflection over all `pub var` function pointer declarations — see `reaper.zig:init()`.

---

## 2. Registration System (plugin_register)

`plugin_register(name, infostruct)` is the central mechanism for extending REAPER. The `name` string determines what's being registered, and `infostruct` is cast to the appropriate type.

**Prefixing with `-`** unregisters (e.g., `plugin_register("-timer", fn_ptr)` removes a timer).

### Complete Registration String Reference

| String | infostruct Type | Description |
|--------|----------------|-------------|
| `"hookcommand"` | `bool (*)(int cmd, int flag)` | Hook called before every action in main section. Return `true` to eat the command. |
| `"hookcommand2"` | `char (*)(KbdSectionInfo*, int cmd, int val, int valhw, int relmode, HWND hwnd)` | Hook called before every action from key/MIDI input. Return `1` to eat, `0` to pass through. More flexible than `hookcommand`. |
| `"hookpostcommand"` | `void (*)(int cmd, int flag)` | Called after each action in main section completes. |
| `"hookpostcommand2"` | `void (*)(int section, int cmd, int val, int valhw, int relmode, HWND hwnd)` | Called after each action from key/MIDI. |
| `"hookcustommenu"` | `void (*)(const char* menuidstr, HMENU menu, int flag)` | Menu hook — see [Menu System](#4-menu-system) below. |
| `"custom_action"` | `custom_action_register_t*` | Register an action into the action list. Returns the command ID. |
| `"toggleaction"` | `int (*)(int cmd)` | Callback to query toggle state. Return: `-1` = not ours/no toggle, `0` = off, `1` = on. |
| `"timer"` | `void (*)()` | Register a function to be called periodically (~30ms). Use `-timer` to unregister. |
| `"accelerator"` | `accelerator_register_t*` | Hook into keyboard processing queue. |
| `"gaccel"` | `gaccel_register_t*` | Register action with a default key binding into main section. |
| `"accel_section"` | `KbdSectionInfo*` | Register a custom keyboard/action section. |
| `"csurf"` | `reaper_csurf_reg_t*` | Register a control surface type (appears in Preferences). |
| `"csurf_inst"` | `IReaperControlSurface*` | Add a control surface instance behind the scenes. |
| `"API_funcname"` | `void*` | Expose a C function to other extensions and ReaScript. |
| `"APIdef_funcname"` | `const char*` | Define ReaScript signature (return type, arg types, arg names, help text — null-separated). |
| `"action_help"` | See docs | Register help text for an action. |
| `"prefpage"` | See docs | Add a preferences page. |
| `"projectimport"` | See docs | Project format converter. |
| `"projectconfig"` | See docs | Extend project configuration. |
| `"editor"` | See docs | External editor integration. |
| `"pcmsrc"` | `pcmsrc_register_t*` | Custom audio source decoder. |
| `"pcmsink"` | `pcmsink_register_t*` | Custom audio output/encoder. |
| `"file_in_project_ex"` | See docs | Register files used in project. |
| `"toolbar_icon_map"` | See docs | Map action IDs to toolbar icons. |
| `"atexit"` | `void (*)()` | Called when REAPER is about to quit (before main window destroyed). |

---

## 3. Action System

### Registering Custom Actions

```c
custom_action_register_t action = {
    .section = 0,           // 0 = main section
    .id_str = "_MY_ACTION", // unique string ID (must start with _)
    .name = "My Action",    // display name in action list
    .extra = NULL            // reserved
};
int cmd_id = plugin_register("custom_action", &action);
```

**Section IDs**:
- `0` or `100` — Main / Main (alt recording)
- `32060` — MIDI Editor
- `32063` — Media Explorer

### custom_action_register_t

```c
typedef struct {
    int uniqueSectionId;     // section to register into
    const char* idStr;       // unique string identifier (persistent across sessions)
    const char* name;        // display name shown in Actions list
    void *extra;             // reserved, set to NULL
} custom_action_register_t;
```

### Looking Up Command IDs

```c
int cmd_id = NamedCommandLookup("_MY_ACTION");
// Returns the runtime integer command ID for the named action
// Returns 0 if not found
```

The string ID (e.g., `"_MY_ACTION"`) is persistent across sessions. The integer command ID is NOT persistent — it's assigned at runtime. Always store/reference actions by their string ID.

### hookcommand2 Callback

```c
// Return 1 to indicate we handled the command, 0 to pass through
char hookcommand2(KbdSectionInfo *sec, int command, int val, int valhw, int relmode, HWND hwnd) {
    if (command == my_cmd_id) {
        doMyAction();
        return 1;
    }
    return 0; // not our command
}
```

### Toggle Actions

Register a toggle callback:

```c
int toggleActionCallback(int cmd) {
    if (cmd == my_cmd_id) return my_state ? 1 : 0;
    return -1; // not our action
}
plugin_register("toggleaction", (void*)toggleActionCallback);
```

REAPER calls this whenever it needs to know the on/off state (for checkmarks in menus, toolbar button states, etc.).

### Timer (Deferred/Periodic Execution)

```c
void myTimerFunc() {
    // Called approximately every 30ms
    // Unregister when done:
    if (done) plugin_register("-timer", (void*)myTimerFunc);
}
plugin_register("timer", (void*)myTimerFunc);
```

Timers run on the main thread. They are the primary mechanism for deferred or periodic actions.

---

## 4. Menu System

### Overview

REAPER's menu system works through a hook-based callback model:

1. Extension calls `AddExtensionsMainMenu()` — tells REAPER "I want to add to the Extensions menu"
2. REAPER calls your registered `hookcustommenu` callback when it needs to build/show menus
3. Your callback populates the menu using Win32/SWELL menu functions

### AddExtensionsMainMenu

```c
bool AddExtensionsMainMenu();
```

Call this during plugin initialization. It ensures the "Extensions" top-level menu exists in REAPER's menu bar. Without this call, your `hookcustommenu` callback won't be triggered for `"Main extensions"`.

### AddCustomizableMenu

```c
bool AddCustomizableMenu(
    const char* menuidstr,    // unique ID string (must be persistent pointer)
    const char* menuname,     // display name for main menu bar (NULL for non-main menus)
    const char* kbdsecname,   // KbdSectionInfo name, or NULL for main section
    bool addtomainmenu        // true to add as top-level menu in menu bar
);
```

Use this to create an entirely new top-level menu (not just a submenu of Extensions). The `menuidstr` is what your `hookcustommenu` callback will receive.

### hookcustommenu Callback

```c
void menuHook(const char* menuidstr, HMENU menu, int flag);
```

**Parameters**:
- `menuidstr` — identifies which menu is being built/shown (see table below)
- `menu` — the Win32/SWELL HMENU handle to populate
- `flag` — `0` = initialization, `1` = about to show

**Flag = 0 (Initialization)**:
- Called once when the menu is first initialized (before first display)
- Add menu items and submenus here — they become part of the "default" menu structure
- The user can customize these items (reorder, remove, etc.)
- **DO NOT** set checked/grayed state during flag=0
- **DO NOT** add menu icons during flag=0
- **DO NOT** retain handles to submenus you create — REAPER owns them after insertion

**Flag = 1 (About to Show)**:
- Called every time the menu is about to be displayed
- Set checked/grayed state here
- Add dynamic items here
- Can add icons here

### Known menuidstr Values

#### Main Menus
| menuidstr | Menu |
|-----------|------|
| `"Main file"` | File menu |
| `"Main edit"` | Edit menu |
| `"Main view"` | View menu |
| `"Main insert"` | Insert menu |
| `"Main item"` | Item menu |
| `"Main track"` | Track menu |
| `"Main options"` | Options menu |
| `"Main actions"` | Actions menu |
| `"Main extensions"` | Extensions menu |

#### Context Menus
| menuidstr | Context |
|-----------|---------|
| `"Ruler context"` | Ruler/arrange ruler |
| `"Track control panel context"` | TCP right-click |
| `"Empty TCP context"` | Empty area of TCP |
| `"Media item context"` | Item right-click |
| `"Envelope context"` | Envelope right-click |
| `"Envelope point context"` | Envelope point right-click |
| `"Mixer context"` | Mixer panel |
| `"FX extended mixer context"` | FX section of mixer |
| `"Sends extended mixer context"` | Sends section of mixer |
| `"Transport context"` | Transport bar |
| `"Automation item context"` | Automation item right-click |

#### Toolbars
| menuidstr | Toolbar |
|-----------|---------|
| `"Main toolbar"` | Main toolbar |
| `"Floating toolbar 1"` through `"Floating toolbar 7"` | Floating toolbars |

### Menu Building Pattern (C++)

```c
void menuHook(const char* menuidstr, HMENU menu, int flag) {
    if (flag != 0) return; // only handle initialization

    if (strcmp(menuidstr, "Main extensions") == 0) {
        // Create submenu, insert into parent, get handle back
        HMENU submenu = addMenu(menu, "My Extension");
        // Populate the submenu
        addAction(submenu, "Action 1", "_MYEXT_ACTION1");
        addSeparator(submenu);
        addAction(submenu, "Action 2", "_MYEXT_ACTION2");
    }
}

HMENU addMenu(HMENU parent, const char* label) {
    // NOTE: Must use InsertMenu + MF_POPUP on macOS/SWELL (not InsertMenuItem + MIIM_SUBMENU)
    HMENU sub = CreatePopupMenu();
    int pos = GetMenuItemCount(parent);
    InsertMenu(parent, pos, MF_BYPOSITION | MF_POPUP | MF_STRING, (UINT_PTR)sub, label);
    return sub;
}

void addAction(HMENU menu, const char* label, const char* command) {
    int pos = GetMenuItemCount(menu);
    MENUITEMINFO mii = {0};
    mii.cbSize = sizeof(mii);
    mii.fMask = MIIM_TYPE | MIIM_ID;
    mii.fType = MFT_STRING;
    mii.dwTypeData = (char*)label;
    mii.wID = NamedCommandLookup(command);
    InsertMenuItem(menu, pos, TRUE, &mii);
}

void addSeparator(HMENU menu) {
    int pos = GetMenuItemCount(menu);
    MENUITEMINFO mii = {0};
    mii.cbSize = sizeof(mii);
    mii.fMask = MIIM_TYPE;
    mii.fType = MFT_SEPARATOR;
    InsertMenuItem(menu, pos, TRUE, &mii);
}
```

### Menu Checked/Grayed State (flag=1 only)

```c
void menuHook(const char* menuidstr, HMENU menu, int flag) {
    if (flag == 1) {
        // Recursively walk menu items, check state
        MENUITEMINFO mi = {sizeof(MENUITEMINFO)};
        mi.fMask = MIIM_ID | MIIM_SUBMENU;
        for (int i = 0; i < GetMenuItemCount(menu); i++) {
            GetMenuItemInfo(menu, i, TRUE, &mi);
            if (mi.hSubMenu)
                menuHook(menuidstr, mi.hSubMenu, flag); // recurse
            else if (mi.wID == my_cmd_id)
                CheckMenuItem(menu, i, MF_BYPOSITION |
                    (my_state ? MF_CHECKED : MF_UNCHECKED));
        }
    }
}
```

---

## 5. SWELL (macOS/Linux GUI)

SWELL (Simple Windows Emulation Layer) is Cockos' cross-platform implementation of Win32 GUI APIs. On macOS, REAPER provides SWELL functions that extensions link against.

### How SWELL Linking Works

On macOS, REAPER's application binary exports SWELL functions. Extensions use `swell-modstub.mm` (provided in the SDK) which:
1. At dylib load time, finds the REAPER app's ObjC delegate
2. Loads all SWELL function pointers from it
3. Exposes them as global symbols that your code can call

The SWELL functions are NOT retrieved via `plugin_getapi`/`GetFunc()`. They're linked at load time by the modstub.

### Key SWELL Menu Functions

```c
HMENU CreatePopupMenu();
void DestroyMenu(HMENU menu);
int GetMenuItemCount(HMENU menu);
BOOL InsertMenuItem(HMENU menu, UINT item, BOOL fByPosition, MENUITEMINFO *mii);
BOOL GetMenuItemInfo(HMENU menu, UINT item, BOOL fByPosition, MENUITEMINFO *mii);
BOOL SetMenuItemInfo(HMENU menu, UINT item, BOOL fByPosition, MENUITEMINFO *mii);
BOOL CheckMenuItem(HMENU menu, UINT item, UINT flags);
int TrackPopupMenu(HMENU menu, UINT flags, int x, int y, int reserved, HWND hwnd, const RECT *rc);
```

### MENUITEMINFO Structure

```c
typedef struct {
    UINT cbSize;          // sizeof(MENUITEMINFO)
    UINT fMask;           // which fields are valid
    UINT fType;           // item type
    UINT fState;          // item state
    UINT wID;             // command ID
    HMENU hSubMenu;       // submenu handle (NULL if not a submenu)
    HBITMAP hbmpChecked;  // not commonly used
    HBITMAP hbmpUnchecked; // not commonly used
    ULONG_PTR dwItemData; // application-defined data
    char *dwTypeData;     // item text (for MFT_STRING)
    UINT cch;             // text length
    HBITMAP hbmpItem;     // not commonly used
} MENUITEMINFO;
```

### MENUITEMINFO Mask Flags (fMask)

| Flag | Value | Meaning |
|------|-------|---------|
| `MIIM_STATE` | `0x00000001` | `fState` field is valid |
| `MIIM_ID` | `0x00000002` | `wID` field is valid |
| `MIIM_SUBMENU` | `0x00000004` | `hSubMenu` field is valid |
| `MIIM_TYPE` | `0x00000010` | `fType` and `dwTypeData` fields are valid |

### Menu Item Types (fType)

| Flag | Value | Meaning |
|------|-------|---------|
| `MFT_STRING` | `0x00000000` | Text string item |
| `MFT_SEPARATOR` | `0x00000800` | Separator line |
| `MFT_RADIOCHECK` | `0x00000200` | Radio-button style check |

### Menu Item States (fState)

| Flag | Value | Meaning |
|------|-------|---------|
| `MFS_ENABLED` | `0x00000000` | Item is enabled (default) |
| `MFS_CHECKED` | `0x00000008` | Item has a checkmark |
| `MFS_DISABLED` | `0x00000003` | Item is grayed out |

### CheckMenuItem Flags

| Flag | Value | Meaning |
|------|-------|---------|
| `MF_BYPOSITION` | `0x00000400` | `item` param is a zero-based position |
| `MF_BYCOMMAND` | `0x00000000` | `item` param is a command ID |
| `MF_CHECKED` | `0x00000008` | Set checkmark |
| `MF_UNCHECKED` | `0x00000000` | Remove checkmark |
| `MF_GRAYED` | `0x00000001` | Gray out the item |

### InsertMenu (SWELL_InsertMenu) — Simpler Menu API

```c
void SWELL_InsertMenu(HMENU menu, int pos, unsigned int flag, UINT_PTR idx, const char *str);
// Aliased as InsertMenu via SWELL defines
```

This is the simpler Win32 `InsertMenu` API. Key flags:

| Flag | Value | Meaning |
|------|-------|---------|
| `MF_STRING` | `0x0000` | Item is a text string |
| `MF_POPUP` | `0x0010` | Item is a submenu — `idx` is the HMENU cast to UINT_PTR |
| `MF_BYPOSITION` | `0x0400` | `pos` is a zero-based index |
| `MF_SEPARATOR` | `0x0800` | Separator item |

### CRITICAL: Adding Submenus on macOS/SWELL

**DO NOT use `InsertMenuItem` with `MIIM_SUBMENU` / `hSubMenu` for submenus on macOS.**
The menu item will appear but the submenu **will not expand**. This is a known SWELL behavior.

**USE `InsertMenu` with `MF_POPUP` instead** — this is what WDL's own `InsertSubMenu` helper does (`WDL/win32_helpers.h`):

```c
// C/C++ — WDL pattern
HMENU sub = CreatePopupMenu();
// ... populate sub with items ...
InsertMenu(parent, pos, MF_BYPOSITION | MF_POPUP | MF_STRING, (UINT_PTR)sub, "Submenu Label");
```

```zig
// Zig equivalent
var sub = Menu.createMenu();
// ... populate sub with items ...
swell.InsertMenu(parent, pos, MF_BYPOSITION | MF_POPUP | MF_STRING, @intFromPtr(sub.handle), label);
```

**`InsertMenuItem` is fine for regular items** (actions, separators) — only submenus require `InsertMenu` + `MF_POPUP`.

### InsertMenuItem Position Parameter

The third parameter `fByPosition`:
- `TRUE` (1) — `item` is a zero-based index position
- `FALSE` (0) — `item` is a command ID to insert before

Always use `TRUE` with position-based insertion (which is what all the examples use).

---

## 6. Extended State (Persistent Storage)

REAPER provides a simple key-value store for extensions:

```c
// Set a value (persist = true survives REAPER restart)
void SetExtState(const char* section, const char* key, const char* value, bool persist);

// Get a value (returns "" if not found)
const char* GetExtState(const char* section, const char* key);

// Check if a key exists
bool HasExtState(const char* section, const char* key);

// Delete a key
void DeleteExtState(const char* section, const char* key, bool persist);
```

Convention: use your extension name as the section (e.g., `"FeralFreq"`).

---

## 7. Common REAPER API Functions

### Project Management

```c
ReaProject* EnumProjects(int idx, char* projfnOutOptional, int projfnOutOptional_sz);
int IsProjectDirty(ReaProject* proj);
void Main_SaveProject(ReaProject* proj, bool forceSaveAs);
void SelectProjectInstance(ReaProject* proj);
double GetProjectLength(ReaProject* proj);
```

### Transport

```c
int GetPlayState();        // &1=playing, &2=paused, &4=recording
double GetCursorPosition(); // edit cursor position in seconds
double GetPlayPosition();   // play cursor position in seconds
void SetEditCurPos(double time, bool moveview, bool seekplay);
void CSurf_OnStop();
void CSurf_OnPlay();
void Main_OnCommand(int command, int flag);
```

### Tracks

```c
int CountTracks(ReaProject* proj);
MediaTrack* GetTrack(ReaProject* proj, int trackidx); // 0-based
int CountSelectedTracks(ReaProject* proj);
MediaTrack* GetSelectedTrack(ReaProject* proj, int seltrackidx);
void SetTrackSelected(MediaTrack* track, bool selected);
void SetOnlyTrackSelected(MediaTrack* track);
double GetMediaTrackInfo_Value(MediaTrack* track, const char* parmname);
bool SetMediaTrackInfo_Value(MediaTrack* track, const char* parmname, double newvalue);
int GetTrackDepth(MediaTrack* track); // folder depth
MediaTrack* GetParentTrack(MediaTrack* track);
```

### Media Items

```c
int CountMediaItems(ReaProject* proj);
MediaItem* GetMediaItem(ReaProject* proj, int itemidx);
int CountSelectedMediaItems(ReaProject* proj);
MediaItem* GetSelectedMediaItem(ReaProject* proj, int selidx);
void SetMediaItemSelected(MediaItem* item, bool selected);
double GetMediaItemInfo_Value(MediaItem* item, const char* parmname);
bool SetMediaItemInfo_Value(MediaItem* item, const char* parmname, double newvalue);
MediaTrack* GetMediaItem_Track(MediaItem* item);
MediaItem* SplitMediaItem(MediaItem* item, double position);
```

### Takes

```c
MediaItem_Take* GetActiveTake(MediaItem* item);
int CountTakes(MediaItem* item);
MediaItem_Take* GetTake(MediaItem* item, int takeidx);
void SetActiveTake(MediaItem_Take* take);
bool TakeIsMIDI(MediaItem_Take* take);
double GetMediaItemTakeInfo_Value(MediaItem_Take* take, const char* parmname);
bool SetMediaItemTakeInfo_Value(MediaItem_Take* take, const char* parmname, double newvalue);
```

### Envelopes

```c
int CountTrackEnvelopes(MediaTrack* track);
TrackEnvelope* GetTrackEnvelope(MediaTrack* track, int envidx);
TrackEnvelope* GetSelectedEnvelope(ReaProject* proj);
int CountEnvelopePoints(TrackEnvelope* envelope);
bool GetEnvelopePoint(TrackEnvelope* envelope, int ptidx, double* timeOut, double* valueOut, int* shapeOut, double* tensionOut, bool* selectedOut);
bool SetEnvelopePoint(TrackEnvelope* envelope, int ptidx, double* timeInOptional, double* valueInOptional, int* shapeInOptional, double* tensionInOptional, bool* selectedInOptional, bool* noSortInOptional);
bool InsertEnvelopePoint(TrackEnvelope* envelope, double time, double value, int shape, double tension, bool selected, bool* noSortInOptional);
bool DeleteEnvelopePointRange(TrackEnvelope* envelope, double time_start, double time_end);
int Envelope_Evaluate(TrackEnvelope* envelope, double time, double samplerate, int samplesRequested, double* valueOut, double* dVdSOut, double* ddVdSOut, double* dddVdSOut);
bool Envelope_SortPoints(TrackEnvelope* envelope);
```

### Undo

```c
void Undo_BeginBlock();
void Undo_EndBlock(const char* descchange, int extraflags);
// extraflags: -1 for auto-detect, or bitfield:
//   1=track config, 2=item/FX, 4=project, 8=freeze, 16=track env, 32=FX env
void Undo_OnStateChange(const char* descchange); // simple one-shot undo point
```

### UI

```c
void UpdateTimeline();    // redraw arrange view
void UpdateArrange();     // redraw arrange
void PreventUIRefresh(int prevent_count); // +1 to prevent, -1 to allow
void ShowConsoleMsg(const char* msg);
void TrackList_AdjustWindows(bool isMajor);
HWND GetMainHwnd();
```

### Grid/Snap

```c
double SnapToGrid(ReaProject* proj, double time_pos);
bool ApplyNudge(ReaProject* proj, int nudgeflag, int nudgewhat, int nudgeunits, double value, bool reverse, int copies);
```

---

## 8. Opaque Types

These are opaque pointer types — you never access their internals directly, only pass them to API functions:

| Type | What it represents |
|------|--------------------|
| `ReaProject` | A project tab |
| `MediaTrack` | A track |
| `MediaItem` | A media item on a track |
| `MediaItem_Take` | A take within a media item |
| `TrackEnvelope` | An envelope (volume, pan, FX param, etc.) |
| `PCM_source` | An audio source |
| `KbdSectionInfo` | A keyboard/action section |
| `HWND` | A window handle (SWELL on macOS) |
| `HMENU` | A menu handle (SWELL on macOS) |
| `HINSTANCE` | Plugin instance handle |

---

## 9. Key Struct Definitions

### custom_action_register_t

```c
typedef struct {
    int uniqueSectionId;  // 0=main, 32060=MIDI editor, 32063=media explorer
    const char* idStr;    // unique string ID, must start with "_", NULL for scripts
    const char* name;     // display name in action list, or path to script file
    void *extra;          // reserved, set to NULL
} custom_action_register_t;
```

### gaccel_register_t

```c
typedef struct {
    ACCEL accel;          // key binding (default, user can customize)
    const char *desc;     // description in action list
} gaccel_register_t;
```

### accelerator_register_t

```c
typedef struct accelerator_register_t {
    int (*translateAccel)(MSG *msg, accelerator_register_t *ctx);
    // Return: 0=not our window, 1=eat keystroke, -1=pass to REAPER,
    //         -10=raw (macOS), -20=pass to window (Windows)
    bool isLocal;  // must be TRUE
    void *user;    // user data
} accelerator_register_t;
```

### KbdSectionInfo

```c
typedef struct {
    int uniqueID;          // 0=main, negative for custom sections
    const char *name;      // section name
    KbdCmd *action_list;   // assignable actions
    int action_list_cnt;
    const KbdKeyBindingInfo *def_keys;
    int def_keys_cnt;
    bool (*onAction)(int cmd, int val, int valhw, int relmode, HWND hwnd);
    // ... internal fields
} KbdSectionInfo;
```

---

## 10. REAPER Info Value Strings

These strings are used with `GetMediaItemInfo_Value` / `SetMediaItemInfo_Value` and similar functions:

### MediaItem Info Values

| String | Type | Description |
|--------|------|-------------|
| `"D_POSITION"` | double | Item start position in seconds |
| `"D_LENGTH"` | double | Item length in seconds |
| `"D_SNAPOFFSET"` | double | Snap offset |
| `"D_FADEINLEN"` | double | Fade-in length |
| `"D_FADEOUTLEN"` | double | Fade-out length |
| `"D_VOL"` | double | Item volume (1.0 = 0dB) |
| `"B_MUTE"` | bool | Muted |
| `"B_LOOPSRC"` | bool | Loop source |
| `"I_GROUPID"` | int | Group ID |
| `"I_CURTAKE"` | int | Active take index |
| `"F_FREEMODE_Y"` | float | Free item positioning Y (0..1) |
| `"F_FREEMODE_H"` | float | Free item positioning height (0..1) |
| `"B_UISEL"` | bool | Selected in UI |

### MediaTrack Info Values

| String | Type | Description |
|--------|------|-------------|
| `"D_VOL"` | double | Track volume |
| `"D_PAN"` | double | Track pan (-1..1) |
| `"I_SOLO"` | int | Solo state |
| `"B_MUTE"` | bool | Muted |
| `"I_RECARM"` | int | Record armed |
| `"I_FOLDERDEPTH"` | int | Folder depth change |
| `"I_FOLDERCOMPACT"` | int | Compact state |
| `"I_SELECTED"` | int | Selected |
| `"I_TCPH"` | int | TCP height |
| `"I_WNDH"` | int | Current TCP window height |
| `"IP_TRACKNUMBER"` | int | 1-based track number (0=not found, -1=master) |
| `"GUID"` | GUID* | Track GUID |

### Take Info Values

| String | Type | Description |
|--------|------|-------------|
| `"D_STARTOFFS"` | double | Start offset in source |
| `"D_VOL"` | double | Take volume |
| `"D_PAN"` | double | Take pan |
| `"D_PLAYRATE"` | double | Playback rate |
| `"D_PITCH"` | double | Pitch adjustment (semitones) |
| `"I_CHANMODE"` | int | Channel mode |
| `"P_SOURCE"` | PCM_source* | Take source |

---

## 11. Patterns and Gotchas

### Menu Hook Critical Rules

1. **Only modify menus inside your hook callback** — never retain HMENU handles and modify them outside the callback.
2. **flag=0 is for structure only** — add items/submenus, but don't set checked/grayed state.
3. **flag=1 is for dynamic state** — set checkmarks, enable/disable items here.
4. **Don't retain submenu handles** — after inserting a submenu into a parent, REAPER owns it. Don't store the handle for later use.
5. **Call AddExtensionsMainMenu()** before your hook will be triggered for `"Main extensions"`.
6. **Use `InsertMenu` + `MF_POPUP` for submenus on macOS** — `InsertMenuItem` with `MIIM_SUBMENU`/`hSubMenu` will create an item that doesn't expand. See [SWELL section](#critical-adding-submenus-on-macosswell) for the correct pattern.

### Action ID Persistence

- String IDs (e.g., `"_FF_MYACTION"`) are persistent — they survive REAPER restarts.
- Integer command IDs are NOT persistent — they're assigned at runtime.
- Always use `NamedCommandLookup()` to convert string IDs to integer IDs.
- When registering a `custom_action`, the returned integer is the runtime command ID.

### Undo Best Practices

```c
Undo_BeginBlock();
PreventUIRefresh(1);    // prevent flicker
// ... do work ...
PreventUIRefresh(-1);
Undo_EndBlock("Description of change", -1);
UpdateTimeline();       // refresh display
```

### Thread Safety

- Most REAPER API functions must be called from the main thread.
- Timer callbacks run on the main thread — safe to call API from them.
- Audio hooks (`audio_hook`) run on the audio thread — very limited API access.

### Extension Loading Order

Extensions load in filesystem order. You cannot depend on another extension being loaded before yours. Use `GetFunc()` to check for optional dependencies (like SWS functions) and handle their absence gracefully.

### Common Mistake: plugin_register Pointer Casting

`plugin_register` takes `void*` as the second argument. You must cast your function pointers or struct pointers appropriately:

```c
// C - straightforward cast
plugin_register("hookcommand2", (void*)myHookFn);

// Zig - need @ptrCast and @constCast
_ = r.plugin_register("hookcustommenu", @ptrCast(@constCast(&menu.menuHook)));
```

### GetContextMenu

```c
HMENU GetContextMenu(int idx);
// idx: 0=track control panel, 1=media items, 2=ruler, 3=empty track area
```

Returns a handle to REAPER's built-in context menu. Use with caution.

---

## Version History

| REAPER Version | Notable SDK Changes |
|----------------|-------------------|
| v5.0+ | `hookcommand2` added |
| v6.0+ | `AddCustomizableMenu` added, customizable menu system |
| v7.0+ | Current stable, `GetContextMenu` expanded |

---

## Useful Resources

- **Official SDK**: https://www.reaper.fm/sdk/plugin/plugin.php
- **SDK GitHub**: https://github.com/justinfrankel/reaper-sdk
- **reaper_plugin.h**: https://www.reaper.fm/sdk/plugin/reaper_plugin.h
- **reaper_plugin_functions.h**: https://www.reaper.fm/sdk/plugin/reaper_plugin_functions.h
- **SWS Extension Source**: https://github.com/reaper-oss/sws (excellent reference for real-world extension code)
- **ReaScript API Docs (X-Raym)**: https://www.extremraym.com/cloud/reascript-doc/
- **Ultraschall API Docs**: https://mespotin.uber.space/Ultraschall/Reaper_Api_Documentation.html
- **CockosWiki**: https://wiki.cockos.com/wiki/index.php/Reaper_plugin_functions.h
- **REAPER Forum**: https://forum.cockos.com/
