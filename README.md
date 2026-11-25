# remotefm1format

Everything a firmware author needs to put their M-VAVE FM-1 firmware into
[RemoteFM1](https://remotefm1.com): the `.rfm1` file format, a Claude Code skill that adds RemoteFM1 support to a
firmware repository, an explainer website, and the official way to ask for a firmware to be listed in the RemoteFM1
catalog.

**RemoteFM1** runs firmwares written for the FM-1 inside host apps: remotefm1.com in a browser, RemoteFM1 for iOS
(and its AUv3), RemoteFM1 for Android, and RemoteFM1 on the Ableton Move. A firmware reaches them as one
**`.rfm1`** file: the firmware's host build compiled to WebAssembly, with a manifest that names its version, licence
and exact source. The Move runs a compiled form, `.rfm1m`, which the catalog makes from the `.rfm1`.

## What is here

| | |
| --- | --- |
| [`docs/`](docs/README.md) | **The format documentation.** The file byte for byte ([RFM1-FORMAT.md](docs/RFM1-FORMAT.md)), the host ABI ([HOST-ABI.md](docs/HOST-ABI.md)), every optional capability and which host honours it ([CAPABILITIES.md](docs/CAPABILITIES.md)), `RemoteFM1Metadata.json` ([METADATA.md](docs/METADATA.md)), the Move's `.rfm1m` ([RFM1M.md](docs/RFM1M.md)), how to build ([BUILDING.md](docs/BUILDING.md)) and test ([TESTING.md](docs/TESTING.md), with a worked example run for real). Every fact cites its source repository and commit ([SOURCES.md](docs/SOURCES.md)). |
| [`skills/add-remotefm1/`](skills/add-remotefm1/SKILL.md) | **A Claude Code skill.** Run it inside your firmware's repository: it finds your build route, builds and checks the `.rfm1`, writes `RemoteFM1Metadata.json`, adds a release workflow, and offers the optional extras one at a time (it asks before adding any). |
| [`site/`](site/index.html) | **The explainer website**, for people who are slightly technical but not developers: what RemoteFM1 and an RFM1 build are, why make one, what players get, the steps, the extras, and how to get listed. Static HTML and CSS. |
| [`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE/) and [`docs/CATALOG-POLICY.md`](docs/CATALOG-POLICY.md) | **The catalog request route.** An issue form that collects what the catalog needs, a form for changes and removals, and the policy: what is accepted, what is checked, updates, removals, hosted-only. |

## For firmware authors

1. Read [BUILDING.md](docs/BUILDING.md) (or the website) to see which route fits your firmware.
2. Build, check with `rfm1_check.py` and `rfm1_run`, and attach the `.rfm1` to a stable release
   ([TESTING.md](docs/TESTING.md)). Or skip this and let the catalog build it for you.
3. [Ask to be listed](https://github.com/Quixotic7/remotefm1format/issues/new?template=add-firmware.yml).

### Install the skill (Claude Code)

As a plugin from this repository:

```
/plugin marketplace add Quixotic7/remotefm1format
/plugin install remotefm1-format@remotefm1
```

Or copy [`skills/add-remotefm1/`](skills/add-remotefm1/) into your firmware repository's `.claude/skills/` (or into
`~/.claude/skills/`). Then, in your firmware's repository, ask Claude to "add RemoteFM1 support", or run
`/add-remotefm1`.

## Viewing the website locally

```sh
python3 -m http.server 8000 --directory site
```

then open http://localhost:8000/. It loads its fonts from Google Fonts; everything else is in `site/`.

## Where the facts come from

This repository describes; it does not define. The tools and hosts it documents live in
[FM1VST](https://github.com/Quixotic7/FM1VST) (the build toolchain, on its branch `rfm1-manager`),
[RemoteFM1IOS](https://github.com/Quixotic7/RemoteFM1IOS) (the original format text),
[RemoteFM1FW](https://github.com/Quixotic7/RemoteFM1FW) (the catalog),
[RFM1Manager](https://github.com/Quixotic7/RFM1Manager) (the catalog tool),
[remoteFM14Move](https://github.com/Quixotic7/remoteFM14Move) (the Move host) and
[fm1-base](https://github.com/charlesvestal/fm1-base). When the code changes, [SOURCES.md](docs/SOURCES.md) says which
commits these docs were checked against.
