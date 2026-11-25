# `.rfm1m`: the Ableton Move's compiled form

**Short version for authors: build the `.rfm1m` from your `.rfm1` and attach it to your release beside it**
([how](#building-the-move-version-yourself): one Docker command, about half a minute). The Move only runs firmwares
from the signed Move catalog, which the maintainer builds from the RemoteFM1 catalog; right now that build still
compiles its own copy from your `.rfm1` and checks that it sounds bit for bit the same. (MOVE `README.md` "The Move
catalog"; MGR `README.md` "The Move catalog (after a publish)")

## Building the Move version yourself

You need Docker, git and make. On an Apple-silicon Mac or an arm64 Linux machine it runs natively; on x86 Docker
emulates arm64 and is much slower. The commands use remoteFM14Move's own build image and the exact check the Move
catalog runs (MOVE `Makefile`, `tools/catalog_fw.sh`):

```sh
git clone --branch v0.4.0 https://github.com/Quixotic7/remoteFM14Move
git clone --branch rfm1-manager https://github.com/Quixotic7/FM1VST      # the wasm3 reference the check compares with
cd remoteFM14Move
F=/path/to/yourfw-1.2.3.rfm1
make image inputs FIRMWARES=$F TEST_FIRMWARES=$F FM1VST=../FM1VST FM1VST_REF=HEAD
docker run --rm -v "$PWD":/w -w /w rfm14move-build make build/tools/rfm1info build/tools/fwrun build/ref/rfm1_run
docker run --rm -v "$PWD":/w -w /w rfm14move-build tools/catalog_fw.sh build/in/yourfw-1.2.3.rfm1 out
cat out/*/result.json
```

`result.json` must say `"ok":true`. The files are in `out/<sha256 of your .rfm1>/files/`: attach
**`yourfw-1.2.3.rfm1m`** to your release, next to `yourfw-1.2.3.rfm1`. (The `.json`, `.licenses.txt` and skin files
there are made from the `.rfm1` and need not be attached.) If `ok` is false, `reason` says why: the build failed, the
object needs a newer glibc than the Move's, it trapped or stayed silent, or its sound differs from the reference.

Run for these docs on Felucca 1.8.2 (FM1VST `e50e77e`, remoteFM14Move `v0.4.0`, Apple silicon): `ok`, the same PCM as
wasm3, needs glibc 2.17, a 2 MB `felucca-1.8.2.rfm1m`, 28 seconds for the check.

## What it is

An `.rfm1m` is a **native Linux aarch64 shared object** made from one `.rfm1`. It has no container or header of its
own. (MOVE `src/rfm1m.h`, `tools/build_fw.sh`)

1. `wasm2c` 1.0.32 translates the `.rfm1`'s WebAssembly to C.
2. That C is compiled with the host's glue (`fw/fwglue.c`): `gcc -O2 -ffp-contract=off -fPIC -shared
   -fvisibility=hidden -DWASM_RT_MEMCHECK_SIGNAL_HANDLER=0 -DRFM1M_MT=1`, in the `rfm14move-build` Docker image
   (Debian bookworm, linux/arm64). The wasm2c runtime is not inside the object: the host provides it.
3. The object exports `rfm1m_get()` (`RFM1M_ABI 1`: the id, the `.rfm1`'s SHA-256, the wasm2c version, and every
   `.rfm1` export as a C call), `rfm1m_caps` (`RFM1M_CAP_MT`), and one table per capability the module has:
   `rfm1m_ain`, `rfm1m_rd1`, `rfm1m_tk1`, `rfm1m_ep1`, `rfm1m_cpu2`. Capabilities are found by the module's exports,
   not by the manifest.
4. It must need glibc 2.35 at most (the Move's), and its output must match wasm3's byte for byte over `rfm1_run`'s
   default run (MOVE `tools/catalog_fw.sh`, `tests/bitident.sh`).

The host loads it with `dlopen`.

## The six files

Every Move firmware is installed as six files (MOVE `tools/build_fw.sh`, `src/storage.c`):

| File | Contents |
| --- | --- |
| `<id>-<version>.rfm1m` | the shared object |
| `<id>-<version>.json` | `{"rfm1m": 1, "file", "sha256" (of the .rfm1), "arch", "wasm2c", "memory_pages", "manifest": <the .rfm1's manifest>}` |
| `<id>-<version>.licenses.txt` | the `rfm1.credits` section, then every `rfm1.license.<n>` |
| `<id>-<version>.rfm1skin` | the `rfm1.skin` section as it is |
| `<id>-<version>.rfm1skin.bg` | `rfm1.skin.background` |
| `<id>-<version>.rfm1skin.bgl` | `rfm1.skin.background.landscape` |

## Who builds it, and how it reaches a Move

- **The maintainer** runs `make catalog` (MOVE `tools/build_move_catalog.py`): it takes every `.rfm1` in
  `https://remotefm1.com/catalog.json`, checks each against the catalog's SHA-256, compiles and checks each with
  `tools/catalog_fw.sh`, leaves out (and reports) any that fail, writes `index.json`, and signs it with the
  maintainer's Ed25519 key. The files go to the GitHub release `move-catalog` of Quixotic7/remoteFM14Move.
- **The Move host** runs only objects listed in a signed index; each file is checked by SHA-256 and size. The
  public key is compiled into the host (MOVE `src/catalog_key.h`). There is no way for a user to side-load an
  `.rfm1m` on the released line, and the Move's web file browser keeps `firmwares/` read-only.
- **On the Move**, the user opens the catalog in the projects view (Mute), and a jog press downloads a firmware into
  `/data/UserData/RemoteFM1/firmwares/`. The index is fetched at start and every 6 hours. A starter set ships with the
  host (MOVE `module/bundle.txt`).
- `index.json` (schema 1) carries per firmware: id, name, version, categories, the description, licence,
  `source_url`, beta, hosted_only, the `.rfm1`'s name and SHA-256, the files, a 64 × 64 1-bit thumbnail, and for
  Fantasy CFWs `virtual`, `needs`, `wants`, `folders`, `memory_mib`.

So the Move gets a firmware **after** it is in the RemoteFM1 catalog, at the maintainer's next Move catalog build.

## What makes a firmware Move-friendly

- **Speed.** The Move's budget is 2902 µs per 128-frame block; a firmware fits while its average stays under about
  2600 µs. The Move's A72 is about 3.5× slower than the simulator; FIZZ found 95 % of its cost was `frame()` drawing
  (MOVE `README.md` "The numbers"). Keep `frame()` cheap.
- **Frames.** The firmware on screen gets `frame()` every 33 ms, the others every 200 ms; `tick()` still runs every ms.
  Read input in `tick()` if it must be prompt.
- **Readouts, labels, takeover, engine parameters, cpu2** make it feel native on the Move ([CAPABILITIES.md](CAPABILITIES.md)).
- **States.** Keep to the default build rules (no exceptions, no table mutation) so states can snapshot the whole
  firmware.

## Testing the Move path without a Move

All in the `rfm14move-build` Docker image from MOVE `docker/Dockerfile` (`make` builds it; needs Docker):

```sh
git clone https://github.com/Quixotic7/remoteFM14Move && cd remoteFM14Move && git checkout v0.4.0
make DEV_FIRMWARES="/path/to/yourfw-1.0.0.rfm1"          # compiles it into build/fw-dev/
build/tools/rfm1info yourfw-1.0.0.rfm1 --check            # the Move's own format check
tools/catalog_fw.sh yourfw-1.0.0.rfm1 out/                # the exact catalog gate: build, glibc, boot, sound, bit-identical PCM
build/tools/fwrun build/fw-dev/yourfw-1.0.0.rfm1m         # run the compiled object offline
make sim-run                                              # the host on a Move simulator, web page at http://localhost:7730/
```

(Commands from MOVE `Makefile`, `tools/`, `README.md` "Test" and "Development builds". The gate needs FM1VST beside
the repository for the wasm3 reference run; it was not run for these docs.) `make package DEV_FIRMWARES=…` makes a
module whose `firmwares-dev/` installs your build at the host's next start; installing a module on a Move is the
Move owner's own business.
