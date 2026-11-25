# The RemoteFM1 catalog: how firmwares get listed

This is the official route for asking that an FM-1 firmware be added to the RemoteFM1 catalog, and what happens
after you ask. The catalog is the list of firmwares that remotefm1.com, the RemoteFM1 apps and RemoteFM1 on the
Ableton Move offer to download.

Anyone can ask: the firmware's author, or a fan who wants to play it. (Authors: how to add `.rfm1` and `.rfm1m`
files to your own releases is in [BUILDING.md](BUILDING.md) and [RFM1M.md](RFM1M.md#building-the-move-version-yourself).)

## How to ask

**Open an issue with the form "Add a firmware to the RemoteFM1 catalog"** in the catalog repository,
[Quixotic7/RemoteFM1FW](https://github.com/Quixotic7/RemoteFM1FW)
([new issue](https://github.com/Quixotic7/RemoteFM1FW/issues/new?template=add-firmware.yml)). One issue per
firmware. If you aren't the author, the author is told before anything is listed.

To change or remove a listing, use the form "Change or remove a catalog listing".

## Two cases

### The firmware's releases have `.rfm1` files

**A link is all we need.** The file is listed exactly as its author made it, and what is in the firmware is the
author's business: its licence, samples and presets are their responsibility, not a condition of listing.

The author's Ableton Move version (`.rfm1m`) belongs beside it in the same release
([RFM1M.md](RFM1M.md#building-the-move-version-yourself)). Right now the Move catalog still compiles its own copy from
the `.rfm1`.

When the files are on the firmware's **GitHub releases**, each newer release is picked up automatically (see
[Updates](#updates)).

### The firmware only has FM-1 releases (`.fwsc`)

Then the maintainer may build the `.rfm1` from its source, with FM1VST's glue or fm1-base's tool. Because we then make
and distribute a binary of someone else's work, these rules apply, and the form's licence questions are for this case:

1. **It is an FM-1 firmware.**
2. **Its source is public**, in a GitHub repository, with a **licence that allows anyone to redistribute it in
   binary form** (GPL, MPL, MIT and the like). A repository with no licence cannot be built and listed.
3. **It has a release** to build from. A stable release is preferred: a GitHub release that is not a draft and not a
   pre-release, with a plain version tag (`v1.2.3`, `name-v1.2.3`); tags with rc, beta, alpha, pre, preview, dev,
   nightly, test, snapshot, draft or wip in them are not stable (MGR `lib/util.js`). A beta can be listed on request.
4. **Everything in it may be redistributed.** Samples, wavetables, fonts, images and presets that are not the
   author's must have a licence that allows it. **No third-party preset dumps or raw `.syx` / `.mid` / `.wav` files
   are bundled** (MGR `lib/publish.js` refuses those file types). Presets compiled into the firmware are fine when
   their licence allows it.

Listing is the maintainer's decision either way (FW `docs/AUTHOR-RFM1.md`).

## What happens to your request

1. The maintainer reads it and asks any questions on the issue.
2. The firmware is added with RFM1Manager: it takes the author's `.rfm1`, or builds one, and tries it: it must load, make
   sound and draw its screen (MGR `README.md`, `lib/rfm1.js`).
3. It appears at the next catalog publish, and on the Move at the next Move catalog build. The issue is answered
   with the result.

For the checks themselves (`rfm1_check.py`, `rfm1_run`, the Move's bit-identical run), see [TESTING.md](TESTING.md);
running them before you ask saves a round trip.

## Your file or ours

- **The author's `.rfm1`** is used **unchanged**: only renamed to `<id>-<version>.rfm1`. The catalog credits it as
  built by the author (FW `README.md` "Build provenance"; MGR `lib/authorrfm1.js`).
- **Ours** is built from the release's tagged commit and published as the catalog's build.
- Either way the catalog **hosts a copy**: the files live in the RemoteFM1FW release `current` with a `SHA256SUMS`
  file, and remotefm1.com re-hosts them daily (browsers can't download GitHub release files directly) (MGR
  `lib/authorrfm1.js`; FW `docs/AUTHOR-RFM1.md`).

## Updates

- **`.rfm1` files on GitHub releases:** nothing to ask. Every night at 03:20 UTC a GitHub Action in RemoteFM1FW looks
  at the newest stable release; if it has an `.rfm1` with the same `id` and a **newer version that equals the tag**,
  and it loads and plays, it replaces the listed one. The site picks it up at its 04:17 UTC deploy. If something
  fails, the old file stays and the run's summary says why (FW `docs/AUTHOR-RFM1.md`).
- **`.rfm1` files somewhere else:** send the new link with the "Change or remove" form.
- **We build it:** the maintainer rebuilds when the firmware has a newer release, at the next catalog publish.
- **Pick a release yourself:** `target` in your `RemoteFM1Metadata.json` names the tag to use (to hold back a broken
  latest release, or to opt in a beta) ([METADATA.md](METADATA.md)).
- **Card text, picture, skin, labels:** change them in your `RemoteFM1Metadata.json`; they are read again at your
  next release. Or ask with the "Change or remove" form.

## Removals

Open the "Change or remove a catalog listing" form (the author, or anyone with a licence or rights concern). The
maintainer removes the firmware's files at the next publish; removed files disappear from the `current` release
(MGR `README.md` §7). Users who already downloaded a firmware keep their copy.

## Hosted-only and gated

Two words appear in the catalog's tooling:

- **Gated** (an FM1VST core flag): a firmware whose rights are not settled for redistribution **inside** apps or
  public native builds. A gated firmware is not built as an `.rfm1` by the catalog, unless the maintainer marks it
  hosted-only (VST `cmake/FM1Cores.cmake`, `cmake/FM1Rfm1.cmake`).
- **Hosted-only** (`HOSTED_DESPITE_GATE`): the firmware is only **hosted for download**: RemoteFM1 doesn't bundle it
  in any app; the app downloads and runs the file the user chooses. The RemoteFM1FW README marks it †
  (FW `README.md`; VST `docs/RFM1.md` "Licences").

Both are the maintainer's call (FW `docs/REMOTEFM1-METADATA.md` §5).

**Fantasy CFWs** (marked ‡) run in RemoteFM1 apps only, never on an FM-1 ([CAPABILITIES.md](CAPABILITIES.md#fantasy-cfw-virtual-fs1)).

## What we ask of authors

- Keep the links working; if a file moves, tell us with the "Change or remove" form.
- Don't change a published `.rfm1` in place; make a new release with a new version.

Fields the form asks for, and why: the link to the release with the `.rfm1` and `.rfm1m` (or the repository and
release, for us to build), the id, category, subtitle, description, picture, site, editor and USB port name (the catalog card: FW
`catalog/SCHEMA.md`), the capabilities (which hosts to test it on). The licence and third-party questions are only
for `.fwsc`-only firmwares that we build.
