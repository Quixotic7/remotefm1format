# Building an `.rfm1`

There are two working routes today. Which one you use depends on how your firmware is built.

| Your firmware is… | Route | What you add to your repository |
| --- | --- | --- |
| an **fm1-base app** (`app.h`, `app.c`, `fm1.toml`) | [A. fm1-base `tools/build.py --rfm1`](#a-fm1-base-apps) | nothing but a release workflow (and optionally `RemoteFM1Metadata.json` for a skin) |
| anything else (Felucca and its forks, Dinghy, your own HAL…) | [B. FM1VST `fm1_add_rfm1`](#b-fm1vst-fm1_add_rfm1) | a release workflow; your firmware needs FM1VST glue (it may already have it) |

Then build the Move version (`.rfm1m`) from the `.rfm1` ([RFM1M.md](RFM1M.md#building-the-move-version-yourself)) and
attach both to your release.

A firmware that only has FM-1 releases (`.fwsc`) can also be built by the catalog maintainer, whose RFM1Manager
detects the firmware's family, scaffolds glue from the closest one and builds its stable releases (MGR `README.md`
§4); anyone can ask for that ([CATALOG-POLICY.md](CATALOG-POLICY.md)).

Either way the result must pass FM1VST's `tools/rfm1_check.py` and boot, sound and draw in `rfm1_run`
([TESTING.md](TESTING.md)).

## Toolchain

- **Emscripten 6.0.13** (`emcc`, with `emnm` and Binaryen's `wasm-opt` from its tree). The FM1VST build refuses to
  configure without all three (VST `docs/RFM1.md`, `cmake/FM1Rfm1.cmake`). Other versions are untested.
- **Python 3** (standard library for the tools; Pillow and fontTools for some firmwares' generate steps; Pillow with
  Raqm for fm1-base's fonts: EG `.github/workflows/release.yml`).
- **CMake 3.22+** for route B.
- A C compiler for the host, to build `rfm1_run` (route B builds it for you).

## The fixed build rules

Both routes compile with these (VST `cmake/FM1Rfm1.cmake`):

```
-mcpu=mvp -msign-ext -mnontrapping-fptoint -mbulk-memory -mmutable-globals     # the allowed WebAssembly
-O3 -std=gnu11 -ffp-contract=off -fno-common -DFM1CORE_RFM1=1                  # same sound on every host
link: --no-entry -sSTANDALONE_WASM -sFILESYSTEM=0 -sINITIAL_MEMORY=<n> -sALLOW_MEMORY_GROWTH=0 -sEXPORTED_FUNCTIONS=[] -Wl,--gc-sections
```

then `tools/rfm1_pack.py` adds the manifest, credits, licences and skin ([RFM1-FORMAT.md](RFM1-FORMAT.md#2-the-container)).
Don't hand-roll this: use one of the routes.

---

## A. fm1-base apps

fm1-base (Charles Vestal's firmware framework, MPL-2.0) builds an `.rfm1` from an app directly, with its own host
layer (`host/rfm1.c`) and pack step (BASE `tools/build.py`):

```sh
python3 fm1-base/tools/build.py path/to/app --rfm1                 # dev build: app/build/rfm1/<id>-dev+<commit>.rfm1
python3 fm1-base/tools/build.py path/to/app --rfm1 --release 1.2.3 # release:  app/build/<name>-1.2.3.rfm1
```

- The manifest comes from `fm1.toml`, `app.h` and git: `id` is the app's `name` (lower case, `[a-z0-9_]`), `name`
  is `APP_NAME` from the app's header, `version` is `--release` (else `dev+<commit>`), `source_url` is the repository's
  `origin` URL (or `fm1.toml`'s `repo`) `@` the HEAD commit, `license` is `fm1.toml`'s `licence` (default
  `MPL-2.0`), `authors` from `fm1.toml` `authors` / `author` (else the repository's first commit's author).
- `fm1.toml`'s `rfm1_memory_mib` sets the fixed memory (1..32, default 8).
- Credits: the app's `CREDITS.md`, then fm1-base's. Licence texts: the app's, fm1-base's, felucca-bsp's and
  Emscripten's runtime.
- Commit everything first: with uncommitted changes the tool warns that `source_url` names a commit that is not what
  was built.
- fm1-base does not read `RemoteFM1Metadata.json`. Add a skin afterwards with FM1VST's tool:
  `python3 -I fm1vst/tools/rfm1_skin.py add in.rfm1 -o out.rfm1 --metadata RemoteFM1Metadata.json`.
- fm1-base's host layer exports only the base set ([RFM1-FORMAT.md](RFM1-FORMAT.md#4-exports)): no live labels,
  readouts, takeover or other capabilities (BASE `host/rfm1.c`). For those, the firmware needs FM1VST glue (route B;
  `core-glue/fm1base/` covers fm1-base apps).
- fm1-base firmwares speak the screen reader protocol through `announce.c` (`fm1.toml` `words`).
- fm1-base upper-cases a `-suffix` in the version (EG workflow comment); stable releases have none anyway.

### Release workflow (route A)

This is the `rfm1` job of a real, working release workflow (EG `.github/workflows/release.yml`), trimmed to the
essentials. Pin both tools to commits; replace `yourfw` and `firmware/app`. This trimmed version has not been run
itself; the full job it comes from runs on every EVILGENIUS release.

```yaml
name: rfm1
on:
  release:
    types: [published]
permissions:
  contents: write
jobs:
  rfm1:
    if: ${{ !github.event.release.prerelease }}
    runs-on: ubuntu-24.04
    env:
      FM1_BASE_REF: c8e05997ce970875038156d2bbd69e712eac1543   # charlesvestal/fm1-base, a tested commit
      FM1VST_REF: e50e77ec41dedd8d8c041186a7d01978c80570e2     # Quixotic7/FM1VST rfm1-manager: rfm1_check / rfm1_skin
      EMSDK_VERSION: 6.0.13
    steps:
      - uses: actions/checkout@v4
      - uses: actions/checkout@v4
        with: { repository: charlesvestal/fm1-base, ref: "${{ env.FM1_BASE_REF }}", path: fm1-base, submodules: recursive }
      - uses: actions/checkout@v4
        with: { repository: Quixotic7/FM1VST, ref: "${{ env.FM1VST_REF }}", path: fm1vst, sparse-checkout: tools }
      - uses: mymindstorm/setup-emsdk@v14
        with: { version: "${{ env.EMSDK_VERSION }}" }
      - run: pip install Pillow
      - name: Build, skin, check
        env: { TAG: "${{ github.event.release.tag_name }}" }
        run: |
          set -euo pipefail
          V="${TAG#v}"
          python3 fm1-base/tools/build.py firmware/app --rfm1 --release "$V"
          mkdir out
          if [ -f RemoteFM1Metadata.json ]; then
            python3 -I fm1vst/tools/rfm1_skin.py add "firmware/app/build/yourfw-$V.rfm1" -o "out/yourfw-$V.rfm1" --metadata RemoteFM1Metadata.json
          else
            cp "firmware/app/build/yourfw-$V.rfm1" out/
          fi
          python3 -I fm1vst/tools/rfm1_check.py "out/yourfw-$V.rfm1"
      - name: Attach to the release
        env: { GH_TOKEN: "${{ github.token }}" }
        run: gh release upload "${{ github.event.release.tag_name }}" out/*.rfm1 --clobber --repo "${{ github.repository }}"
```

EVILGENIUS's full job also installs Pillow with Raqm (`libraqm0`), pins emsdk by commit, and fails the release if
the manifest's version is not the tag's. Adding `rfm1_run` (it needs FM1VST's `tools/rfm1_run` and wasm3 built:
[TESTING.md](TESTING.md)) makes the CI test the same as the catalog's.

---

## B. FM1VST `fm1_add_rfm1`

FM1VST builds the same firmware two ways from one glue: the native plugin core and the `.rfm1`. You need your
firmware to be **a core in FM1VST**: a submodule under `cores/<Name>` and a glue folder under `core-glue/` with an
`fm1_add_core` call (VST `docs/RFM1.md` "What you need"; `core-glue/common/README.md` "Adding a core").

**Is your firmware already there?** Many are: the glue families cover Felucca and about twenty forks
(`core-glue/felucca_web/`), Melodee, SLOOP and its family, Dinghy and GR!MM, ChoralRoot, fm1-base apps
(`core-glue/fm1base/`) and more. Look in VST `core-glue/` and `.gitmodules` for your repository. If it is, skip to
"Build one".

**If not, the glue is:** (VST `core-glue/common/README.md`)

1. your firmware as a pinned, unmodified submodule `cores/<Name>`;
2. `core-glue/<family>/core_<id>.cmake` calling `fm1_add_core(<id> … SOURCE … UNIT core_<id>.c DESC core_<id>_desc.c
   FAMILY … LICENSE … GEN …|NO_GEN VERSION_FILE …/VERSION_REGEX … )`;
3. `core_<id>.c`: includes your firmware's sources plus a host HAL (time, LEDs, LCD, flash on `common/nor.c`, audio
   re-blocked to 128 frames by `common/fifo.c`, MIDI) and defines the `fm1core_<p>_*` entry points;
4. `core_<id>_desc.c`: the descriptor (id, version, `source_url`, family, licence, credits, the 14 button names);
5. a conformance file and a ctest.

If FM1VST has no glue for your firmware, the simplest route is to ask to be listed and choose "Please build it
for me": the maintainer adds the glue with RFM1Manager ([CATALOG-POLICY.md](CATALOG-POLICY.md)).

### Build one

In your core's glue file, after `fm1_add_core`, one line (it does nothing unless `-DFM1_RFM1=ON`):

```cmake
fm1_add_rfm1(<id> AUTHORS "Your Name")
```

```sh
git clone https://github.com/Quixotic7/FM1VST && cd FM1VST
git checkout e50e77ec41dedd8d8c041186a7d01978c80570e2       # rfm1-manager: the .rfm1 toolchain (not in a release yet)
git submodule update --init cores/<Name>
cmake -S . -B build-rfm1 -DFM1_RFM1_ONLY=ON -DFM1_PUBLIC_BUILD=ON -DFM1_ONLY_CORES=<id>
cmake --build build-rfm1 --target rfm1_<id> rfm1_run
ctest --test-dir build-rfm1 -R "^rfm1_<id>$" --output-on-failure
# -> build-rfm1/rfm1/<id>-<version>.rfm1
```

(VST `docs/RFM1.md` "Build one"). `-DFM1_RFM1_EXTRA_CORES=<id>` tries a core whose glue has no `fm1_add_rfm1` line
yet. The build reads `RemoteFM1Metadata.json` from your repository's root in the submodule ([METADATA.md](METADATA.md)).

### `fm1_add_rfm1` options

(VST `cmake/FM1Rfm1.cmake`, `docs/RFM1.md`)

| Option | What |
| --- | --- |
| `AUTHORS <names…>` | the manifest's authors |
| `MEMORY_MIB <n>` | fixed memory, default 8, at most 32 (512 with `VIRTUAL`) |
| `STACK_KIB <n>` | C stack inside that memory, default 1024 |
| `LICENSE_FILES <name=path…>` | licence texts to carry; default your `LICENSE` / `COPYING`, `LICENSES/*`, `LICENSING.md` |
| `SOURCE_URL <url>` | instead of the descriptor's |
| `VERSION <v>` | replaces the version for the Wasm build only |
| `DEFINES`, `FLAGS`, `INCLUDES` | more, for the Wasm build only |
| `RUN_ARGS <args…>` | for the ctest's `rfm1_run` (`--sysex bank.syx`, `--settle 5000`) |
| `WANTS <cap…>` | `rd1`, `tk1`, `ep1`, `cpu2`, `ain1` ([CAPABILITIES.md](CAPABILITIES.md)) |
| `READOUT_TEST <…>` | turns and expected readouts for an `rfm1_rd_<id>` test |
| `TAKEOVER_NAME`, `TAKEOVER_CLAIMS` | with `WANTS tk1`, the manifest's `takeover` |
| `AUDIO_OUT <n>` | more output channels (aout1) |
| `VIRTUAL`, `NEEDS <cap…>`, `FOLDERS <name=r\|rw…>` | a Fantasy CFW |
| `METADATA <file>` | another `RemoteFM1Metadata.json` than the repository root's |
| `NO_SKIN`, `NO_LABELS` | leave the metadata's skin / labels out |
| `HOSTED_DESPITE_GATE "<reason>"` | the maintainer's switch for a gated core ([CATALOG-POLICY.md](CATALOG-POLICY.md#hosted-only-and-gated)) |

### Release workflow (route B)

VST `docs/RFM1.md` "Attach it to your releases" has a template that checks out FM1VST, puts your repository at
`fm1vst/cores/<Name>`, builds and tests only your core, and uploads the file. Two corrections to it: the template pins
FM1VST `v0.7.0`, which **does not exist**; pin `e50e77ec41dedd8d8c041186a7d01978c80570e2` (or a later commit of
`rfm1-manager`) instead. And the doc itself says the template "has not run on GitHub yet". The skill in this
repository writes this workflow for you ([../skills/add-remotefm1/](../skills/add-remotefm1/SKILL.md)).

---

## What does not work in WebAssembly

(VST `docs/RFM1.md`)

- One thread, no file system, no SIMD, threads, exceptions or reference types.
- A main loop that never returns is fine: it runs on a coroutine (Asyncify), nothing to add.
- Lua's `setjmp` / `longjmp` errors can run on the same pass (`core-glue/common/lj_asyncify.h`).
- Native-only headers can get stand-ins in `core-glue/common/wasm_compat/`.
- Use `#ifdef FM1CORE_RFM1` (or `__EMSCRIPTEN__`) only when you must tell.

## Licences

An `.rfm1` is a binary of your firmware; your licence's rules for binaries apply (VST `docs/RFM1.md` "Licences").

- `source_url` must be your repository **at the exact commit** the file was built from, and that commit must be
  public. Hosts show it, so users can get the source.
- The file carries the licence texts: yours, the glue's (FM1VST's glue is GPL-3.0-only; fm1-base is MPL-2.0), and the
  Emscripten runtime's (MIT). A GPL firmware's `.rfm1` built with FM1VST includes GPL glue; `built_by` names its
  FM1VST commit.
