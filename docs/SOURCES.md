# Sources and how the docs cite them

Every fact in these docs comes from code or docs in the public repositories below, read at the commit given. A
citation looks like **(VST `tools/rfm1_check.py`)**: the short name of the repository, then the path at that commit.

| Short name | Repository | Read at | What it is |
| --- | --- | --- | --- |
| VST | [Quixotic7/FM1VST](https://github.com/Quixotic7/FM1VST/tree/e50e77ec41dedd8d8c041186a7d01978c80570e2) | `e50e77e` (branch `rfm1-manager`, 2026-11-23) | the emulator, the plugin, the core glue and the whole `.rfm1` toolchain (`cmake/FM1Rfm1.cmake`, `tools/rfm1_*.py`, `tools/rfm1_run/`, `core-api/`) |
| IOS | [Quixotic7/RemoteFM1IOS](https://github.com/Quixotic7/RemoteFM1IOS/tree/3c261bef58fd0987391c9341a504752a32b9a2b8) | `3c261be` (`main`) | the iOS app; `docs/RFM1-FORMAT.md` is the original format text (VST has a copy of it at an older commit) |
| FW | [Quixotic7/RemoteFM1FW](https://github.com/Quixotic7/RemoteFM1FW/tree/345ee0266f10330905faccc4806c69bf5ac7b55d) | `345ee02` (`main`) | the catalog: `catalog/catalog.json`, `catalog/SCHEMA.md`, `docs/AUTHOR-RFM1.md`, `docs/REMOTEFM1-METADATA.md`, the nightly author sync |
| MGR | [Quixotic7/RFM1Manager](https://github.com/Quixotic7/RFM1Manager/tree/6c18885d1e59dfcd45ae8dd0685c88e392c7b3fb) | `6c18885` (`master`) | the maintainer's tool that adds, builds, checks and publishes catalog firmwares |
| MOVE | [Quixotic7/remoteFM14Move](https://github.com/Quixotic7/remoteFM14Move/tree/v0.4.0) | tag `v0.4.0` (`517c952`) | RemoteFM1 on the Ableton Move: the `.rfm1m` compiler, the Move catalog, the host side of every capability |
| CS | [Quixotic7/rfm1cs-abletonmove](https://github.com/Quixotic7/rfm1cs-abletonmove/tree/v0.14.0) | tag `v0.14.0` (`1004f14`) | the Move controller core (knob pages, readouts, takeover gestures) |
| SITE | [Quixotic7/RemoteFM1](https://github.com/Quixotic7/RemoteFM1/tree/7c97be5b99e478287a4b7ca6026326eae8d2ea68) | `7c97be5` (`main`) | remotefm1.com: the browser host, `docs/PROTOCOL.md` |
| BASE | [charlesvestal/fm1-base](https://github.com/charlesvestal/fm1-base) | `main` (`c8e0599` is the commit EVILGENIUS pins) | the fm1-base firmware framework; `tools/build.py --rfm1` builds an `.rfm1` without FM1VST |
| EG | [Quixotic7/evilgenius](https://github.com/Quixotic7/evilgenius) | `main` | a firmware whose `.github/workflows/release.yml` builds and attaches its `.rfm1` on GitHub |

Links in the docs go to these commits, so they keep pointing at what was read.

## Important: where the toolchain lives

FM1VST's `.rfm1` toolchain is **not on its `main` branch and not in any FM1VST release yet** (the newest release,
v0.6.2, has no `cmake/FM1Rfm1.cmake` and no `tools/rfm1_*`). It is on the pushed branch `rfm1-manager`. Pin a commit
of that branch (the docs use `e50e77e`) until a release carries it. VST `docs/RFM1.md` shows a workflow pinned to
`v0.7.0`, a tag that does not exist yet.

## Where code and docs disagree (the code wins)

| Topic | The docs say | The code does | Where |
| --- | --- | --- | --- |
| `AUDIO_OUT` channel counts | 4, 6, 8 or 10 | `fm1_add_rfm1` also accepts 2 (stereo only: no aout1 exports are added) | VST `docs/RFM1.md` vs `cmake/FM1Rfm1.cmake` |
| `fm1_add_rfm1` options | the table has no `METADATA` or `READOUT_TEST` | both exist (`METADATA <file>` overrides the repository's `RemoteFM1Metadata.json`) | VST `docs/RFM1.md` vs `cmake/FM1Rfm1.cmake` |
| `glue_url` in the manifest | IOS `docs/RFM1-FORMAT.md` §4 lists it (optional) | `tools/rfm1_pack.py` never writes it; `built_by` names the FM1VST commit instead | VST `tools/rfm1_pack.py` |
| WebAssembly features | IOS §6 allows multi-value | `rfm1_check.py` accepts it, but the build's `-mcpu=mvp …` flags don't turn it on | VST `cmake/FM1Rfm1.cmake`, `tools/rfm1_check.py` |
| ep1 operations | ops 2-4 (LOCK, FREE, FREE_ALL) answer −3 "until a later version" | the shim implements them (−2 for a NOLOCK parameter, −4 when holds run out; −3 only for an unknown op). The Move host 0.4.0 uses only SET | VST `docs/RFM1-FORMAT-ep1.md` vs `core-glue/common/rfm1_shim.c` |
| `ain1` in `needs` | MOVE README: "in wants or needs" | the Move's `rfm1info --check` refuses any `needs` other than `fs1`; use `wants` | MOVE `tools/rfm1info.c` |
| cpu2 imports | MOVE README: published files must use the shim form | MOVE `rfm1info` also accepts the `rfm1_cpu2_1` import form; VST `rfm1_check.py` refuses any import outside WASI (and `rfm1_host_1` for Fantasy CFWs) | MOVE `tools/rfm1info.c`, VST `tools/rfm1_check.py` |
| catalog categories | FW `catalog/SCHEMA.md`: "Synth \| Groovebox \| Drums \| Sampler \| Other ..." | the metadata schema, the Move index and the catalog in use have exactly Synth, Groovebox, Chords, Drums, Other | FW `docs/RemoteFM1Metadata.schema.json`, `catalog/catalog.json` |
| the Move README version | "version 0.1.0" in places | the release is 0.4.0 (CHANGES.md) | MOVE `README.md` vs `CHANGES.md` |
| the RemoteFM1FW nightly sync | — | pins FM1VST `5634390` (2026-11-14), whose `rfm1_check.py` predates the rd1 / tk1 / ep1 / cpu2 rules | FW `.github/workflows/author-rfm1-sync.yml` |
