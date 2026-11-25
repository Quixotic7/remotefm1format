# Optional capabilities

Every capability here is optional. A plain `.rfm1` with none of them runs on every host. Each one is added at build
time; none adds an import (except the Fantasy CFW's file API), so a host that does not know a capability simply
ignores its exports and the firmware still runs (VST `docs/RFM1.md`).

**How a firmware opts in.** With FM1VST, one word in its build line: `fm1_add_rfm1(<id> … WANTS rd1 tk1 …)`. That word
(1) adds a define to every compile unit, so the firmware's own calls switch on, (2) adds the matching exports from
the shim (VST `core-glue/common/rfm1_shim.c`), and (3) writes the name into the manifest's `wants`. Without the
define, the firmware-side macros compile to nothing, so **the FM-1 device image stays byte for byte the same** (VST
`docs/RFM1.md`; `cmake/FM1Rfm1.cmake`).

| `WANTS` | Define in every unit | Firmware-side header (vendor it; MPL-2.0) |
| --- | --- | --- |
| `rd1` | `FM1_HOST_EXT=1` | `core-api/rfm1_ext.h` |
| `tk1` | `FM1_HOST_TK=1` | `core-api/rfm1_move.h` |
| `ep1` | `FM1_HOST_EP1=1` | `core-api/rfm1_ep.h` |
| `cpu2` | `FM1CORE_RFM1_CPU2=1` | `core-glue/common/rfm1_cpu2.h` |
| `ain1` | — | the core's own audio-in hook (VST `docs/VIRTUAL-CARTS.md` §8) |

`rfm1_check.py` enforces the pairing: a capability's exports must be all there or none, and its name must be in
`wants` exactly when they are (VST `tools/rfm1_check.py`).

## Which host honours what

From the hosts' code at the commits in [SOURCES.md](SOURCES.md) (a search for each capability's export or section
name in the host's source; the Move from its README and code). "—" means no support was found.

| Capability | remotefm1.com | iOS / AUv3 | Android | Ableton Move (0.4.0) |
| --- | --- | --- | --- | --- |
| Skin (`rfm1.skin`) | yes | yes | yes | on the Move's web page only, not the Move's own screen |
| Fixed labels (from `RemoteFM1Metadata.json`) | yes | — (iOS's label rows are pending: VST `docs/DYNAMIC-LABELS.md`) | yes | yes |
| Live labels | yes | — | yes | yes |
| Readouts `rd1` | — | — | — | yes (since 0.1.0) |
| Takeover `tk1` | — | — | — | yes (since 0.2.0) |
| Engine parameters `ep1` | — | — | — | yes, the SET operation (since 0.4.0) |
| Second core `cpu2` | runs inline | runs inline | runs inline | on its own thread (since 0.1.0) |
| Audio input `ain1` | yes | yes | yes | yes |
| More outputs `aout1` | — | yes (the AUv3's extra buses) | — | — |
| Fantasy CFW | yes | yes | yes | yes |
| Screen reader protocol | — | — | — | yes (since 0.1.2) |
| Working memory (states) | — | — | — | yes (since 0.4.0), host-side |

"Runs inline": the module works as on a one-core host, as built.

---

## Skin (`rfm1.skin`)

**What the user gets:** the firmware's own look on the on-screen FM-1: a background image behind the faceplate and its
colours. Users can turn firmware skins off.

**Opt in:** a `skin` object in `RemoteFM1Metadata.json` at the repository root ([METADATA.md](METADATA.md)); FM1VST
packs it at build time, or add it to a built file with `python3 tools/rfm1_skin.py add in.rfm1 -o out.rfm1
--metadata RemoteFM1Metadata.json` (VST `tools/rfm1_skin.py`). Adding a skin changes the file's SHA-256.

**Layout** (IOS `docs/RFM1-FORMAT.md` §8; VST `docs/RFM1-METADATA.md`):

- `rfm1.skin`: UTF-8 JSON `{"skin": 1, "dim": 0.2, "theme": {…}}`. `dim` 0..0.8 (default 0): how much of the `body`
  colour lies over the image. `theme`: either the five roles `body key knob led bezel`, or all fourteen
  `body label bed key hi lo keyText knob mark bezel lcd ink led ledOff`, each `"#RRGGBB"`.
- `rfm1.skin.background`: PNG or JPEG, at most 4 MiB (the metadata route also limits the longer side to 4096 px).
  Cropped to fill. Hosts scale it to at most 2048 px.
- `rfm1.skin.background.landscape` (optional): the image for wide views.
- A skin needs `rfm1.skin` and at least an image or a theme.

**The Move** copies the three sections as `.rfm1skin`, `.rfm1skin.bg`, `.rfm1skin.bgl` and serves them to its web
page (MOVE `tools/rfm1info.c`, `src/web.c`).

## Labels (fixed and live)

**What the user gets:** the panel's buttons and knobs named for what they do in this firmware ("CUTOFF" instead of
"K1"). An empty label `""` means the control has no job: drawn blank or dimmed (on the Move, a dark, dead knob).

**Fixed labels:** a `labels` object in `RemoteFM1Metadata.json` ([METADATA.md](METADATA.md)). FM1VST compiles them
into the shim as a static table (VST `tools/rfm1_labels.py`), served only while the firmware has no live labels.

**Live labels** (they change with the page or mode): the firmware defines two optional native symbols, and the
shim exports them (VST `docs/DYNAMIC-LABELS.md`; `core-api/fm1core.h`):

| Export | Signature | Meaning |
| --- | --- | --- |
| `rfm1_labels_epoch` | () → i32 | moves whenever any label changed; 0 = no live labels (fixed ones apply) |
| `rfm1_label` | (which) → ptr | the label of slot `which`, NUL-terminated, at most 10 characters; 0 = none |

Slots (49): 0..7 the knobs (SELECT ALGO PRESETS K1..K4 MASTER), 8..21 the buttons (FX … OCT+), 22..37 the white keys
W1..W16 (F3 … G5), 38..48 the black keys B1..B11 (F#3 … F#5). Labels are printable ASCII, shown upper case; up to 5
characters draw best.

## Readouts (`rd1`)

**What the user gets:** on the Move, a knob's popup shows the firmware's real value while it turns ("Voicing +1",
"CUTOFF 412 Hz") with a bar, the sound's name when a preset changes, and short messages; with the screen reader on
they are spoken. Without rd1 the Move shows detents ("K1 +3") (MOVE `README.md` "Readouts (rd1)"; CS `README.md`).

**Opt in:** `WANTS rd1`. In the firmware's UI code (never from audio), with `core-api/rfm1_ext.h` vendored:

```c
#include "rfm1_ext.h"
RFM1_READOUT_BAR(-1, "Cutoff", buf, cutoff, 0, 127);   /* ctl -1 = the control in use */
RFM1_PRESET("Preset", name);
RFM1_MESSAGE("Saved");
```

Macros: `RFM1_READOUT(ctl, title, value)`, `RFM1_READOUT_BAR(ctl, title, value, v, lo, hi)`,
`RFM1_READOUT_SUB(ctl, title, value, sub, v, lo, hi)`, `RFM1_PRESET(title, name)`, `RFM1_MESSAGE(text)`. `ctl` is a
label slot (0..6 the knobs) or −1. Without `FM1_HOST_EXT` each is `((void)0)` and its arguments are not evaluated.

**No source changes needed** when the firmware has a parameter map (`knob_target` / `params`): a bridge makes a
readout for each knob it turned. A glue can also read the firmware's drawn knob row (`core-glue/common/rd_cols.h`) or
define the weak adapter `fm1core_rd_knob` (VST `docs/RFM1.md` "Readouts").

**Layout:** export `rfm1_readout() → ptr`, one record at a stable address, read after each `rfm1_frame` when `seq`
moved (VST `core-glue/common/rfm1_shim.c`; MOVE `src/fwrt.c`):

| Offset | Type | Field |
| --- | --- | --- |
| 0 | u16 | version: 1 |
| 2 | u16 | size: 48 |
| 4 | u32 | seq |
| 8 | i8 | control (slot, or −1) |
| 9 | u8 | kind: 1 parameter, 2 preset, 3 message |
| 10 | u8 | flags: bit 0 bipolar |
| 11 | u8 | ms10: how long to show it, in 10 ms (0 = the host's default) |
| 12 | i16 | frac: 0..1000 for the bar, −1 none |
| 14 | char[17] | title |
| 31 | char[17] | value ("value sub") |

**Test:** `rfm1_run --turn K1@2000:3 --readouts`, `--expect-readout <text>` (VST `docs/RFM1.md`).

## Move takeover (`tk1`)

**What the user gets:** the firmware's own Ableton Move interface: its own picture on the Move's 128 × 64 screen, its
own pad and button lights, and the Move's controls going straight to it. The user holds Track n and presses pad 2 to
open it; touching the volume knob and pressing Back always leaves (MOVE `README.md` "Takeover (tk1)").

**Opt in:** `WANTS tk1`, optionally `TAKEOVER_NAME "CHORDS" TAKEOVER_CLAIMS screen pads …` for the manifest's
informative `takeover` entry. In the firmware, with `core-api/rfm1_move.h` vendored, keep the Move code in its own
files or behind `#ifdef FM1_HOST_TK`: `rfm1_tk_register(name, claims, &ops)` once from init (`ops`: `input(pkt[4])`,
`state(on, claims, flags)`), draw with `rfm1_tk_screen()` / `rfm1_tk_screen_done()`, light with `rfm1_tk_led` /
`rfm1_tk_white`, speak with `rfm1_tk_say`. Without the define every call is an inline no-op (VST `docs/RFM1.md`).

**Claims** (each covers the controls' input and their LEDs; VST `docs/RFM1-TAKEOVER.md` §2):
SCREEN 0x01, PADS 0x02, STEPS 0x04, KNOBS 0x08, JOG 0x10, VOLUME 0x20, BUTTONS 0x40, TRACKS 0x80, ALL 0xFF.

**Exports** (all three or none): `rfm1_tk() → ptr`; `rfm1_tk_input(pkt) → i32` (one Move surface packet as a
USB-MIDI u32, only for claimed controls; returns 0); `rfm1_tk_state(on, claims, flags)` (flags bit 0: screen reader
on). Before switching off, the host sends the release of everything held.

**The record** (1468 bytes, little-endian; VST `docs/RFM1-TAKEOVER.md` §4.2): 0 u16 version 1 · 2 u16 size 1468 ·
4 u32 claims · 8 u32 scr_seq · 12 u32 led_seq · 16 u32 say_seq · 20 u8 say_flags (bit 0 interrupt) · 21 u8[3] 0 ·
24 char[16] name · 40 char[128] say · 168 u8[1024] screen (128 × 64, 1 bit a pixel, 8 pages of 128 columns, bit 0 at
the top) · 1192 u8[92][3] LEDs (r, g, b; white LEDs 63..91 use r as brightness). LED order: 0..31 pads, 32..47 steps,
48 Play, 49 Rec, 50 Sample, 51..54 Track 1..4, 55..62 knob rings, 63..75 the white buttons, 76..91 the step icons.

**Manifest:** `"wants": ["tk1"], "takeover": {"name": "CHORDS", "claims": ["screen", "pads", …]}`; `name` at most 15
printable ASCII characters. A later version adds fields only at the end; a later firmware may name `tk2` and keep the
tk1 exports working (VST `docs/RFM1-TAKEOVER.md` §3, §7).

**Test:** `rfm1_run --tk --tk-ev pad:5@200+50 --tk-screen out.pbm`; the test firmware `tkdemo` (VST
`tools/rfm1/tkdemo`). Follow the Move controller's UX rules (CS `design/UX-RULES.md`).

## Engine parameters (`ep1`)

**What the user gets:** on the Move, an ENGINE knob page after the FM-1 page: a list of the firmware's sound-engine
pages, up to eight parameters each, set absolutely with the Move's knobs; layers chosen by MIDI channel (MOVE
`README.md` "Engine parameters (ep1)"; CS `README.md` "Engine parameters (0.14.0)").

**Opt in:** `WANTS ep1`, and in the firmware (`core-api/rfm1_ep.h` vendored) a weak `const rfm1_ep_src_t
*fm1core_ep(void)` giving the layers, engines, pages and parameters with `get` / `set` callbacks. Parameter ids are
frozen: a pid's meaning never changes.

**Exports** (all three or none): `rfm1_ep() → ptr` (the EP1R record, 264 bytes); `rfm1_ep_set(op, layer, pid, value)
→ i32` (op 1 SET; 0 done, −1 no such layer or pid); `rfm1_ep_watch(layer, page)`. The record holds the layers'
engines and MIDI channels, every value of every layer, and the watched page's eight cells (name and value words); each
part has its own `seq`. Field table: VST `docs/RFM1-FORMAT-ep1.md`.

**Status:** released on the Move in 0.4.0 for the SET operation; ChoralRoot is the only firmware using it. The binary
description it points to (EP1D) is summarised in VST `docs/RFM1-FORMAT-ep1.md`, but its field-by-field specification
(engine-params `SPEC.md`) **is not published yet**, and further operations are in development. Treat ep1 as early: ask
the RemoteFM1 maintainer before building on it.

## Second core (`cpu2`)

**What the user gets:** on the Move, a firmware that splits work across the FM-1's two cores (voices on core 1) runs
that part on its own thread, so heavier firmwares fit. Elsewhere it runs inline, as built.

**Opt in:** `WANTS cpu2`. Where the device hands a job to core 1, the firmware's host back end calls
`rfm1_cpu2_run(fn, arg)` (1: handed over; 0: run it yourself), then `rfm1_cpu2_wait()` (0: done; else run it yourself
in the same block); `rfm1_cpu2_core()` says whether code is running on core 1. Rules: one job at a time, core 0 leaves
the job's data alone until the wait, a job never hands a job on, never yields, never allocates, and the samples must
not depend on which side ran it (VST `docs/RFM1.md` "Core 1"; `core-glue/common/rfm1_cpu2.h`).

**Exports** (all three or none): `rfm1_cpu2() → ptr` gives `{u16 version 1, u16 size 12, u32 run, u32 wait}`: two
function-table indices the host may overwrite with its own after instantiating; `rfm1_c1_stack(top)` (at least
64 KiB); `rfm1_c1_call(fn, arg)`, run on a second instance sharing the memory. A trapping job is reported not done and
the module runs it itself.

## Audio input (`ain1`)

**What the user gets:** live audio (the Move's line in or mic; the computer's or phone's input) reaches the firmware
— a sampler, an effect, a looper.

**Opt in:** `WANTS ain1`, with the core's audio-in hook. **Exports:** `rfm1_audio_in(ptr, frames)` (the input block,
called right before each `rfm1_audio`) and `rfm1_audio_in_info(state, latency)` (called when the input starts, stops
or changes: `(1, latency)` before the first block, `(0, 0)` on stop). `ain1` must be in `wants` (VST
`docs/VIRTUAL-CARTS.md` §8; MOVE `README.md`). On the Move the round trip is a fixed 1152 frames.

**Test:** `rfm1_run --audio-in in.wav`.

## More output channels (`aout1`)

**What the user gets:** separate tracks (drums, bass…) on their own outputs in a DAW, as fm1-base's USB audio gives
them on a computer. RemoteFM1's AUv3 has Main 1-2 and Out 3-4 … 9-10 (IOS `docs/RFM1-FORMAT.md` §9).

**Opt in:** `fm1_add_rfm1(… AUDIO_OUT 4|6|8|10)`, with the core's `fm1core_audio_out_info` / `fm1core_audio_out`.
**Exports:** `rfm1_audio_out_info() → n` (4, 6, 8 or 10; else just the master) and `rfm1_audio_out(buf, frames)`
(channels 3..n for the next `rfm1_audio`: `frames` frames of `n − 2` int16, zeroed by the host, the firmware adds into
it at the master's scale). The host asks once after init and calls it right before each `rfm1_audio` (after
`rfm1_audio_in`). Not checked by `rfm1_check.py`.

## Fantasy CFW (`virtual`, `fs1`)

**What the user gets:** a "Fantasy CFW": a firmware that only exists in RemoteFM1 hosts, never on an FM-1, with far
more memory and access to a shared folder of samples and projects (Elasticat, Ripple). Hosts show a badge (VST
`docs/VIRTUAL-CARTS.md`).

**Opt in:** `fm1_add_rfm1(<id> VIRTUAL NEEDS fs1 FOLDERS audio=r data=rw MEMORY_MIB 280 …)`. The manifest says
`"virtual": true`, `"needs": ["fs1"]` and `"folders"`. Limits: memory up to 512 MiB (fixed), the file under 16 MiB,
the flash image up to 4 MiB; at least 2× real time in wasm3 on an Apple-silicon Mac.

**The shared folder:**

```
RemoteFM1/
  audio/        read by every cart
  code/<id>/    the cart's own, read-only  (seen as /code)
  data/<id>/    the cart's own, read-write (seen as /data)
```

No network, no `..`. **The file API** is imported from module `rfm1_host_1`: `fs_request`, `fs_poll`, `fs_cancel`,
`fs_changed` (exact signatures: VST `core-glue/common/rfm1_host.h`). Operations STAT, LIST, READ, WRITE, DELETE, MKDIR;
at most 32 requests in flight, 1 MiB each, answered asynchronously between calls (VST `docs/VIRTUAL-CARTS.md` §4;
MOVE `README.md` "Fantasy CFWs"). On the Move a Fantasy CFW's whole memory is reserved at load, and the load is
refused when memory is short.

**Test:** `rfm1_run --folder DIR --press PLAY@3000 --expect-ui TEXT`.

**Catalog:** RFM1Manager copies `virtual` from the manifest into the catalog; the catalog README marks such entries ‡
(FW `catalog/SCHEMA.md`, `README.md`).

## Screen reader protocol

**What the user gets:** on the Move, with its screen reader on, the firmware's own words are spoken (what the screen
shows, not just knob numbers).

**Opt in:** answer the host's HELLO over SysEx, as specified by
[charlesvestal/fm1-reader `PROTOCOL.md`](https://github.com/charlesvestal/fm1-reader) (fm1-base firmwares get it
from its `announce.c`). Frames start `F0 7D 53 52`: the host sends HELLO `F0 7D 53 52 01 01 F7` every 2 s, BYE
(`… 02 F7`) and WHERE (`… 03 F7`); the firmware answers with HELLO (its name and a words-file URL) and SAY
(`… 10 <flags> <text> F7`, at most 240 characters). The Move removes these frames from MIDI out (MOVE
`src/fmreader.h`, `README.md` "Screen reader"). The protocol's own document is the reference; these docs only
summarise it.

## Working memory and states (host-side)

**What the user gets:** on the Move, a saved state brings back the exact running firmware, not just its flash.

**Opt in:** nothing to export. The Move snapshots a firmware's whole memory only if the firmware passes the host's
`tools/snapsafe.py` check (no table or passive-segment changes, no exception handling, no imported globals, memories
or tables, not a Fantasy CFW) **and** is listed in the host's `src/snapsafe.h`. A firmware published after a Move host
release gets flash-only states until the maintainer re-runs the check and releases a new host (MOVE `README.md`
"The working memory"). Authors keep the default build rules and are covered.

## INFO / PING and other SysEx

There is **no RemoteFM1 "INFO / PING" message for firmwares**. Several unrelated protocols use the non-commercial
manufacturer id `0x7D`, each told apart by the next two bytes:

| Prefix | What | Source |
| --- | --- | --- |
| `F0 7D 52 46` | RemoteFM1's own host ↔ FM-1 protocol (discovery is HELLO `0x01` → HELLO_R `0x41`) | SITE `docs/PROTOCOL.md` |
| `F0 7D 46 4C` | Felucca's editor protocol ("FL": INFO is command 1, `F0 7D 46 4C 01 F7`; PING is command 25, `… 19 F7`) — a firmware's own, passed through by the hosts; its forks use it too | hugelton/Felucca `web/EDITOR_PROTOCOL.md`; MOVE `tests/sim_e2e.sh` |
| `F0 7D 53 52` | the screen reader protocol (above) | charlesvestal/fm1-reader |

A firmware's editor SysEx is passed between the firmware and its web editor unchanged by every host (the Move: MOVE
`tests/page_e2e.mjs`). Pick a two-byte sub-id no one else uses, and never send `F0 00 59 …` (the M-UPGRADE updater).
The catalog's `port` field lets an editor find the in-app MIDI port by name (FW `catalog/SCHEMA.md`).
