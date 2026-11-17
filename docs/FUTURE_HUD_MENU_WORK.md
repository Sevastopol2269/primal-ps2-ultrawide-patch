# Primal (SCUS-97142 / FCD89DC3) — open work: gameplay HUD and full pause-menu fix

Written 2026-11-17 so this can be picked up later. Everything here was found live over PINE and the PCSX2
debugger on the test copy. All test edits listed below were
reverted unless the section says "kept".

## Shipped now (kept)

- Camera Hor+/Vert+ for all ratios.
- **Pause menu, simple fix:** side Flash layer only. One word at `0x00117E94` (`lui at,X`, stock `3C013F40` = 0.75),
  X = 0.75 × (16/9)/aspect. 21:9 `3C013F10`, 32:9 `3C013EC0`, 48:9 `3C013E80`. Only runs when the in-game
  widescreen option is on (branch condition `0x0010F390` returns the widescreen flag `0x004883FC`), so 4:3 is stock.
  Result: background, orbs, Jen/Scree portrait, logo and icons correctly proportioned; main panels, text, button
  prompts and map stay stretched (stock behaviour).

## Useful addresses

| What | Address | Notes |
|---|---|---|
| In-game widescreen flag | `0x004883FC` | 1 = on |
| Pause menu open flag | `0x004885D8` | 1 = menu open, 0 = gameplay |
| Screen size ints | `0x004883AC` / `0x004883B0` | 640 / 448 |
| GS x / y offset ints | `0x004883B8` / `0x004883B4` | 27648 / 29184 (×16 units) |
| Global SCISSOR (64-bit) | `0x00488648` | copied into frame packets at `0x00298968` (ctx1) and `0x00298998` (ctx2) |
| Flash player instances | `0x0052A4D4`, `0x0057C804` | stage width/height floats 640.0 / 480.0, then frame time 1/30 |
| Main Flash layer matrix (stored) | `0x0052A540` | sx, sy, tx, ty (0x0052A4D0 + 112) |
| Free code-cave area | `0x00095564`–`0x000FFFFF` | zero in every dump; `0x000FFF00`–`0x000FFF1C` used by Vert+ |

## Flash UI (menus)

- Menus are SWF v4 movies (`FSCommand`, `DisplayFrame`, `iFocus`). Loaded with menu open: `0x00B6FF40`, `0x0165A7C0`,
  `0x01781B80` (640×480 stage), `0x00BF8A80` (250×480 stage = side layer).
- **Main layer builder `0x00117D44`** (runs every frame): x scale = screenW / stageW, x offset from `0x004883B8`.
  Squeeze patch (works, NOT kept):
  `00117D44 3C01xxxx` (lui W) · `00117D48 44810000` · `00117D4C 00000000` ·
  `00117D68 3C01yyyy` (lui X) · `00117D6C 44812800` · `00117D70 00000000`;
  W = 640·s, X = 27648 + 320·(1−s)·16, s = (16/9)/aspect. 32:9: W `43A0`, X `46EC`; 21:9: W `43F0`, X `46E2`.
- **Side layer builder `0x00117E6C`:** WS squeeze 0.75 at `0x00117E94`; right-edge anchor screenW × 0.3953125 at
  `0x00117EB4`/`0x00117EB8` (`3C013ECA`/`34216667`). With the main squeeze, anchor 0.5 + (0.3953125 − 0.5)·s
  (32:9: `3C013EE5` / `34213333`) lines the column up with the squeezed panel.

### Why the full menu squeeze was not kept
1. **Text:** glyphs keep their unsqueezed pixel width while text boxes shrink, so lines wrap and overlap
   (Audio: "combat music / volume" over the slider; footer wraps). The glyph width does not come from builder 1,
   builder 3 (`0x001154F0`, tested, no effect), or the global screen width. The stored main matrix is read only by the
   scene-graph multiply `0x0037BC20` (caller `0x002EBB7C`, world = local × parent), so glyph sizing happens later —
   likely a glyph/quad builder or VU1. **Next step:** read breakpoint on a text object's world matrix (child of
   the root at `0x0052A4D0`), or find the font quad emitter and make glyph width use sx instead of sy/1.0.
2. **Map tab overflow:** content beyond the 640 stage is visible once squeezed. Narrowing SCISSOR while the menu flag
   is set (cave on `0x00298968`/`0x00298998`, x 160..479) changed the frame packets but did not clip the map, so the
   Flash draws set their own scissor (probably in VU1 packets). **Next step:** find that scissor write.

## Gameplay HUD (not fixed)

- Stretched 2× at 32:9 (compass/orb gauge top-left, gem counters, bottom health bar).
- **Not** affected by: main Flash builder; side Flash builder (`0x00117E94` to 0.375 had no effect in gameplay);
  sprite routine `0x003C3220` (WS squeeze 0.7512 at `0x003C3228`; cave with width ×0.5 and +160 offset, no
  change); no HUD camera with aspect 4/3 exists (the 1.333 values are animation data).
- **Leads:** HUD code passes normalized positions with WS tweaks, e.g. `0x0012853C`–`0x0012858C` (x 0.15 → 0.1126
  in WS) into `0x0010B628`, which draws via a vtable call at `0x0010B71C`. Helpers `0x00118BE0` (x_norm × screenW)
  and `0x00118BF8` (y_norm × screenH). Many WS-keyed constants in `0x00124xxx`–`0x00128xxx`.
  **Next step:** execute breakpoint at `0x0010B628` / `0x0010B71C` in gameplay, step into the draw, find where the
  HUD x becomes a GS coordinate, then apply x' = 320 + (x − 320)·s and width × s.

## Other open items

- **Loading screen** (user, 2026-11-17): to be fixed later (stretched 2D at wide ratios, like the menus; not investigated yet).
- **60 fps** is solved in release v2 (video-aware vsync hook) — keep it in mind if a HUD/menu fix touches frame pacing.

- 48:9: sky/fog/backdrop does not reach the screen edges (blue void). 32:9 is clean.
- Main menu (title) uses FMV backgrounds → handled by PCSX2 FMV Aspect Ratio = 16:9.

## Method notes

- Live tests: write values over PCSX2's PINE interface and keep a log of the original values so every edit can be reverted.
- Debugger breakpoints: Breakpoints tab → right-click → New → Memory, Read or Write, size 4. Delete the breakpoint and
  press Run afterwards. Keep the debugger window on screen before clicking (it moves).
- Loading a save state wipes live edits unless the state was saved with them active.
