# SuperDuper8 — Improvement Plan

> Working from the Super8 MIDI-controlled synchronized looper (Cockos, LGPL v2+).
> This document outlines two phases of improvement: standardizing MIDI input via a "common" channel routing mode, and a full UI overhaul using the Phil Ranger JSFX Audio Library.

---

## Phase 1: Standardize MIDI Inputs

### 1.1 Current State of MIDI Assignability

Super8 uses a right-click/drag system to assign MIDI notes, CCs, and PCs to controls. The current assignability status of every control is as follows.

**Per-lane — currently MIDI-assignable:**
- `rec` (`st_note1`) — start/set/stop recording
- `ply` / `stp` (`st_note2`) — toggle playback
- `sel` (`st_note3`) — toggle channel selected-for-monitoring
- `rev` (`st_note4`) — reverse loop

**Global — currently MIDI-assignable:**
- `kill` (`cfgp_reset`)
- `halve` (`cfgp_halve`)
- `double` (`cfgp_double`)
- `double no rep` (`cfgp_double_norep`)
- `x-fade shortened loop` (`cfgp_xfade`)
- `play all` (`cfgp_playall`)
- `rec/play selected` (`cfgp_playsel`)
- `add to project` (`cfgp_export`)

**Not currently MIDI-assignable (UI-only):**
- **Monitoring mode** (`st_monmode`, per-lane) — cycles off/auto/always-on via right-click on the speaker icon only
- `latch` toggle (`cfgp_latch`) — topbar button only
- `sync` mode (`cfgp_sync`) — topbar button only
- `gate` (`cfgp_gate`), `fade size` (`cfgp_fade`), `click count` (`cfgp_clickcnt`), `vclick offset` (`cfgp_vclick`), `offs` (`cfgp_offs`) — topbar drag only
- `link` (`cfgp_link`) — click on the link bracket between paired lanes only
- Per-lane `RDC`, `div`, `fade` sub-controls — UI drag only

**New per-lane actions to be added (do not exist in Super8):**
- **`clr`** — immediately clears (destroys) the audio content of a single lane, with no UI confirmation
- **`undo`** — restores the lane's audio buffer to the state it was in immediately before the most recent overdub; single-level per lane (see §1.4 Undo Buffer Management)

For Phase 1, the critical gaps are **monitoring mode**, **per-lane clear**, and **per-lane undo**: none of these can currently be triggered by MIDI. All three must be addressed.

---

### 1.2 The `common` Toggle

Add a new global boolean state variable `g_common` (default `1` = on). It should be persisted via `@serialize` alongside the existing `mem_gen_cfg` block (extend `mem_gen_sz` or use a separate config slot). A UI toggle button in the top bar (styled like the existing `latch` button) exposes it.

---

### 1.3 UI Changes

#### When `common` is ON

**Hide** per-lane instances of `rec`, `ply`, `sel`, and monitoring state controls from each loop lane row. Free up that real estate — it will be needed for the `channel` input (see below).

**Show globally** (in the top bar or a dedicated global-controls strip below the top bar) six MIDI-assignable input widgets:
- `rec` — same semantics as the per-lane `rec` note
- `clr` — new: immediately clears the target lane's audio buffer (no confirmation)
- `ply` — same semantics as per-lane `ply`
- `undo` — new: restores the target lane's audio buffer to its pre-overdub snapshot (single level)
- `sel` — same semantics as per-lane `sel`
- `mon` — new: cycles the target lane's monitoring mode (off → auto → always-on → off)

These six global inputs use the same `draw_value_tweaker` widget and right-click assignment mechanism as all existing controls. They read from six new global config pointers: `cfgp_common_rec`, `cfgp_common_clr`, `cfgp_common_ply`, `cfgp_common_undo`, `cfgp_common_sel`, `cfgp_common_mon`.

**Show per lane** a `channel` input widget in place of the hidden rec/ply/sel controls. This is an integer value in the range 1–16 representing the MIDI channel that lane listens on. It can be drawn with `draw_value_tweaker` using a new label (`"ch"`). Persist it as a new per-lane field; add `st_midichan` to the state record (increment `st_num` accordingly and update `@serialize`).

#### When `common` is OFF

Behavior is identical to the original Super8: per-lane `rec`, `ply`, `sel` tweakers are shown; the `channel` tweaker is hidden. The global monitoring-mode control (`mon`) also remains hidden in this mode; monitoring is still toggled only via right-click on the speaker icon.

