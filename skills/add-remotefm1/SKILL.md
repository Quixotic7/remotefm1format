---
name: add-remotefm1
description: Add RemoteFM1 support to an M-VAVE FM-1 firmware repository - build its .rfm1 (the WebAssembly host build that remotefm1.com, the RemoteFM1 apps and the Ableton Move run), write RemoteFM1Metadata.json, add a release workflow that attaches the .rfm1, optionally add capabilities (skin, labels, readouts rd1, Move takeover tk1, engine parameters ep1, cpu2, audio in), and verify with rfm1_check and rfm1_run. Use when an FM-1 firmware author asks to "add RemoteFM1 support", "build an .rfm1", "make my firmware run on remotefm1.com / the Move", or "get listed in the RemoteFM1 catalog".
---

# Add RemoteFM1 (`.rfm1`) support to an FM-1 firmware

You are working inside a **firmware author's own repository**. The goal: their stable GitHub releases carry a
`<id>-<version>.rfm1` that passes the public checks, so it can be listed in the RemoteFM1 catalog. The Move's
`.rfm1m` is **never** built here: the catalog maintainer compiles it from the `.rfm1` ([reference/move.md](reference/move.md)).

The format and tooling are documented at https://github.com/Quixotic7/remotefm1format (`docs/`). The reference files
next to this skill summarise what you need; fetch the docs for detail.

## Ground rules

- **Ask before adding any optional capability.** Step 5 lists them; offer them, explain each in one sentence, and add
  only what the author says yes to, one at a time, testing each.
- **Don't change how the FM-1 device image is built.** RemoteFM1 code goes behind the defines the toolchain sets
  (`FM1CORE_RFM1`, `FM1_HOST_EXT`, `FM1_HOST_TK`, `FM1_HOST_EP1`) or into files only the host build compiles. If the
  repository has a device build you can run, confirm its output is byte-identical before and after.
- **Never push, tag, create releases, or open issues or pull requests** (here or upstream) without the author's
  explicit yes for that action. Commit locally only if the author wants commits.
- **Never bundle third-party preset dumps or raw `.syx` / `.mid` / `.wav` files** in a release asset. Presets compiled
  into the firmware are fine if their licence allows.
- **Pin tools to commits.** FM1VST's `.rfm1` toolchain is on its branch `rfm1-manager`, not in a release yet: use
  commit `e50e77ec41dedd8d8c041186a7d01978c80570e2` or a later commit of that branch the author picks. FM1VST's own
  docs mention a `v0.7.0` tag; it does not exist.
- If something here disagrees with what you find in the tools' own code, trust the code and tell the author.

## Step 1: Look at the repository

Find out, and tell the author in a few lines:

1. **Build system and family** (see [reference/routes.md](reference/routes.md)):
   - `fm1.toml` + `app.h` / `app.c` → an **fm1-base app** → route A.
   - Felucca or a Felucca fork (`firmware/src/felucca.c`, `web/emu/`), Melodee, SLOOP, Dinghy / GR!MM, ChoralRoot, or
     anything else → route B (FM1VST glue).
2. **Is it already in FM1VST?** Check https://github.com/Quixotic7/FM1VST/blob/e50e77ec41dedd8d8c041186a7d01978c80570e2/.gitmodules
   for this repository's URL, and `core-glue/*/core_<id>.cmake` for its `fm1_add_core` / `fm1_add_rfm1` lines. Note
   the core id and the glue folder.
3. **Licence** (LICENSE / COPYING / LICENSES/ / LICENSING.md) and SPDX id. No licence → stop and tell the author the
   catalog cannot list it until there is one.
4. **Version source:** where the version string lives and how release tags are named (`v1.2.3`?). The `.rfm1`'s
   version must equal the tag's.
5. **Existing CI**, existing `RemoteFM1Metadata.json`, existing `.rfm1` release assets.
6. **Third-party material** (samples, fonts, images, presets): list it for the author.

## Step 2: Check the toolchain

- Emscripten **6.0.13** (`emcc --version`; `emnm` and `wasm-opt` must be on PATH from the same emsdk). Other 6.0.x
  versions worked in testing; say which one you used.
- Python 3; Pillow (and fontTools for Felucca-family generators); CMake ≥ 3.22 for route B.
- If something is missing, tell the author the exact install commands; don't install system-wide without asking.

## Step 3: Build the `.rfm1` locally

Follow the route in [reference/routes.md](reference/routes.md). Work in a scratch directory outside the repository
(for example `../rfm1-work/`) for FM1VST checkouts and build trees; never commit them.

