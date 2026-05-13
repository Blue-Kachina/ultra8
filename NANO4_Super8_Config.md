# PaintAudio MIDI Captain NANO 4 — Super 8 Looper config draft

Generated for Matt's Reaper Super 8 Looper plan.

## Important caveats

This is based on PaintAudio Super Mode syntax as documented publicly:

- `keytimes = [x]`
- `ledmode = [select|normal|tap]`
- `ledcolorX = [0xRRGGBB][0xRRGGBB][0xRRGGBB]`
- `short_dwX`, `short_upX`, `longX`, `long_upX`
- MIDI commands use `[channel][CC|PC|NT][number][value]`

The NANO 4 Super Mode page files are usually placed in the `supersetup` directory as `page0`, `page1`, etc. Back up your factory files first.

## Page/channel map

- page0 = MIDI channel 1
- page1 = MIDI channel 2
- page2 = MIDI channel 3
- page3 = MIDI channel 4
- page4 = MIDI channel 5
- page5 = MIDI channel 6
- page6 = MIDI channel 7
- page7 = MIDI channel 8

Copy the same `supersetup` folder to each NANO 4. Then use page 1 for Unit A, page 2 for Unit B, page 3 for Unit C, etc.

## Physical switch assumptions

Based on the public NANO 4 Super Mode guide:

- key0 = Button 1
- key1 = Button 2
- key2 = Button A
- key3 = Button B

Verify this on the hardware before relying on it live.

## LED approach in this draft

Because the NANO 4 is not receiving Reaper/Super 8 state feedback here, LEDs are local hints only. They show the most recently pressed action color, not the actual lane state.

- Button 1: red = record/overdub action
- Button 2: green = play/stop action
- Button A: blue = alt/undo action
- Button B: white = sync/global action

Long-press LED colors are not independently documented as a separate visual state in the sources I found, so the long-press actions send MIDI but leave LED behavior simple.

## Page changing warning

Public notes indicate NANO 4 Super Mode may reserve long-press behavior on some switches for page up/down. Your requested long actions on Button 2 and Button B may conflict with built-in page navigation depending on firmware. Test this first.

If long-press conflicts, the safer plan is:

- Keep Button 2 long/B long unassigned for page navigation, or
- Put clear/stop-all on short press of a dedicated utility page, or
- Use a different firmware/editor that exposes page navigation more explicitly.
