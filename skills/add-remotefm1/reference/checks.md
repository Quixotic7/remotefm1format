# Checks and common failures

Full text: https://github.com/Quixotic7/remotefm1format/blob/main/docs/TESTING.md and `docs/RFM1-FORMAT.md`.

## What the catalog checks

`rfm1_check.py` ok · `rfm1_run` exit 0 (no trap or halt, chord RMS ≥ 0.0005, the screen drew) · manifest `id`,
`name`, `version`, `license`, non-empty `authors`, `source_url` = `https://github.com/<owner>/<repo>@<7-40 hex>` ·
at least one `rfm1.license.<n>` · `version` equals the release tag (`v1.2.3` → `1.2.3`) and is newer than the listed
one · a stable release (no rc / beta / alpha / pre / preview / dev / nightly / test / snapshot / draft / wip in the
tag; not a GitHub pre-release or draft).

## Reading the output

- `rfm1_check.py file.rfm1` → `file: ok: <id> <version> (<licence>), <capabilities>, <bytes>, <n> MiB memory, <n> exports, <n> licence texts`.
  Exit 1 on an error, 2 on a usage or file problem.
- `rfm1_run file.rfm1` → `<id> <version>: chord rms … (peak …), <n> LCD writes, <x>x real time …`. Useful flags:
  `--wav out.wav`, `--screen out.ppm`, `--seconds N`, `--settle MS`, `--keys K,..`, `--press LABEL@MS`,
  `--sysex bank.syx`, `--describe`, `--help`.

## Common failures

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `imports … not allowed` | the build pulled in something outside WASI's six (file I/O, `emscripten_*`, threads) | keep the host build free of file system and threads; `-sFILESYSTEM=0` is set; find the caller with `wasm-objdump -x` |
| SIMD / atomics / exceptions found | flags or a library using them | build with the route's flags; no `-msimd128`, no `-pthread`, no C++ exceptions |
| memory too big / not fixed | `MEMORY_MIB` / `rfm1_memory_mib` too high, or growth enabled | ≤ 32 MiB, fixed |
| link says the data does not fit | memory too small | raise `MEMORY_MIB` / `rfm1_memory_mib` |
| `rfm1_run`: silent | the firmware needs time or input to make sound (a sequencer, a looper) | `--settle`, `--keys`, `--press PLAY@MS`; tell the catalog it is silent by design |
| `rfm1_run`: never drew | the firmware draws only on events | press a key with `--keys`; check `frame()` runs |
| trap | an out-of-bounds access or `abort()` the device survives | fix the bug; `--verbose` and a debug build (`FM1_RFM1_NAMES=ON`, never for a published file) |
| `source_url` warning | dirty tree, or the URL isn't `…@<commit>` | build from a clean, pushed commit |
| version ≠ tag | the version string in the source wasn't bumped | bump it before tagging |
| licence texts missing | no LICENSE file found | add one; route B: `LICENSE_FILES` |
| different sound on another host | fused multiply-add | `-ffp-contract=off` is set by both routes; don't override it |
