# RemoteFM1 format documentation

For firmware authors who want their FM-1 firmware to run in RemoteFM1 hosts (remotefm1.com, the iOS and Android apps,
and RemoteFM1 on the Ableton Move).

| Read | For |
| --- | --- |
| [BUILDING.md](BUILDING.md) | how to make an `.rfm1` from your firmware: the two routes, the release workflow, licences |
| [TESTING.md](TESTING.md) | a worked example (Felucca 1.8.2, run for real) and the test checklist for the browser and the Move |
| [RFM1-FORMAT.md](RFM1-FORMAT.md) | the file byte for byte: container, sections, manifest, exports, imports, allowed WebAssembly |
| [HOST-ABI.md](HOST-ABI.md) | the `fm1core` interface a firmware's host build implements |
| [CAPABILITIES.md](CAPABILITIES.md) | the optional extras: skin, labels, readouts, Move takeover, engine parameters, cpu2, audio in and out, Fantasy CFW, screen reader; and which host honours which |
| [METADATA.md](METADATA.md) | `RemoteFM1Metadata.json`: your card, skin, labels and target release |
| [RFM1M.md](RFM1M.md) | the Move's compiled `.rfm1m` (built by the catalog, not by you) |
| [CATALOG-POLICY.md](CATALOG-POLICY.md) | how to get listed, what is checked, updates, removals, hosted-only |
| [SOURCES.md](SOURCES.md) | the repositories and commits every fact comes from, and where code and docs disagree |

Quick path: if your firmware is an **fm1-base app**, run `tools/build.py app --rfm1 --release X.Y.Z`; otherwise check
whether **FM1VST** already has glue for it. Then `rfm1_check.py`, `rfm1_run`, attach the file to a stable release, and
[ask to be listed](CATALOG-POLICY.md).
