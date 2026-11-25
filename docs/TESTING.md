# Worked example and test checklist

## Worked example: Felucca 1.8.2 → `felucca-1.8.2.rfm1`

Felucca already has FM1VST glue (`core-glue/felucca_web/core_felucca.cmake`), so this is route B of
[BUILDING.md](BUILDING.md) with nothing to write. It was run for these docs on macOS (Apple silicon) with Emscripten
6.0.11 (6.0.13 is the tested version), CMake and Python 3 with Pillow and fontTools.

```sh
# 1. FM1VST at a commit that has the .rfm1 toolchain
git clone https://github.com/Quixotic7/FM1VST && cd FM1VST
git checkout e50e77ec41dedd8d8c041186a7d01978c80570e2

# 2. only the firmware's submodule (a full fetch: a shallow one may miss the pinned commit)
git submodule update --init --recursive cores/Felucca

# 3. configure just this core, with the .rfm1 on, and build it and the test runner
source ~/emsdk/emsdk_env.sh
cmake -S . -B build-rfm1 -DFM1_RFM1_ONLY=ON -DFM1_PUBLIC_BUILD=ON -DFM1_ONLY_CORES=felucca
cmake --build build-rfm1 --target rfm1_felucca rfm1_run -j8
```

The build prints:

```
[100%] rfm1 felucca: felucca-1.8.2.rfm1
rfm1_pack: …/build-rfm1/rfm1/felucca-1.8.2.rfm1: felucca 1.8.2, 1126132 bytes (13 licence texts)
```

```sh
# 4. check it against the format
python3 tools/rfm1_check.py build-rfm1/rfm1/felucca-1.8.2.rfm1
#   …: ok: felucca 1.8.2 (GPL-3.0-only), readouts (rd1), 1126132 bytes, 8 MiB memory, 60 exports, 13 licence texts

# 5. run it as an app would: boot, 4 s, then a D major chord for 5 s
build-rfm1/rfm1_run build-rfm1/rfm1/felucca-1.8.2.rfm1
#   felucca 1.8.2: chord rms 0.0644 (peak 0.252), 312 LCD writes, 35.3x real time (chord 30.3x), …

# 6. its capability: turn K1 three detents at 4.1 s and print the readout
build-rfm1/rfm1_run build-rfm1/rfm1/felucca-1.8.2.rfm1 --turn K1@4100:3 --seconds 1 --readouts
#   readout@4110: seq 1 ctl 3 kind 1 flags 0 frac 417 "E5 CUT" "412 Hz"

# 7. or both through ctest
ctest --test-dir build-rfm1 -R '^rfm1_(rd_)?felucca$' --output-on-failure
```

(`ctest -R rfm1_` also runs the takeover demo's tests, which fail unless you build `rfm1_all`: build `--target
rfm1_all` or filter as above.)

That file is what an author attaches to a release. For your own firmware, replace `felucca` with your id and
`cores/Felucca` with your submodule; if FM1VST has no glue for it yet, see [BUILDING.md](BUILDING.md#b-fm1vst-fm1_add_rfm1).
An fm1-base app takes one command instead of steps 1-3 (route A), then the same checks.

## Checklist

### The file

- [ ] `python3 tools/rfm1_check.py yourfw-X.Y.Z.rfm1` says `ok` (FM1VST at `e50e77e` or later; standard library only).
- [ ] `rfm1_run yourfw-X.Y.Z.rfm1` exits 0: no trap, no halt, the chord sounds (RMS ≥ 0.0005), the screen draws
  (VST `tools/rfm1_run/rfm1_run.c`). `--wav out.wav` lets you listen; `--screen out.ppm` saves the screen.
- [ ] The manifest's `version` equals your release tag (`v1.2.3` → `1.2.3`) and `source_url` ends in
  `@<the tag's commit>`, which is pushed. (`rfm1_check` prints the id and version; `rfm1_run --describe` prints the descriptor.)
- [ ] The licence texts are inside (`rfm1_check` counts them).
- [ ] Each capability you added has its test: `--turn … --readouts` (rd1), `--tk --tk-ev …` (tk1), `--ep-info` (ep1),
  `--audio-in in.wav` (ain1), `--folder DIR` (Fantasy CFW) ([CAPABILITIES.md](CAPABILITIES.md)).
- [ ] A skin: `python3 tools/rfm1_skin.py show yourfw.rfm1` lists it.

### The browser host (remotefm1.com)

The site checks the file before running it (the format's section 7 checks, then the instance: SITE
`web/engines/rfm1/engine.js`).

- [ ] Open remotefm1.com and **drop the `.rfm1` on the page** (or open it). It is kept under "Your firmware" in that
  browser (SITE `web/site/js/main.js`, `js/library.js`). A refusal says why ("the browser can't compile it", a missing
  export, a mismatch).
- [ ] It boots to your first screen; keys play; knobs and buttons do what they do on the FM-1.
- [ ] Your skin and labels show (if you have them).
- [ ] Your web editor, if you have one, finds the firmware by its USB port name.
- [ ] Reload the page: your settings and saved data survive (the flash is kept).

### The iOS and Android apps

- [ ] iOS: open the `.rfm1` from Safari's downloads, Files or AirDrop and choose RemoteFM1, or use "Add firmware…", or
  put it in Files › On My iPhone › RemoteFM1 › Firmware (FW `README.md`). Then as for the browser.
- [ ] Android: the RemoteFM1 Android app runs catalog `.rfm1` files; how to add your own file there is not documented
  in its README.

### The Move path

Build the `.rfm1m` and attach it beside the `.rfm1` ([RFM1M.md](RFM1M.md#building-the-move-version-yourself)).

- [ ] Your firmware is fast enough: `rfm1_run` reports how many times faster than real time it runs on your Mac. The
  Move's A72 is about 3.5× slower than the simulator and the budget is about 2600 µs per 128-frame block on average
  (MOVE `README.md`); a firmware that runs only a few times faster than real time in `rfm1_run` on a Mac may not fit.
  Keep `frame()` cheap.
- [ ] The Move build: `tools/catalog_fw.sh` in remoteFM14Move `v0.4.0` (the exact catalog gate: compile, glibc,
  boot, sound, bit-identical PCM) gives `"ok":true` and `<id>-<version>.rfm1m`. Run for these docs on Felucca 1.8.2.
- [ ] Optional: `make sim-run` there runs the host on a Move simulator with its web page at `http://localhost:7730/`.
- [ ] With readouts, labels or a takeover: check them on the simulator's page or a Move after the catalog lists you.

### Before you ask to be listed

- [ ] A GitHub release (not draft, not pre-release) with `<id>-<version>.rfm1` and `<id>-<version>.rfm1m` attached.
- [ ] `RemoteFM1Metadata.json` (optional) validates against its schema in your editor.
- [ ] Open the request form ([CATALOG-POLICY.md](CATALOG-POLICY.md)).
