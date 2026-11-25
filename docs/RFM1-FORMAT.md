# The `.rfm1` file, version 1

This is the complete description of an `.rfm1` file as the RemoteFM1 hosts read it and as the public tools build and
check it. Citations name the repository and file (see [SOURCES.md](SOURCES.md) for the commits). The format's
original text is IOS `docs/RFM1-FORMAT.md`; the checker that every catalog file must pass is VST
`tools/rfm1_check.py`. Where they differ, this document follows the checker and says so.

Contents: [1 What it is](#1-what-it-is) · [2 The container](#2-the-container) · [3 The manifest](#3-the-manifest-rfm1-section) ·
[4 Exports](#4-exports) · [5 Imports](#5-imports) · [6 `rfm1_layout`](#6-rfm1_layout) ·
[7 Allowed WebAssembly](#7-the-webassembly-a-module-may-use) · [8 How a host runs it](#8-how-a-host-runs-it) ·
[9 Versions, hashes and signing](#9-versions-hashes-and-signing) · [10 File name](#10-file-name)

Optional parts (skin, labels, readouts, takeover, engine parameters, cpu2, audio in and out, Fantasy CFW) are in
[CAPABILITIES.md](CAPABILITIES.md). The C interface the firmware itself implements is in [HOST-ABI.md](HOST-ABI.md).
The Move's compiled form is in [RFM1M.md](RFM1M.md).

## 1. What it is

An `.rfm1` is an FM-1 firmware built to run inside a host app instead of on the FM-1. It is the firmware's host build
(the `fm1core` ABI, [HOST-ABI.md](HOST-ABI.md)) compiled to **WebAssembly**. Hosts that load `.rfm1` files: remotefm1.com
(the browser's own WebAssembly, in an AudioWorklet: SITE `web/engines/rfm1/engine.js`), RemoteFM1 for iOS and its
AUv3 (wasm3: IOS), RemoteFM1 for Android (its catalog firmwares are `.rfm1`: RemoteFM1Android `README.md`), the test
runner `rfm1_run` (wasm3), and the Ableton Move, which compiles each file ahead of time ([RFM1M.md](RFM1M.md)). The
FM1VST desktop plugin builds its firmwares natively from the same glue; the code read for these docs has no `.rfm1`
loader in the plugin. One file runs on every platform, and each instance gets its own memory. (IOS
`docs/RFM1-FORMAT.md` intro; VST `docs/RFM1.md` intro)

An `.rfm1` never changes an FM-1: nothing is installed on a unit. (FW `README.md`)

## 2. The container

A standard WebAssembly binary module: magic `00 61 73 6D` (`\0asm`), version `01 00 00 00`. On top of the code it
carries **custom sections** (WebAssembly section id 0, a name, then bytes). (IOS §1; VST `tools/rfm1_pack.py`)

| Custom section name | Contents | Required |
| --- | --- | --- |
| `rfm1` | the manifest: UTF-8 JSON (§3) | **yes**. A module without it is not an `.rfm1` |
| `rfm1.credits` | UTF-8 text, the credits a host shows in its About screen | no (written when the descriptor has credits) |
| `rfm1.license.0`, `rfm1.license.1`, … | one licence text each, UTF-8, the first line its name (non-empty) | the checker accepts none, but the catalog requires at least one (FW `docs/AUTHOR-RFM1.md`; MGR `lib/rfm1.js`) |
| `rfm1.skin` | UTF-8 JSON, the skin ([CAPABILITIES.md](CAPABILITIES.md#skin-rfm1skin)) | no |
| `rfm1.skin.background` | PNG or JPEG bytes | no |
| `rfm1.skin.background.landscape` | PNG or JPEG bytes | no |

How `tools/rfm1_pack.py` writes them (VST `tools/rfm1_pack.py`):

1. It checks the magic and version, then **removes every existing custom section named `rfm1` or `rfm1.*`**, so
   packing the same module twice gives the same bytes.
2. It appends, in this order: `rfm1`, `rfm1.credits` (only if there are credits), `rfm1.license.0..n`, then the skin
   sections. Identical licence texts (ignoring whitespace) are merged into one section whose name line joins the names
   with `; `.
3. It writes to a temporary file and moves it into place.

Hosts may find the sections anywhere in the file. A section a host doesn't know is ignored; `rfm1_check.py` warns
about an unknown custom section and about licence sections not numbered 0..n (VST `tools/rfm1_check.py`).

## 3. The manifest (`rfm1` section)

```json
{
  "rfm1": 1,
  "abi": 6,
  "id": "felucca",
  "name": "Felucca",
  "version": "1.6.1",
  "license": "GPL-3.0-only",
  "source_url": "https://github.com/hugelton/Felucca@<commit>",
  "authors": ["Leo Kuroshita (Hügelton Instruments)"],
  "built_by": "FM1VST 0.6.2 (<commit>) fm1_add_rfm1, emcc 6.0.13",
  "memory_mib": 8
}
```

(Example from IOS §4, `built_by` in the form VST `cmake/FM1Rfm1.cmake` writes.)

| Field | Type | Required | Rule | Source |
| --- | --- | --- | --- | --- |
| `rfm1` | number | yes | must be `1` | VST `tools/rfm1_check.py` |
| `abi` | number | yes | must be `6` for `rfm1_check.py`. (The Move's `rfm1info` accepts 5 or 6.) | VST `tools/rfm1_check.py`; MOVE `tools/rfm1info.c` |
| `id` | string | yes | `[a-z0-9_]{1,31}`; matches the descriptor's id | IOS §4; VST `tools/rfm1_pack.py` |
| `name` | string | yes | shown by hosts | IOS §4 |
| `version` | string | yes | shown by hosts; the catalog compares it to the release tag (§9) | IOS §4; FW `docs/AUTHOR-RFM1.md` |
| `license` | string | yes | an SPDX expression (the checker warns if it doesn't look like one) | IOS §4; VST `tools/rfm1_check.py` |
| `source_url` | string | yes | `https://…/<owner>/<repo>@<commit>`: the exact source the file was built from. The checker warns unless it is http(s) and ends in `@` + 7-40 hex digits; the catalog requires that form | IOS §4; VST `tools/rfm1_check.py`; MGR `lib/rfm1.js` |
| `authors` | array of strings | no (the catalog requires a non-empty list) | shown by hosts | IOS §4; MGR `lib/rfm1.js` |
| `built_by` | string | no | what built it: `FM1VST <ver> (<commit>) fm1_add_rfm1, emcc <ver>` or `fm1-base <commit> tools/build.py --rfm1, emcc <ver>` | VST `cmake/FM1Rfm1.cmake`; BASE `tools/build.py` |
| `memory_mib` | number | no | must equal the module's memory size | VST `tools/rfm1_check.py` |
| `glue_url` | string | no | the build glue at a commit. Listed by IOS §4, **not written by the current tools** | IOS §4; VST `tools/rfm1_pack.py` |
| `wants` | array of strings | no | host capabilities the module uses where a host has them: `rd1`, `tk1`, `ep1`, `cpu2`, `ain1`. Informative: hosts ignore names they don't know. The checker requires the name when the matching exports are there | VST `docs/RFM1.md`, `tools/rfm1_check.py` |
| `takeover` | object | no | `{"name": "…", "claims": [...]}`, only with `tk1` | VST `docs/RFM1-TAKEOVER.md` §3 |
| `virtual` | boolean | no | `true` marks a Fantasy CFW (host-only) | VST `docs/VIRTUAL-CARTS.md` |
| `needs` | array of strings | no | with `virtual`: capabilities the module cannot run without (`fs1`) | VST `docs/VIRTUAL-CARTS.md` |
| `folders` | object | no | with `virtual`: shared-folder access, e.g. `audio=r`, `data=rw` | VST `docs/VIRTUAL-CARTS.md` |

Hosts show `name`, `version`, `license`, `authors`, `source_url` and the credits. A host may refuse a module whose
manifest and descriptor disagree on `id`, `name` or `version` (IOS §4). The tools take id, name, version, licence and
source URL from the firmware's own descriptor, so they always match (VST `docs/RFM1.md`).

## 4. Exports

All pointers are offsets into the module's one memory (wasm32). Every value is i32. Each export calls the descriptor's
function of the same name ([HOST-ABI.md](HOST-ABI.md)); they exist because a host cannot call the descriptor's
function pointers through an interpreter (IOS §2). The signatures are the ones `rfm1_check.py` enforces (VST
`tools/rfm1_check.py`, `EXPORTS`).

| Export | Params → results | What it does |
| --- | --- | --- |
| `memory` | memory | the module's one linear memory |
| `_initialize` | () → () | optional; called once, first (a WASI reactor's constructors) |
| `rfm1_version` | () → i32 | the format version: `1` |
| `rfm1_get` | (abi) → ptr | the descriptor for that ABI, or 0 |
| `rfm1_layout` | (i) → i32 | struct sizes and offsets (§6); −1 past the end |
| `rfm1_alloc` / `rfm1_free` | (n) → ptr / (ptr) → () | memory for strings and buffers the host passes in |
| `rfm1_options` | (path, no_flash, save, headless) → () | the descriptor's `options` |
| `rfm1_boot_options` | (fail, reason, stage) → i32 | `boot_options`; 1 if the descriptor has none |
| `rfm1_init` | (demo) → () | `init` |
| `rfm1_tick` | (ms) → () | `tick` |
| `rfm1_frame` / `rfm1_idle` | () → () | `frame` / `idle` |
| `rfm1_audio` | (out, frames) → () | `frames` stereo int16 frames written at `out` |
| `rfm1_midi_in` | (pkt) → i32 | a USB-MIDI packet as a u32 |
| `rfm1_midi_out_take` | (pkt_out) → i32 | the next outgoing packet |
| `rfm1_halted` | () → i32 | 0 while running |
| `rfm1_shutdown` | () → () | `shutdown` |
| `rfm1_flash_stage` | (bytes, n) → i32 | stage a flash image before init |
| `rfm1_flash` | () → ptr | the flash image (`flash_size` bytes) |
| `rfm1_flash_dirty` / `rfm1_flash_sync` | () → i32 | `flash_dirty` / `flash_sync` |
| `rfm1_stats` | (shed_out, cpu_out) → () | `stats` |
| `rfm1_ui_info` | (buf, n) → () | `ui_info` |

Optional exports, each described in [CAPABILITIES.md](CAPABILITIES.md): `rfm1_labels_epoch`, `rfm1_label` (live
labels); `rfm1_readout` (rd1); `rfm1_tk`, `rfm1_tk_input`, `rfm1_tk_state` (tk1); `rfm1_ep`, `rfm1_ep_set`,
`rfm1_ep_watch` (ep1); `rfm1_cpu2`, `rfm1_c1_stack`, `rfm1_c1_call` (cpu2); `rfm1_audio_in`, `rfm1_audio_in_info`
(ain1); `rfm1_audio_out_info`, `rfm1_audio_out` (aout1). Hosts ignore exports they don't know, including a
toolchain's own (`emscripten_stack_*`, `__indirect_function_table`) (IOS §1).

The host calls every export from one thread at a time (IOS §2). The descriptor's parameter functions are not
reachable through version 1 exports (IOS §2).

## 5. Imports

Only these six, all from module `wasi_snapshot_preview1`, with exact signatures. A module that imports anything else
is refused (IOS §3; VST `tools/rfm1_check.py`). Imported tables, memories, globals or tags are errors.

| Import | Host behaviour |
| --- | --- |
| `fd_write` | stdout / stderr go to the host's log (rate-limited); other fds: EBADF |
| `fd_read`, `fd_close`, `fd_seek` | EBADF |
| `clock_time_get` | the host's monotonic clock. Sound and screen must not depend on it: a firmware's time is `tick`'s |
| `proc_exit` | the instance halts (`halted` = 0x100) |

No file system, network, randomness or threads. The one exception is a Fantasy CFW, which may also import the file
API from module `rfm1_host_1` ([CAPABILITIES.md](CAPABILITIES.md#fantasy-cfw-virtual-fs1)); any other module that
imports `rfm1_host_1` fails the check (VST `tools/rfm1_check.py`).

## 6. `rfm1_layout`

`rfm1_layout(i)` returns, by index (IOS §5; VST `core-glue/common/rfm1_shim.c`):

| i | Value |
| --- | --- |
| 0 | `sizeof(fm1panel_t)` |
| 1-16 | offsets in `fm1panel_t` of `keys`, `buttons`, `keys_tap`, `buttons_tap`, `enc`, `master`, `key_vel`, `key_lean`, `led`, `led_play_green`, `lcd`, `lcd_writes`, `btn_id`, `enc_id`, `enc_dir`, `ready` |
| 17 | `sizeof(fm1core_t)` |
| 18-33 | offsets in `fm1core_t` of `abi_version`, `id`, `name`, `version`, `source_url`, `family`, `license`, `gated`, `flash_size`, `button_names`, `params`, `nparams`, `panel`, `keymap`, `options`, `credits` |
| 34+ | −1 |

A host checks every value it uses against its own expectation and refuses a mismatch. It reads only data from the
descriptor (strings, the panel, the keymap, `flash_size`, button names), never its function pointers.

## 7. The WebAssembly a module may use

(IOS §6; VST `tools/rfm1_check.py`, `cmake/FM1Rfm1.cmake`)

- **One memory**, defined and exported as `memory`, **fixed size** (initial = maximum, no growth), not shared, not
  64-bit. At most **32 MiB** (8 MiB is the default). A Fantasy CFW may use up to 512 MiB.
- **Features:** the MVP plus sign-extension, non-trapping float-to-int, mutable globals, multi-value, and of bulk
  memory only `memory.copy` and `memory.fill`. The build uses `-mcpu=mvp -msign-ext -mnontrapping-fptoint
  -mbulk-memory -mmutable-globals`.
- **Not allowed** (the checker decodes every function and rejects them): SIMD, threads / atomics, reference types,
  extended-const, tail calls, exception handling, `memory.init`, `data.drop`, `table.*`, passive data segments,
  non-MVP element segments, memory64. Data segments are active, in memory 0. (wasm3 0.5.0, the iOS interpreter, runs
  nothing more.)
- **Same sound everywhere:** compile with `-ffp-contract=off` (no fused multiply-add). The build adds it.
- Warnings, not errors: `memory.grow` present, a start function.

## 8. How a host runs it

(IOS §7)

1. Read the file; check the magic, the `rfm1` section, the imports, the exports and `rfm1_version() == 1`.
2. Instantiate it, one instance per firmware instance. Call `_initialize` if exported.
3. `rfm1_get(6)`; check `rfm1_layout`; read the descriptor's data.
4. Boot from a fresh flash: `rfm1_flash_stage(0, 0)`; `rfm1_options(p, 0, 0, 1)` with `p` the string
   `"fm1core:staged"` placed in memory with `rfm1_alloc`; `rfm1_boot_options(-1, 0, -1)`; set the panel's `ready` to
   2; `rfm1_init(0)`. From a saved flash: stage its bytes instead of `(0, 0)`.
5. Drive it as a native core ([HOST-ABI.md](HOST-ABI.md#timing)): `tick` every device millisecond, `frame` every 15 ms
   (the Move: 33 ms on screen, 200 ms when not shown; MOVE `README.md` "Frames"), `idle` otherwise, `audio` per
   128-frame block at 44.1 kHz. Panel reads and writes go straight to linear memory.
6. Saved state: the firmware's id, version and the file's SHA-256, plus the flash image. Restoring with a different
   file of the same id, the host warns and boots the flash anyway; the firmware decides what its flash means.

## 9. Versions, hashes and signing

- **Format version:** `rfm1_version()` and the manifest's `rfm1` are both 1. A later format changes them. Capability
  records each carry their own `version` and `size`, and grow only at the end (VST `docs/RFM1-TAKEOVER.md` §7).
- **ABI version:** the manifest's `abi` and `rfm1_get`'s argument: 6 ([HOST-ABI.md](HOST-ABI.md)).
- **No signature, no hash inside the file.** `rfm1_pack.py` writes only the manifest, credits, licences and skin
  (VST `tools/rfm1_pack.py`).
- **Hashes outside the file:** hosts identify a file by its SHA-256 (iOS stores imports as `<sha256>.rfm1` and checks
  downloads against size and SHA-256: IOS `RFMKit/Sources/RFMKit/Firmware.swift`, `App/GetFirmware.swift`). The
  catalog release carries `SHA256SUMS`, and remotefm1.com rejects the whole release on any mismatch (MGR `README.md`
  §7). Adding a skin to a built file changes its SHA-256 (IOS §8).
- **Signing exists only for the Move:** its catalog `index.json` is signed with the maintainer's Ed25519 key, whose
  public half is compiled into the Move host ([RFM1M.md](RFM1M.md)).
- **Versions in the catalog:** the manifest's `version` must equal the release tag's (`v1.2.3` → `1.2.3`) and be
  newer than the listed one. Pre-release words (rc, beta, alpha, pre, preview, dev, nightly, test, snapshot, draft,
  wip) make a tag not stable (MGR `lib/util.js`; FW `tools/author_rfm1_sync.mjs`). Apps rank versions by semver, so
  a rebuild named `0.42-r2` ranks below `0.42`; a same-version rebuild is offered by its SHA-256 (VST commit `6f16a01`
  message; MOVE `README.md` "A rebuild of the same version").

## 10. File name

`<id>-<version>.rfm1`, e.g. `felucca-1.6.1.rfm1`. Hosts don't rely on the name (IOS §1). The catalog renames an
author's file to that form and never changes its bytes (FW `docs/AUTHOR-RFM1.md`).