#### Monitoring Mode — MIDI Activatability (both modes)

Regardless of the `common` toggle state, the monitoring mode needs a path to be triggered via MIDI. When a `mon` event fires (in common mode, routed to the correct lane; in any future per-lane mode), call the same `(rec[st_monmode]+1)%3` cycle logic that right-click already uses, then call `sliderchange(-1)` and force a redraw.

---

### 1.4 Logic Changes

#### MIDI Channel Extraction

Currently the `@sample` block compares `nextmsg_1 == 0x90` exactly. In REAPER JSFX, the status byte encodes the message type in the top nibble and the MIDI channel (0-based) in the bottom nibble — so `0x90` = note on channel 1, `0x91` = note on channel 2, etc. The current code therefore silently ignores all note-on messages arriving on channels 2–16.

Introduce two helper expressions:

```
msg_type = nextmsg_1 & 0xF0;   // 0x80, 0x90, 0xB0, 0xC0 ...
msg_chan = (nextmsg_1 & 0x0F) + 1;  // 1-based MIDI channel (1–16)
```

Replace the current hard-coded status byte checks with comparisons against `msg_type`. The existing CC→note and PC→note remapping logic can remain, but apply it after extracting the channel so `msg_chan` is preserved.

#### CC Value Gating

Before any CC→note remapping or common-mode dispatch, gate on the CC value:

```
nextmsg_3 >= 1 ? (
  // proceed with remapping and routing
);
```

This suppresses the value=0 release messages that button controllers (e.g. PaintAudio MIDI Captain NANO 4) send on button release, preventing spurious double-triggers. Value=0 messages are discarded silently.

#### Routing in `common` Mode

After the existing CC/PC-to-note remapping, add a `common` mode branch in `chan_onmsg`:

