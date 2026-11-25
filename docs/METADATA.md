# `RemoteFM1Metadata.json`

One optional file at the **root of your firmware's repository** that tells RemoteFM1 how to show your firmware:
its catalog **card**, a **skin** for the on-screen FM-1, **labels** for the panel, and the **target** release to
build. Every part is optional; without the file nothing changes, and the file never fails a build.

The authoritative guide is FW `docs/REMOTEFM1-METADATA.md`, and its JSON schema is
[`RemoteFM1Metadata.schema.json`](https://raw.githubusercontent.com/Quixotic7/RemoteFM1FW/main/docs/RemoteFM1Metadata.schema.json).
This page summarises them.

```json
{
  "$schema": "https://raw.githubusercontent.com/Quixotic7/RemoteFM1FW/main/docs/RemoteFM1Metadata.schema.json",
  "schema": 1,
  "target": "v1.4.2",
  "card": {
    "tag": "Granular sampler",
    "what": "A granular sampler with four voices, a pitch-shifting freeze and a tape delay.",
    "category": "Synth",
    "site": "https://example.github.io/my-fw/",
    "editor": "https://example.github.io/my-fw/editor/",
    "port": "My FW",
    "image": "docs/remotefm1-card.png"
  },
  "skin": {
    "background": "art/skin-portrait.png",
    "landscape": "art/skin-landscape.png",
    "dim": 0.2,
    "theme": { "body": "#1B1B22", "key": "#2C2C36", "knob": "#E8E2D0", "led": "#FF7A2F", "bezel": "#0E0E12" }
  },
  "labels": {
    "buttons": { "FX": "DRUMS", "SEL": "TRACK", "SEQ": "STEPS", "GLO": "SHIFT" },
    "knobs": { "K1": "CUTOF", "K2": "RESO", "K3": "DECAY", "K4": "MIX" }
  }
}
```

## Fields

| Field | Limits | Used by |
| --- | --- | --- |
| `$schema` | the schema URL | your editor (VS Code checks the file as you type) |
| `schema` | `1` | everyone |
| `target` | a tag in your repository (`1.4.2` also matches `v1.4.2`) | RFM1Manager: build this release instead of the newest stable one. Read from your **default branch's tip**; a missing tag or a downgrade is warned about and ignored |
| `card.tag` | 1-40 characters, one line | the subtitle under the name |
| `card.what` | 1-600 characters | the description shown when the firmware is selected |
| `card.category` | `Synth`, `Groovebox`, `Chords`, `Drums`, `Other` | where it is listed |
| `card.site` | `https://`, ≤ 300 characters | a link to your page |
| `card.editor` | `https://`, ≤ 300 characters | your web editor or manager; the apps open it with Web MIDI wired to the firmware |
| `card.port` | 1-31 characters | your USB MIDI port name exactly as the USB descriptor gives it; editors find the in-app port by it |
| `card.image` | PNG, JPEG or WebP, square, ≤ 1 MiB, 512 × 512 recommended | the card picture (copied into the catalog) |
| `skin.background`, `skin.landscape` | PNG or JPEG, ≤ 4 MiB, longer side ≤ 4096 px | packed into the `.rfm1` ([CAPABILITIES.md](CAPABILITIES.md#skin-rfm1skin)) |
| `skin.dim` | 0-0.8 | how much faceplate colour lies over the image |
| `skin.theme` | exactly 5 roles (`body key knob led bezel`) or exactly 14, each `#RRGGBB` | faceplate colours |
| `labels.buttons` | keys `FX SEL ENV LFO EDIT GLO HOME SAVE ARP SEQ PLAY REC OCT- OCT+` | fixed button names |
| `labels.knobs` | keys `SELECT ALGO PRESETS K1 K2 K3 K4 MASTER` | fixed knob names |
| each label | 0-10 printable ASCII, shown upper case; `""` = no job; ≤ 5 draws best | |

## Rules

- **Which copy:** the file is read from the release commit that is built, so a release's card, skin and labels match
  its code. Only `target` comes from the default branch's tip.
- **Paths** are relative to the repository root, `/`-separated, no `..`, no absolute paths, no symlinks; the file must
  be committed in the tagged release. Nothing is fetched from the network. The JSON itself ≤ 256 KiB.
- **Unknown or invalid fields** are skipped with a warning; the rest is used. Warnings appear in the catalog under
  your firmware's `author_meta.warnings` (FW `catalog/catalog.json`).
- **What you can't set here:** id, name, version, licence, authors, credits, source link, beta flag and hosting. They
  come from the `.rfm1` itself or from the catalog maintainer (FW `docs/REMOTEFM1-METADATA.md` §5).
- **Who wins:** the maintainer can override any card field, turn off a skin or labels (for example an image whose
  licence is unclear), and pin another release. Images must be yours to publish.

## Who reads it

- **FM1VST's build** (`fm1_add_rfm1`) reads `skin` and `labels` from the checkout it builds and packs them into the
  `.rfm1` (VST `docs/RFM1-METADATA.md`). `NO_SKIN` / `NO_LABELS` leave them out; `METADATA <file>` uses another file.
- **fm1-base's `build.py --rfm1`** does not read it; add the skin afterwards with FM1VST's `tools/rfm1_skin.py add …
  --metadata RemoteFM1Metadata.json`, as EVILGENIUS's release workflow does (EG `.github/workflows/release.yml`).
  Fixed labels need the FM1VST build (they are compiled into the shim).
- **RFM1Manager** reads `card` and `target` when it adds or updates your firmware (FW `docs/REMOTEFM1-METADATA.md`).
