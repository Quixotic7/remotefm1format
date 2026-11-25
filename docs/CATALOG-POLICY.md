# The RemoteFM1 catalog: how firmwares get listed

This is the official route for asking that an FM-1 firmware be added to the RemoteFM1 catalog, and what happens
after you ask. The catalog is the list of firmwares that remotefm1.com, the RemoteFM1 apps and RemoteFM1 on the
Ableton Move offer to download.

## How to ask

**Open an issue with the form "Add a firmware to the RemoteFM1 catalog"** in the catalog repository,
[Quixotic7/RemoteFM1FW](https://github.com/Quixotic7/RemoteFM1FW)
([new issue](https://github.com/Quixotic7/RemoteFM1FW/issues/new?template=add-firmware.yml)). One issue per
firmware. You don't have to be the author, but if you aren't, the author is asked before anything is listed.

To change or remove a listing, use the form "Change or remove a catalog listing".

## What is accepted

A firmware is listed when all of these hold:

1. **It is an FM-1 firmware** (or a Fantasy CFW: a firmware made only for RemoteFM1 hosts).
2. **Its source is public**, in a GitHub repository, with a **licence that allows anyone to redistribute it in
   binary form** (GPL, MPL, MIT and the like). A repository with no licence cannot be listed.
3. **It has a stable release**: a GitHub release that is not a draft and not a pre-release, with a plain version tag
   (`v1.2.3`, `name-v1.2.3`). Tags with rc, beta, alpha, pre, preview, dev, nightly, test, snapshot, draft or wip in
   them are not stable (MGR `lib/util.js`). A stable release is preferred before a firmware is listed; if you only have a beta and want it
   listed, ask for it in the request and it can be added.
4. **Everything in it may be redistributed.** Samples, wavetables, fonts, images and presets that are not yours must
   have a licence that allows it. **No third-party preset dumps or raw `.syx` / `.mid` / `.wav` files are bundled**:
   the publish step refuses those file types outright (MGR `lib/publish.js`). Presets compiled into the firmware are
   fine when their licence allows it.
5. **It runs:** its `.rfm1` passes the checks below.

Listing is the maintainer's decision. Adding a new firmware, and whether a firmware is hosted, is never automatic
(FW `docs/AUTHOR-RFM1.md`).

## What is checked before listing

Whether you attach your own `.rfm1` or the maintainer builds it, the same checks run (MGR `README.md`, `lib/rfm1.js`,
`lib/publish.js`; FW `docs/AUTHOR-RFM1.md`):

- **The format:** FM1VST's `tools/rfm1_check.py` passes ([TESTING.md](TESTING.md)).
- **It plays:** `rfm1_run` boots the file from a fresh flash, plays a chord that sounds, and draws the screen, with no
  trap. (A looper that is silent by design is judged against its previous file.)
- **The manifest:** `id`, `name`, `version`, `license`, a non-empty `authors`, and `source_url` naming **your
  repository at a commit** (`https://github.com/<you>/<repo>@<commit>`), and at least one licence text inside the file.
- **The version:** equals the release tag's version (`v1.2.3` → `1.2.3`).
- **The licence and the repository:** read by the maintainer, including where samples and art come from.
- **The card:** picture square, PNG / JPEG / WebP, at most 1 MiB; text within its limits ([METADATA.md](METADATA.md)).
- **For the Move**, later and separately: the file is compiled for the Move and kept only if its sound is bit for bit
  the same as the reference run ([RFM1M.md](RFM1M.md)). A firmware that fails there is still listed for the other
  hosts.

## Your file or ours

- **If your stable release has an `.rfm1` attached**, it is used **unchanged**: only renamed to
  `<id>-<version>.rfm1`. The catalog credits it as built by you (FW `README.md` "Build provenance"; MGR
  `lib/authorrfm1.js`).
- **Otherwise** the maintainer builds it from your tagged commit with FM1VST's glue (or fm1-base's tool) and
  publishes that build. You don't have to do anything.
- Either way the catalog **hosts a copy**: the files live in the RemoteFM1FW release `current` with a `SHA256SUMS`
  file, and remotefm1.com re-hosts them daily (browsers can't download GitHub release files directly) (MGR
  `lib/authorrfm1.js`; FW `docs/AUTHOR-RFM1.md`).

## Updates

- **You ship the `.rfm1`:** nothing to ask. Every night at 03:20 UTC a GitHub Action in RemoteFM1FW looks at your
  newest stable release; if it has an `.rfm1` that passes the checks, is the same `id`, and is a **newer version that
  equals the tag**, it replaces the listed one. The site picks it up at its 04:17 UTC deploy. If a check fails, the
  old file stays and the run's summary says why (FW `docs/AUTHOR-RFM1.md`).
- **We build it:** the maintainer rebuilds when you make a newer stable release, at the next catalog publish.
- **Pick a release yourself:** `target` in your `RemoteFM1Metadata.json` names the tag to build (to hold back a broken
  latest release, or to opt in a beta) ([METADATA.md](METADATA.md)).
- **Card text, picture, skin, labels:** change them in your `RemoteFM1Metadata.json`; they are read again at your
  next release. Or ask with the "Change or remove" form.

## Removals

Open the "Change or remove a catalog listing" form (the author, or anyone with a licence or rights concern). The
maintainer removes the firmware's files at the next publish; removed files disappear from the `current` release
(MGR `README.md` §7). Users who already downloaded a firmware keep their copy. A rights problem is handled first.

## Hosted-only and gated

Two words appear in the catalog's tooling:

- **Gated** (an FM1VST core flag): a firmware whose rights are not settled for redistribution **inside** apps or
  public native builds (samples, art or other material with an open question). A gated firmware is not built as an
  `.rfm1` at all, unless the maintainer marks it hosted-only (VST `cmake/FM1Cores.cmake`, `cmake/FM1Rfm1.cmake`).
- **Hosted-only** (`HOSTED_DESPITE_GATE`): the firmware is built exactly as its author ships it and only **hosted for
  download**: RemoteFM1 doesn't bundle it in any app; the app downloads and runs the file the user chooses. Each such
  entry has a reason, and the RemoteFM1FW README marks it †. Examples: SLOOP and Clipper (an icon atlas), X0X (drum
  samples) (FW `README.md`; VST `docs/RFM1.md` "Licences").

Authors can't set either: they are the maintainer's call (FW `docs/REMOTEFM1-METADATA.md` §5). Tell us in your request
about anything that might make your firmware hosted-only.

**Fantasy CFWs** (marked ‡) run in RemoteFM1 apps only, never on an FM-1 ([CAPABILITIES.md](CAPABILITIES.md#fantasy-cfw-virtual-fs1)).

## What we ask of you

- Keep the commit your `.rfm1` names public, and keep `source_url` equal to the tagged commit.
- Don't change a release's `.rfm1` after publishing it; make a new release.
- Tell us if your firmware's licence or contents change.

## What happens to your request

1. The maintainer reads it, checks the licence and the repository, and asks any questions on the issue.
2. The firmware is added with RFM1Manager: it uses your `.rfm1` or builds one, and runs every check above.
3. It appears at the next catalog publish, and on the Move at the next Move catalog build. The issue is answered
   with the result.

Fields the form asks for, and why: the repository and stable release (RFM1Manager's input), the licence and
third-party material (the rights check), how the `.rfm1` is made (your file or ours), the id, category, subtitle,
description, picture, site, editor and USB port name (the catalog card: FW `catalog/SCHEMA.md`), the capabilities
(which hosts to test it on).