```
```
g_common ? (
  // Find the lane whose st_midichan matches msg_chan
  // then fire the global rec/clr/ply/undo/sel/mon assignment against that lane
  loop(g_nchan,
    mem_stlist[i*st_num + st_midichan] == msg_chan ? (
      nextmsg_2 == cfgp_common_rec[]  ? /* inject rec note for lane i */  :
      nextmsg_2 == cfgp_common_clr[]  ? /* clear audio buffer for lane i */ :
      nextmsg_2 == cfgp_common_ply[]  ? /* inject ply note for lane i */  :
      nextmsg_2 == cfgp_common_undo[] ? /* restore undo snapshot for lane i */ :
      nextmsg_2 == cfgp_common_sel[]  ? /* inject sel note for lane i */  :
      nextmsg_2 == cfgp_common_mon[]  ? /* cycle monmode for lane i */    ;
    );
    i += 1;
  );
) : (
  // existing per-lane dispatch unchanged
  ch1.onmsg(m1,m2,m3);
  ch2.onmsg(m1,m2,m3);
  ...
);
```

The "inject" approach for `rec`/`ply`/`sel` can reuse the existing `g_inject_midinote` mechanism: set `g_inject_midinote = 1024 + mem_stlist + lane*st_num + st_note1` (or `st_note2`, `st_note3`) and let the next sample block dispatch it naturally. The `mon` action can call the cycle logic inline. `clr` and `undo` are handled inline (see Undo Buffer Management below).

#### Undo Buffer Management

Per-lane undo stores a single pre-overdub snapshot of each lane's audio buffer. The approach is:

**State fields (new per-lane):**
- `st_undo_buf` — memory address of the start of this lane's undo snapshot (0 = no snapshot available)
- `st_undo_len` — number of samples stored in the snapshot

**On overdub start** (when a `rec` trigger fires on a lane that already has content): before recording begins, copy the lane's current audio buffer to its undo region and record its length in `st_undo_len`. This establishes the single undo point. Any subsequent overdub on the same lane overwrites the previous snapshot.

**On `undo` trigger** for lane i: if `st_undo_len > 0`, copy the undo snapshot back over the active audio buffer and force a redraw. If `st_undo_len == 0` (lane has never been overdubbed, or was cleared), the message is silently ignored.

**On `clr` trigger** for lane i: clear the active audio buffer and reset `st_undo_len = 0` (the cleared state is not itself undoable).

**Memory allocation:** Undo buffers must be pre-allocated in `@init` alongside the primary audio buffers. Each lane requires 2× its maximum audio buffer capacity (one active, one snapshot). Use `options:maxmem` to raise JSFX memory ceiling accordingly — document the required value in the plugin header comment once the per-lane buffer sizes are known. If available memory is insufficient for a given lane's loop length at snapshot time, skip the snapshot silently (undo will remain unavailable for that lane until the loop is shorter).

#### Latch Compatibility

The latch queue (`latchq_add` / `latchq_process`) must work with routed messages. When `g_latchmode` is on and a common-mode message arrives that would be queued, store the *resolved* per-lane note (post-routing) in the latch queue rather than the raw global assignment, so that the latch fires correctly at loop boundary.

#### Serialize / Backward Compatibility

- Extend `mem_gen_sz` to accommodate `cfgp_common_rec`, `cfgp_common_clr`, `cfgp_common_ply`, `cfgp_common_undo`, `cfgp_common_sel`, `cfgp_common_mon`, and `g_common`. Default all to `128` (OFF) except `g_common` which defaults to `1`.
- Add `st_midichan` to the per-lane serialize loop (after the existing `st_note4`, `st_rdc`, `st_div`, `st_fade_size` blocks) wrapped in `file_avail(0) != 0 ?` guards for backward compatibility.
- Default `st_midichan` per lane to `lane_index + 1` (lane 0 → channel 1, lane 1 → channel 2, etc.) so that fresh instances "just work" with the most natural 1:1 mapping.
- `st_undo_buf` and `st_undo_len` are runtime-only (not serialized); undo snapshots do not persist across plugin reload. Both initialize to `0` in `@init`.

---

## Phase 2: Improve UI Appearance

### 2.1 The Phil Ranger JSFX Audio Library

The library is `PhilRangerAudioLibraryAlpha5.jsfx-inc` from [PhilRangerQuebec/Phil-Ranger-JSFX-Audio-Library](https://github.com/PhilRangerQuebec/Phil-Ranger-JSFX-Audio-Library). It is included in a plugin via:

```
import PhilRangerAudioLibraryAlpha5.jsfx-inc
```

Files are distributed under GPL-3.0 and installed into a `philranger_GUI and Audio Library/` subfolder alongside the plugin.

#### Key API Surface (Alpha5)

All layout dimensions are expressed as **percentages of the plugin window**, making the interface automatically resizable.

| Function | Description |
|---|---|
| `PR.SetShape(cx%, cy%, w%[, h%])` | Set position and size for the next widget call. If `h%` is omitted, the widget is square. |
| `PR.DisplayPanel(x%, y%, w%, h%)` | Draw a background panel. Color controlled by `PR.Color.Panel` (8-char hex RRGGBBAA). |
| `PR.DisplayText(x%, y%, size%, text[, 'b'])` | Draw centered text. `'b'` = bold. |
| `PR.DisplaySlider(slider[, pic, x, y, w, h])` | Vertical or horizontal slider (auto-detected from shape aspect ratio). Can use a custom sprite image. |
| `PR.DisplayKnob(slider[, pic, x, y, size])` | Rotary knob, default drawn or custom sprite. |
| `PR.DisplaySwitch(slider[, pic, x, y, w, h])` | Toggle switch (2-state), default drawn or custom sprite. |
| `PR.DisplayRotarySwitch(slider, steps[, pic, x, y, size])` | Multi-position rotary selector. |
| `PR.DisplayMeter(value, min, max[, x, y, size])` | VU-style meter bar. |
| `PR.DisplayPicture(x%, y%, w%, pic_num)` | Blit a loaded image. |
| `PR.NewSlider(num, default, min, max[, linearity])` | Register a slider with optional log/exp mapping. Returns a handle. |
| `PR.NewList("a","b","c",...)` | Build a string list for rotary switch labels. |
| `PR.NumbertoText(value)` | Format a numeric value as a display string. |
| `PR.Color.Panel`, `PR.Color.Text`, `PR.Color.Main`, `PR.Color.Contour`, `PR.Color.Shadow` | Global color theme variables (hex strings). |

Right-click on any PR widget resets it to default; Ctrl-drag gives finer adjustment. The library handles all mouse interaction internally.

#### Important Usage Note

The library must be initialized in `@init` by calling `PR.NewSlider(...)` for every slider that will be displayed. In `@gfx`, call `PR.SetShape(...)` immediately before any display call that doesn't pass explicit coordinates, or pass `x, y, size` inline in the display call itself. Percentage coordinates refer to the plugin window's current pixel dimensions, so the layout adapts automatically when the window is resized — no manual `gfx_w`/`gfx_h` arithmetic is needed in layout code.

---

### 2.2 Layout Concept

The current Super8 window is `420×480` px with a square-grid layout (`rowsize` driven by aspect ratio). SuperDuper8 will replace this with a **horizontal-row-per-lane** layout at a much larger default size, e.g. `900×600` px (or `900×720` for 8 lanes with comfortable row height).

#### Global Structure (top → bottom)

```
┌─────────────────────────────────────────────────────────┐
│  Top Bar: animated name │ sync │ offs/length │ click cnt │
│           vclick │ gate │ latch │ [common toggle]        │
├─────────────────────────────────────────────────────────┤
│  Position / BPM bar                                     │
├─────────────────────────────────────────────────────────┤
│  [common mode global strip: rec / clr / ply / undo / sel / mon]  │  ← visible only when common=on
├─────────────────────────────────────────────────────────┤
│  Lane 1 row                                             │
│  Lane 2 row                                             │
│  ...                                                    │
│  Lane N row                                             │
├─────────────────────────────────────────────────────────┤
│  Global action buttons: halve │ double │ double-norep   │
│  x-fade │ play all │ rec/play sel │ add to project │ kill│
├─────────────────────────────────────────────────────────┤
│  Peak meter / last MIDI message                         │
└─────────────────────────────────────────────────────────┘
```

#### Per-Lane Row Layout

Each lane occupies a full-width horizontal strip (~70–80 px tall for an 8-channel config at 720 px height). Within each row, from left to right:

```
 [#] [State] [Peak strip] [────── Waveform ──────] [RDC][div][fad] [mon] [MIDI inputs] [link bracket]
```

More specifically:

- **Lane number & state color** — left ~6% of width. A `PR.DisplayPanel` colored red (recording), green (playing), or dark grey (stopped/empty). Lane number via `PR.DisplayText`.
- **Input peak meter** — a thin vertical `PR.DisplayMeter` strip (~1% wide) immediately to the right of the state block, running the full row height.
- **Waveform** — center region (~45% of width). Retains the existing `draw_waveform` logic; no change to the waveform rendering itself. Surrounded by a `PR.DisplayPanel` for visual framing.
- **Sub-controls** (RDC, div, fad) — right of waveform, ~12% of width. Three stacked `draw_value_tweaker` widgets or their PR equivalents. These are compact but benefit from increased row height giving more vertical room.
- **Monitor toggle** — a `PR.DisplaySwitch` or custom drawn speaker icon (~5% of width). Right-click cycles the three modes as before; the switch can visually reflect the 3-state value (off/auto/on) using a rotary-switch-style approach with `PR.DisplayRotarySwitch(lane_monmode_slider, 3)`.
- **MIDI inputs** — when `common=off`: three or four `draw_value_tweaker` boxes for rec, ply, sel, rev (~18% of width); when `common=on`: one `draw_value_tweaker` for `ch` (~6%). These can be made appreciably taller (matching row height minus padding) and wider than the current 24×24 px.
- **Link bracket** — retained between odd/even adjacent lane pairs, now oriented vertically on the far right edge.

#### Common-Mode Global Strip

When `common` is on, insert a dedicated horizontal strip between the position bar and the first lane row:

```
 [rec]  [clr]  [ply]  [undo]  [sel]  [mon]
```

These six widgets are the same `draw_value_tweaker` style, but larger (e.g. 40 px wide, 32 px tall) with clearer labels and a visually distinct background panel (`PR.DisplayPanel` with a contrasting color) to signal their global scope. `clr` and `undo` should be visually distinguished from `rec`/`ply` — consider a slightly warmer panel color for the destructive/restorative pair.

#### Global Action Button Strip

The existing `mem_gen_order` buttons (halve, double, x-fade, play all, etc.) move to a dedicated strip at the bottom. Render them as `PR.DisplayPanel` + `PR.DisplayText` blocks, styled with the same color coding as today but with consistent padding. The per-button MIDI tweaker sits in the bottom-right corner of each button block.

---

### 2.3 Using the PR Library for Lane Rows

The existing `draw()` function will be substantially rewritten. The PR library's percentage-based coordinate system maps cleanly to rows: compute each row's top-Y percentage from `lane_index / g_nchan * available_height_pct` and pass it inline to display calls.

Example sketch for a single lane row:

```eel2
// Compute lane row bounds (percentages)
row_top_pct  = topbar_pct + pos_bar_pct + common_strip_pct + i * lane_h_pct;
row_cy_pct   = row_top_pct + lane_h_pct * 0.5;

// State background panel
mode==2 ? PR.Color.Panel = "AA2020FF" :  // recording — red
mode==1 ? PR.Color.Panel = "208820FF" :  // playing — green
          PR.Color.Panel = "303040FF";   // stopped — dark blue-grey
PR.DisplayPanel(3, row_cy_pct, 5, lane_h_pct * 0.9);
PR.DisplayText(3, row_cy_pct, 3.5, sprintf(#,"%d", i+1));

// Waveform area panel
PR.Color.Panel = "1A1A2AFF";
PR.DisplayPanel(32, row_cy_pct, 42, lane_h_pct * 0.88);
// ... call existing draw_waveform() with pixel coords derived from gfx_w/gfx_h

// Monitor rotary switch (3 positions)
PR.SetShape(82, row_cy_pct, lane_h_pct * 0.55);
PR.DisplayRotarySwitch(lane_monmode_slider[i], 3);

// MIDI input tweakers (common=off)
g_common == 0 ? (
  draw_value_tweaker(...); // rec
  draw_value_tweaker(...); // ply
  draw_value_tweaker(...); // sel
  draw_value_tweaker(...); // rev
) : (
  draw_value_tweaker(...); // ch
);
```

Note: `draw_value_tweaker` is not part of the PR library — it's Super8's own widget. It should be retained and can be called alongside PR library calls freely, since the PR library only manages its own internal state.

---

### 2.4 Color Theme

Propose a default theme that differentiates zones clearly:

| Zone | Color (RRGGBBAA) |
|---|---|
| Top bar | `00004FFF` (deep navy) |
| Lane row (stopped, empty) | `252530FF` |
| Lane row (stopped, has content) | `2A2A40FF` |
| Lane row (playing) | `1A3A1AFF` (dark green) |
| Lane row (recording) | `3A1A1AFF` (dark red) |
| Lane row (selected) | adds bright border, 2px |
| Waveform panel | `141420FF` |
| Global common strip | `201A40FF` (purple-tinted) |
| Action button strip | `101018FF` |
| Panel contour | `5060A0FF` |

These can all be set via `PR.Color.*` globals at the start of `@gfx`, overriding the library defaults.

---

### 2.5 Recommended Default Window Size

Set `@gfx 900 720` as the new default. This gives ~80 px per lane row for 8 channels, leaving room for the topbar (~30 px), position bar (~20 px), optional common strip (~36 px), and action button strip (~60 px). The PR library's percentage layout ensures the plugin remains usable at any size the user drags it to.

---

## Implementation Order Summary

1. **Phase 1a** — Add `st_midichan`, `st_undo_buf`, `st_undo_len` per-lane fields; add all six `cfgp_common_*` global fields; extend `@serialize` and pre-allocate undo buffers in `@init`; set `options:maxmem` appropriately.
2. **Phase 1b** — Add `g_common` toggle button to topbar; implement UI show/hide logic for per-lane vs. global tweakers and per-lane `channel` tweaker; add all six tweakers to the common-mode global strip.
3. **Phase 1c** — Fix MIDI channel extraction (`msg_type` / `msg_chan`); add CC value gate (`>= 1`); implement common-mode routing in `chan_onmsg` including `clr` and `undo` inline dispatch; ensure latch compatibility.
4. **Phase 1d** — Add monitoring mode MIDI activation path (`cfgp_common_mon` in common mode); verify right-click per-lane cycling still works; verify `clr` immediately silences and wipes the lane; verify `undo` correctly restores the pre-overdub snapshot and is a no-op when no snapshot exists.
5. **Phase 2a** — Install PR library (`import` directive, `filename:` entries, `PR.NewSlider` init calls). Validate it renders correctly in a stripped-down test branch.
6. **Phase 2b** — Rewrite `draw()` to use horizontal row layout; integrate PR library panels, text, and monitor switch.
7. **Phase 2c** — Style global action button strip and common-mode strip using PR library panels.
8. **Phase 2d** — Tune color theme, resize default window, final visual polish.