- **Route A (fm1-base):** `python3 <fm1-base>/tools/build.py <app dir> --rfm1 --release <X.Y.Z>`. Commit or stash
  nothing yourself: if the tree is dirty, the tool warns that `source_url` won't match; tell the author.
- **Route B (FM1VST):** clone FM1VST at the pinned commit, put this repository at the submodule path its glue expects
  (`cores/<Name>`) at the current commit, configure with `-DFM1_RFM1_ONLY=ON -DFM1_PUBLIC_BUILD=ON
  -DFM1_ONLY_CORES=<id>`, build `rfm1_<id>` and `rfm1_run`.
  - If FM1VST has glue but no `fm1_add_rfm1` line, add `-DFM1_RFM1_EXTRA_CORES=<id>` to try it.
  - **If FM1VST has no glue for this firmware**, stop and explain ([reference/routes.md](reference/routes.md#no-glue-yet)):
    the catalog maintainer builds it from their releases; they file the catalog request and choose "Please build it
    for me". Still offer `RemoteFM1Metadata.json` (step 5). Don't write FM1VST glue.

## Step 4: Verify

Run both, and show the author the output lines:

```sh
python3 <fm1vst>/tools/rfm1_check.py <file>.rfm1        # must say "ok"
<build>/rfm1_run <file>.rfm1                             # must exit 0: boots, chord sounds, screen draws
```

Then check the manifest: `id`, `version` (= the tag's), `license`, `source_url` = `https://github.com/<owner>/<repo>@<commit>`
with the commit that was built, at least one licence text. Details and failure fixes: [reference/checks.md](reference/checks.md).

## Step 5: Offer the optional extras (ask first)

Present this list and ask which, if any, the author wants. Add one at a time, rebuild, re-run step 4 plus its own
test. Full details: [reference/capabilities.md](reference/capabilities.md).

| Extra | One-line pitch | Effort |
| --- | --- | --- |
| Card (`RemoteFM1Metadata.json` `card`) | the catalog description, category, picture, links, USB port name | a JSON file |
| Skin | the firmware's own background and colours on the on-screen FM-1 | a JSON file + images |
| Fixed labels | name the buttons and knobs for what they do | a JSON file (route B) |
| Live labels | labels that change per page or mode | two C functions |
| Readouts `rd1` | the Move shows the real value while a knob turns ("CUTOFF 412 Hz") | often free from a parameter map; else a few macro calls |
| Move takeover `tk1` | the firmware's own Move screen, pads and lights | substantial: its own UI code |
| Engine parameters `ep1` | an ENGINE knob page on the Move | substantial; early, ask the RemoteFM1 maintainer first |
| `cpu2` | a dual-core firmware's core-1 job on its own thread on the Move | only for firmwares that already use core 1 |
| Audio input `ain1` | live audio into a sampler or effect | a hook in the host back end |
| Fantasy CFW | a RemoteFM1-only firmware with big memory and a shared sample folder | a different kind of firmware |

Write `RemoteFM1Metadata.json` from [reference/metadata-template.json](reference/metadata-template.json): keep only the
parts the author wants, and keep the `$schema` line so their editor checks it.

## Step 6: The release workflow

Add `.github/workflows/rfm1.yml` from [reference/workflow-fm1base.yml](reference/workflow-fm1base.yml) (route A) or
[reference/workflow-fm1vst.yml](reference/workflow-fm1vst.yml) (route B). Fill in the id, paths and submodule
folder. It runs on a published, non-pre-release GitHub release, builds the `.rfm1` from the tagged commit, checks it,
and attaches `<id>-<version>.rfm1`. If the repository already has a release workflow, offer to add a job to it
instead. Validate the YAML (for example `python3 -c 'import yaml,sys; yaml.safe_load(open(sys.argv[1]))' file`) and
tell the author it will first run at their next release; nothing is pushed by you.

## Step 7: Hand over

Tell the author, briefly:

- what you added (files, workflow, capabilities) and the `rfm1_check` / `rfm1_run` results;
- that the Move build happens in the catalog, not here;
- next steps: make a stable release (not a pre-release) so the workflow attaches the file, try it on
  https://remotefm1.com by dropping the file on the page, then ask to be listed with the form
  https://github.com/Quixotic7/RemoteFM1FW/issues/new?template=add-firmware.yml (policy:
  https://github.com/Quixotic7/remotefm1format/blob/main/docs/CATALOG-POLICY.md). Once listed, each newer stable
  release with an attached `.rfm1` is picked up automatically every night.
