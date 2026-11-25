# The Move (`.rfm1m`)

Full text: https://github.com/Quixotic7/remotefm1format/blob/main/docs/RFM1M.md

- Authors **never** build or ship `.rfm1m`. The catalog maintainer compiles every catalog `.rfm1` for the Move
  (wasm2c → aarch64 shared object), keeps it only if it sounds bit-identical to the reference run, signs the Move
  catalog's index, and publishes it. The Move runs only signed catalog entries.
- So the Move gets the firmware after it is in the RemoteFM1 catalog.
- What helps on the Move: a cheap `frame()` (the CPU budget is tight: ~2600 µs average per 128-frame block on a
  Cortex-A72; the on-screen firmware gets `frame()` every 33 ms, others every 200 ms), readouts (`rd1`), labels,
  optionally a takeover (`tk1`).
- Optional local test with Docker: Quixotic7/remoteFM14Move at tag `v0.4.0`, `tools/catalog_fw.sh <file>.rfm1 out/`
  (the catalog's exact gate) and `make sim-run` (a Move simulator, page at http://localhost:7730/). Offer it; don't
  require it.
