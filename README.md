# magiceyes compatibility

GP2X, Wiz, and Caanoo compatibility with [magiceyes](https://github.com/zdiemer/magiceyes).
Each title gets its own automatically generated GitHub issue detailing its inferred compatibility
based on a scoring system along with a screenshot and video if the game or application produced any
output. A playable rating is not a guarantee that the game will play, but it does indicate that the
emulator properly boots it and the game appears to render frames. Future passes will automate testing
of games' real playability.

## Current Status

| Platform | Titles | Playable | Ingame | Black | Incompatible | Crashed |
|---|--:|--:|--:|--:|--:|--:|
| GP2X | 631 | 490 | 16 | 35 | 90 | 0 |
| Wiz | 147 | 118 | 0 | 14 | 15 | 0 |
| Caanoo | 194 | 116 | 45 | 7 | 26 | 0 |
| **All** | **972** | **724** | **61** | **56** | **131** | **0** |

## How to read it

| Label | Meaning |
|---|---|
| `platform: GP2X` / `Wiz` / `Caanoo` | Which handheld it targets |
| `status: playable` | Held 20 fps and the picture stood up to inspection (audio may not play) |
| `status: ingame` | Real gameplay on screen with a notable gap: slow, flat, or wrong |
| `status: black` | Still running, but every frame sampled was black |
| `status: incompatible` | Never drew anything, usually died in the loader |
| `status: crashed` | Booted, then crashed the emulator |
| `group: ...` | The single thing that stopped it, like `mmio-spin` |
| `blocker: ...` | The specific culprit, like `0x90a` or `163 (mremap)` |
| `visual corruption` | The frame measured as sheared, duplicated, or noise |
| `flat fill` | Runs, but paints a single color |

## Methodology

Each title gets launched headlessly under the native engine, left alone for 25 seconds, and
watched through the shared framebuffer. START and A get tapped a few times on the way through so
the run has a chance of reaching a menu rather than sitting on a splash screen. A six second
window is recorded frame by frame for the clip. Meanwhile the engine writes a structured report of
everything it could not fully handle: syscalls it has no implementation for, symbols it failed to
resolve, device nodes it does not model, hardware registers it ignored.

The verdict comes from those two streams. Frame rate and frame count come off the framebuffer,
the blocker lists come out of the report, and the screenshot is whichever captured frame scored
highest on how much of it was drawn and how varied its colors were.

Then the chosen frame gets measured, because running and looking right are different questions. A
consistent offset between neighboring scanlines means a stride or pitch mismatch, so the image
shears. Strong self-similarity at a large offset means the screen is holding more than one copy of
itself. Pixel-to-pixel noise far above what dithered artwork reaches means the frame is memory
being read as pixels rather than a picture. Anything the frame is drawn at other than 320x240 is
wrong on its face. A title that trips one of those is `ingame`, whatever its frame rate says.

Those checks are heuristics tuned against this corpus, deliberately set to flag rather than to be
certain. They are for pointing a human at the right titles.

## Refresh Script (for devs)

It all regenerates from the main repo:

```sh
bash    tools/test/run_nas_sweep.sh
python3 tools/test/compat_report.py  --results ~/me-sweep/results
python3 tools/test/compat_publish.py --manifest tools/test/compat_manifest.json \
        --repo-dir <clone of this repo> --summary COMPATIBILITY.md --push
python3 tools/test/compat_issues.py  --manifest tools/test/compat_manifest.json \
        --repo zdiemer/magiceyes-compat \
        --shots-base-url https://github.com/zdiemer/magiceyes-compat/blob/main/shots \
        --clips-base-url https://github.com/zdiemer/magiceyes-compat/blob/main/clips \
        --clips-dir <clone of this repo>/clips
```

Issues carry a hidden marker keyed to the title, so a second run edits what is already here rather
than filing duplicates. Stopping it partway and starting again is fine.
