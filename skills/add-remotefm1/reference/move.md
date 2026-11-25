# The Move (`.rfm1m`)

Full text: https://github.com/Quixotic7/remotefm1format/blob/main/docs/RFM1M.md

- The author builds `<id>-<version>.rfm1m` from the `.rfm1` and attaches it to the same release. It is a wasm2c →
  aarch64 shared object, made in remoteFM14Move's build image and kept only if its sound is bit-identical to the wasm3
  reference run. (The Move runs only firmwares from its signed catalog; right now that catalog still compiles its own
  copy from the `.rfm1`.)
- Locally, with Docker (native on Apple silicon or arm64 Linux; emulated and slow on x86), in a scratch folder outside
  the repository:

```sh
git clone --branch v0.4.0 https://github.com/Quixotic7/remoteFM14Move
git clone --branch rfm1-manager https://github.com/Quixotic7/FM1VST      # the wasm3 reference
cd remoteFM14Move
F=/abs/path/to/yourfw-1.2.3.rfm1
make image inputs FIRMWARES=$F TEST_FIRMWARES=$F FM1VST=../FM1VST FM1VST_REF=HEAD
docker run --rm -v "$PWD":/w -w /w rfm14move-build make build/tools/rfm1info build/tools/fwrun build/ref/rfm1_run
docker run --rm -v "$PWD":/w -w /w rfm14move-build tools/catalog_fw.sh build/in/yourfw-1.2.3.rfm1 out
cat out/*/result.json                                  # "ok":true
ls out/*/files/                                        # yourfw-1.2.3.rfm1m (+ .json, .licenses.txt: not needed)
```

  Run for real on Felucca 1.8.2: ok, same PCM as wasm3, glibc 2.17, 2 MB object, 28 s.
- In CI: [workflow-move-job.yml](workflow-move-job.yml), a job for the release workflow on `ubuntu-24.04-arm`.
- What helps on the Move: a cheap `frame()` (the CPU budget is tight: ~2600 µs average per 128-frame block on a
  Cortex-A72; the on-screen firmware gets `frame()` every 33 ms, others every 200 ms), readouts (`rd1`), labels,
  optionally a takeover (`tk1`).
- `make sim-run` in that checkout runs the Move host on a simulator, page at http://localhost:7730/.
