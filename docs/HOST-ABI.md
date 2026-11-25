# The host ABI (`fm1core`, ABI 6)

An `.rfm1` wraps a firmware's **host build**: the firmware compiled against a small C interface, `fm1core`, instead of
the FM-1's hardware. This page summarises that interface. The header itself is VST `core-api/fm1core.h`
(GPL-3.0-only); read it for the exact types. Every fact below is from that header unless another source is named.

## The idea

A host build exports one function:

```c
const fm1core_t *fm1core_get(uint32_t abi_version);   /* NULL if built for another ABI */
```

It returns the **descriptor** (`fm1core_t`): strings that describe the firmware, a pointer to its **panel**
(`fm1panel_t`: the keys, buttons, knobs, LEDs and screen as plain memory), and the entry points the host calls.
Nothing else in the host links a firmware symbol. In an `.rfm1`, the `rfm1_*` exports call these entry points for the
host ([RFM1-FORMAT.md](RFM1-FORMAT.md#4-exports)).

Each firmware gets there through **glue**: C files that include the firmware's own sources and translate between its
hardware layer and `fm1panel_t`. Firmwares built on fm1-base have a ready-made host layer in fm1-base itself
([BUILDING.md](BUILDING.md)).

- **ABI 6** = ABI 5 plus `credits` at the descriptor's end. Hosts still load ABI 5 (`FM1CORE_ABI_MIN`), without
  credits. A change to either struct's layout bumps the ABI.

## The panel (`fm1panel_t`)

| Constant | Value |
| --- | --- |
| Screen | 240 × 240, RGB565 big-endian (`lcd[240*240]`) |
| Sample rate / block | 44100 Hz, 128 frames per `audio()` block, stereo int16 interleaved |
| Note keys | 27: key 0 = F3 (MIDI 53) … key 26 = G5 |
| Buttons | 14, label order `FM1P_B_*`: FX SEL ENV LFO EDIT GLO HOME SAVE ARP SEQ PLAY REC OCT− OCT+ |
| Knobs | 7 encoders, roles `FM1P_E_*`: SELECT ALGO PRESETS K1 K2 K3 K4, plus the MASTER pot (0..1023) |
| LEDs | 11 columns × 4 lines, brightness 0..255 (OFF 0, FAINT 12, DIM 40, MID 96, LIT 255; "lit" from 176), plus PLAY's green |

Host → firmware (written between calls, read at the next `tick` / `frame` / `idle`): `keys`, `buttons` (bit masks),
`keys_tap`, `buttons_tap` (presses shorter than 1 ms; the firmware clears them), `enc[7]` (detents not yet taken; the
firmware zeroes them), `master`, `key_vel[27]` (1..127, 0 = default), `key_lean[27]` (aftertouch 0..100 %).

Firmware → host (valid after `init`, `frame` and `idle` return): `led[11][4]`, `led_play_green`, `lcd[]`,
`lcd_writes`. Filled by `init`: `btn_id[14]`, `enc_id[7]`, `enc_dir[7]` (which matrix id is which labelled control).
`ready`: the host sets 2 (simulated time) before `init`.

## The descriptor (`fm1core_t`)

In order: `abi_version`, `id` (stable, `[a-z0-9_]`), `name`, `version`, `source_url` (repo and commit), `family` (the
glue family), `license` (SPDX), `gated` (NULL, or why the core stays out of public builds), `flash_size` (1 MiB),
`button_names[14]`, `params` / `nparams` (the optional parameter map), `panel`, `keymap`; then the functions
`options`, `boot_options`, `init`, `tick`, `frame`, `idle`, `audio`, `midi_in`, `midi_out_take`, `dump`, `stats`,
`ui_info`, `shutdown`, `halted`, `flash_stage`, `flash`, `flash_dirty`, `flash_sync`, `param_epoch`, `knob_target`;
then `credits` (ABI 6).

The parameter map (`fm1param_t`: name, range, default, value names, unit, `get`, `set`, `text`, `target`, flags) is
optional. In an `.rfm1` it is not reachable by the host directly (RFM1-FORMAT §4), but the readout bridge uses it to
make knob readouts with no extra code (VST `docs/RFM1.md` "Readouts").

## Timing

- **One thread.** Every entry point of one core, and every panel read or write, comes from one thread at a time, in
  order. The firmware never starts a thread or a timer.
- **The clock is the host's:** `tick(ms)` once per device millisecond counting up from 0; then `frame()` at ms 0 and
  every 15 ms, or `idle()` otherwise; then `audio()` for each 128-frame block that has fallen due at 44.1 kHz. The Move
  calls `frame()` every 33 ms for the firmware on screen and every 200 ms for the others, while `tick()` still runs
  every ms (MOVE `README.md` "Frames"): a firmware that reads its panel only in `frame()` sees input later there.
- **Lifetime:** `options` → `boot_options` (optional) → `ready = 2` → `init` → the clock → `shutdown`.
- **Exit and reboot:** a firmware that calls `exit()` or reboots does not end the host: `halted()` turns non-zero
  (`0x100 | code`), the clock entry points do nothing (silence), and the host powers it on again.
- **Flash** is a RAM image of `flash_size` bytes: staged before `init` (the path `"fm1core:staged"`), read through
  `flash()`, with `flash_dirty()` a counter that moves on every erase or write so the host knows when to save.

## Optional symbols

A host looks these up by name when present: `fm1core_latency_frames`, `fm1core_boot_option`,
`fm1core_labels_epoch` / `fm1core_label` (live labels), `fm1core_audio_out_info` / `fm1core_audio_out` (more output
channels). Their `.rfm1` exports are in [CAPABILITIES.md](CAPABILITIES.md).

## Inside an `.rfm1`

- `FM1CORE_RFM1` is defined to 1 in every `.rfm1` build, and `__EMSCRIPTEN__` as always with emcc (VST `docs/RFM1.md`).
- A firmware main loop that never returns runs on a coroutine; in an `.rfm1` that coroutine is built on Binaryen's
  Asyncify, with nothing for the author to add (VST `docs/RFM1.md` "What does not work in Wasm").
- Native-only headers can get stand-ins in VST `core-glue/common/wasm_compat/`.
- No file system, no threads, no SIMD ([RFM1-FORMAT.md](RFM1-FORMAT.md#7-the-webassembly-a-module-may-use)).
