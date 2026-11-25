# Build routes

Full text: https://github.com/Quixotic7/remotefm1format/blob/main/docs/BUILDING.md

## Route A: fm1-base apps

Signs: `fm1.toml` next to `app.h` / `app.c` (often under `firmware/app/`), fm1-base as a submodule or a sibling.

```sh
git clone --recursive https://github.com/charlesvestal/fm1-base ../rfm1-work/fm1-base     # or the repo's own submodule
git -C ../rfm1-work/fm1-base checkout c8e05997ce970875038156d2bbd69e712eac1543           # a tested commit; newer is fine
python3 ../rfm1-work/fm1-base/tools/build.py <app dir> --rfm1 --release X.Y.Z
# -> <app dir>/build/<name>-X.Y.Z.rfm1     (without --release: <app dir>/build/rfm1/<id>-dev+<commit>.rfm1)
```

The manifest comes from: `id` = `fm1.toml` `name` (lower case, `[a-z0-9_]`, ≤ 31), `name` = `#define APP_NAME` in the
app's headers, `version` = `--release`, `source_url` = `origin` remote (or `fm1.toml` `repo`) `@HEAD`, `license` =
`fm1.toml` `licence` (default `MPL-2.0`, so **set it** if the app's licence differs), `authors` = `fm1.toml` `authors` /
`author`, else the first commit's author. `rfm1_memory_mib` (1..32, default 8) sets memory. Credits: the app's
`CREDITS.md`.

fm1-base does not read `RemoteFM1Metadata.json`; add a skin after with
`python3 -I <fm1vst>/tools/rfm1_skin.py add in.rfm1 -o out.rfm1 --metadata RemoteFM1Metadata.json`. fm1-base's `host/rfm1.c`
exports only the base set: no labels, readouts, takeover or other capabilities on this route (they need route B).

For `rfm1_check.py` / `rfm1_skin.py` you need only FM1VST's `tools/` (a sparse checkout is enough). For `rfm1_run`
build route B's target, or skip it locally and rely on the catalog's run.

## Route B: FM1VST glue

```sh
git clone https://github.com/Quixotic7/FM1VST ../rfm1-work/fm1vst
cd ../rfm1-work/fm1vst
git checkout e50e77ec41dedd8d8c041186a7d01978c80570e2
# put the author's repository where the glue expects it, at the commit to build:
#   find the path: git config -f .gitmodules --get-regexp 'submodule\..*\.url' | grep -i <repo name>
#   then either: git submodule update --init --recursive cores/<Name>      (the pinned upstream commit)
#   or replace cores/<Name> with a clone of the author's working tree at its HEAD (what CI does)
source <emsdk>/emsdk_env.sh
cmake -S . -B build-rfm1 -DFM1_RFM1_ONLY=ON -DFM1_PUBLIC_BUILD=ON -DFM1_ONLY_CORES=<id>
cmake --build build-rfm1 --target rfm1_<id> rfm1_run -j8
ctest --test-dir build-rfm1 -R "^rfm1_(rd_)?<id>$" --output-on-failure
# -> build-rfm1/rfm1/<id>-<version>.rfm1
```

Tested for real with Felucca 1.8.2 (id `felucca`, folder `cores/Felucca`): 1126132 bytes, `rfm1_check` ok, `rfm1_run`
35x real time. Use a full (not shallow) submodule fetch. `ctest -R rfm1_` alone also runs the takeover demo's tests,
which need `--target rfm1_all`.

The version in the file comes from the glue's `VERSION_FILE` / `VERSION_REGEX` (read from the firmware's source). A
glue with a fixed `VERSION` gives the same version for every release; tell the author (it needs an FM1VST change).
`fm1_add_rfm1` reads `RemoteFM1Metadata.json` from the repository root inside `cores/<Name>`.

Notable `fm1_add_rfm1` options: `AUTHORS`, `MEMORY_MIB` (≤ 32), `LICENSE_FILES`, `WANTS rd1 tk1 ep1 cpu2 ain1`,
`TAKEOVER_NAME` / `TAKEOVER_CLAIMS`, `AUDIO_OUT`, `VIRTUAL NEEDS fs1 FOLDERS …`, `NO_SKIN`, `NO_LABELS`, `METADATA`.
The line lives in FM1VST's glue file, not in the author's repository; to change it the author needs FM1VST changed
(a fork in CI, or a request upstream).

## No glue yet

FM1VST has no glue for the firmware when its repository is not in FM1VST's `.gitmodules`. Options, in order of effort
for the author:

1. **Let the catalog build it.** The maintainer's RFM1Manager detects the firmware's family and scaffolds glue from
   the closest one. The author just files the catalog request (choose "Please build it for me").
2. **Port to fm1-base** if the firmware is small or already structured like an fm1-base app (route A, no glue).
3. **Write glue** in a fork of FM1VST: `core-glue/<family>/core_<id>.cmake`, `core_<id>.c`, `core_<id>_desc.c`
   (start from the closest family; `core-glue/common/README.md` "Adding a core" lists the steps), and offer it
   upstream. The route for outside glue contributions is not documented yet; say so and suggest asking in the catalog
   request. Ask the author before starting this: it is real work.
