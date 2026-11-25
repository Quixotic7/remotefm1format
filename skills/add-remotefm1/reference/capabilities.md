# Optional capabilities: how to add each

Full text and byte layouts: https://github.com/Quixotic7/remotefm1format/blob/main/docs/CAPABILITIES.md

**Route A (fm1-base) gives only the base exports**: its `host/rfm1.c` has none of the optional ones (no live labels,
rd1, tk1, ep1, cpu2, ain1, aout1), so on route A only the skin (added after the build) and the screen reader apply.
Every other capability needs route B (FM1VST glue).

**Ask the author before each one.** After adding one: rebuild, `rfm1_check` (it enforces the pairing of exports and
`wants`), `rfm1_run`, plus the test named below. Device builds must stay byte-identical: firmware-side calls are
macros or inline no-ops unless the host build's define is set.

Headers to vendor (MPL-2.0) come from FM1VST at commit `e50e77ec41dedd8d8c041186a7d01978c80570e2`:
`core-api/rfm1_ext.h` (rd1), `core-api/rfm1_move.h` (tk1), `core-api/rfm1_ep.h` (ep1),
`core-glue/common/rfm1_cpu2.h` (cpu2). Copy them into the firmware tree with their licence header intact.

## Card, skin, fixed labels (`RemoteFM1Metadata.json`)

Start from [metadata-template.json](metadata-template.json). Card: `tag` ≤ 40 chars, `what` ≤ 600, `category` one of
Synth / Groovebox / Chords / Drums / Other, `site` / `editor` https, `port` = the USB MIDI product name exactly,
`image` square PNG/JPEG/WebP ≤ 1 MiB (512 px). Skin: `background` / `landscape` PNG or JPEG ≤ 4 MiB, ≤ 4096 px;
`dim` 0..0.8; `theme` exactly 5 roles (`body key knob led bezel`) or exactly 14. Labels: buttons `FX SEL ENV LFO EDIT
GLO HOME SAVE ARP SEQ PLAY REC OCT- OCT+`, knobs `SELECT ALGO PRESETS K1 K2 K3 K4 MASTER`, ≤ 10 ASCII each (≤ 5 draws
best), `""` = unused. Images must be the author's own and committed in the release. Test: route B packs skin and
labels at build (`rfm1_check` reports them); route A: `rfm1_skin.py add … --metadata RemoteFM1Metadata.json` then
`rfm1_skin.py show`.

## Live labels

Two optional native functions in the host build (the shim exports them as `rfm1_labels_epoch` / `rfm1_label`):

```c
uint32_t fm1core_labels_epoch(void);      /* change it whenever any label changes; 0 = no live labels */
const char *fm1core_label(int32_t which); /* slot 0..48 -> label (<= 10 chars) or NULL */
```

Slots: 0..7 knobs (SELECT ALGO PRESETS K1..K4 MASTER), 8..21 buttons (FX … OCT+), 22..37 white keys, 38..48 black
keys. Test: `rfm1_run --describe --labels`.

## Readouts `rd1`

Route B: add `WANTS rd1` to `fm1_add_rfm1` (an FM1VST change). If the glue has a parameter map (`params` /
`knob_target`) readouts come with no firmware change. Otherwise, in the firmware's UI code (never in audio):

```c
#include "rfm1_ext.h"           /* no-ops unless FM1_HOST_EXT */
RFM1_READOUT_BAR(-1, "Cutoff", text, value, 0, 127);
RFM1_PRESET("Preset", preset_name);
RFM1_MESSAGE("Saved");
```

Test: `rfm1_run file --turn K1@4100:3 --seconds 1 --readouts` prints `readout@… ctl … "TITLE" "VALUE"`.

## Move takeover `tk1`

Big feature: the firmware draws the Move's 128×64 screen and LEDs and takes the Move's controls it claims. Needs
`WANTS tk1` (+ `TAKEOVER_NAME`, `TAKEOVER_CLAIMS`). Put the Move UI in its own files or behind `#ifdef FM1_HOST_TK`;
register once from init with `rfm1_tk_register(name, claims, &ops)`. Read `docs/CAPABILITIES.md` and FM1VST
`docs/RFM1-TAKEOVER.md` first, and the Move UX rules (rfm1cs-abletonmove `design/UX-RULES.md`). Test:
`rfm1_run file --tk --tk-ev pad:5@200+50 --tk-screen -`. Plan it with the author before writing code.

## Engine parameters `ep1`

Early (released on the Move 0.4.0 for SET only; its full description spec is not published). Recommend the author
asks the RemoteFM1 maintainer before starting. Needs `WANTS ep1` and a `fm1core_ep()` table with frozen parameter ids.

## Second core `cpu2`

Only for firmwares that hand work to the FM-1's core 1. `WANTS cpu2`; where the device starts core 1's job, call
`rfm1_cpu2_run(fn, arg)` (0 → run it yourself) and later `rfm1_cpu2_wait()` (non-zero → run it yourself). One job at a
time, no allocation, no yielding, same samples either way. Test: `rfm1_run` must sound identical.

## Audio input `ain1`

`WANTS ain1`, with the core's audio-in hook (FM1VST `docs/VIRTUAL-CARTS.md` §8). Test: `rfm1_run file --audio-in in.wav`.

## Fantasy CFW

A firmware for RemoteFM1 only (no FM-1 build), with up to 512 MiB and the shared folder (`audio/`, `code/<id>/`,
`data/<id>/`). `fm1_add_rfm1(<id> VIRTUAL NEEDS fs1 FOLDERS audio=r data=rw MEMORY_MIB <n>)`. Only relevant if the
author is making one; it changes the whole design. Test: `rfm1_run file --folder DIR --press PLAY@3000`.

## Screen reader

fm1-base firmwares get it from `announce.c` and `fm1.toml` `words`. Others: implement the HELLO / SAY SysEx of
https://github.com/charlesvestal/fm1-reader `PROTOCOL.md`.
