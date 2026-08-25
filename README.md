# magiceyes compatibility

How far every GP2X, Wiz, and Caanoo game gets when you run it under
[magiceyes](https://github.com/zdiemer/magiceyes). One issue per title, a thousand of them.
If you want to know whether some particular game works, search the issues.

No code lives here. This is the scoreboard, kept apart from the main repo so its issue list can
stay about the emulator instead of disappearing under a thousand game reports.

## Where things stand

972 titles booted headlessly on 14 August 2026 (the fifth full sweep), and on 24 August a
targeted pass re-verified 20 more titles fixed since then, updating their issues in place. The
corpus accounting is unchanged: 59 folders on the share contain no executable at all (source
dumps, skin packs, data-only add-ons), so with nothing to run they are not counted as titles and
their issues stay closed. Another 60 entries were skipped as firmware images, SDKs, or loose
readmes.

| Platform | Titles | Playable | Ingame | Black | Incompatible | Crashed |
|---|--:|--:|--:|--:|--:|--:|
| GP2X | 631 | 481 | 18 | 42 | 90 | 0 |
| Wiz | 147 | 116 | 2 | 14 | 15 | 0 |
| Caanoo | 194 | 109 | 49 | 10 | 26 | 0 |
| **All** | **972** | **706** | **69** | **66** | **131** | **0** |

That is 706 titles running properly and 775 putting real gameplay on screen, out of 972 that can
run at all: nearly three quarters of the corpus is now playable, and nothing crashes the engine.

The recent work went after the black screens, and four root causes fell. A present-path
starvation: the Wiz firmware SDL pans once at start-up, the engine locked its display to that
page, and GLBasic games then drew every frame into the other page forever (full-speed game, music
playing, black screen). A library gap: static GP2X binaries load the firmware's `libpng` at
runtime, that library never declared its zlib dependency, and the usual preload workaround cannot
reach a static binary's loader, so every PNG in those games silently failed to decode. A
touchscreen model that only reported changes, when the real hardware free-runs and certain games
wait for any sample at all before leaving their loading screen. And repeat framebuffer mappings
that each got their own private memory, so a game drawing through its second mapping was painting
somewhere the display never read: that one, plus implementing the Wiz's portrait-mode ioctl,
revived the whole Wiz GLBasic family at once, several of them straight to fully playable with
working menus and gameplay. Touch input now flows end to end as well, verified all the way into a
game's own internal mouse state.

The 24 August pass flipped those 20 issues (14 Wiz, 5 GP2X, 1 Caanoo) from `black` to `playable`
with fresh screenshots and clips; the table above includes them. A handful of siblings that also
render now still carry `black`, honestly: their newly-working menus respond to the sweep's
scripted button taps, and one of those taps exits them mid-run. They will re-grade once the sweep
script is gentler.

`playable` is stricter than "it ran". 28 titles clear every timing check and still fail on the
picture: 17 paint nothing but a flat colour, and 11 draw something visibly wrong. Those are graded
`ingame` instead, and the specific reason is in each issue.

These counts move by a handful between sweeps. Titles near a threshold, and a few that are simply
unstable, land on different sides on different days, so treat single-title verdicts as indicative
rather than final.

The black-screen bucket, long the biggest, is down to 66 titles that keep running at full speed
while drawing nothing we present: half of what it was two sweeps ago. The biggest bucket now is
quieter: 107 titles that render at full speed but never produce audio, which does not cost them
their grade but is the next thing worth a debugging session.

## How to read it

Every issue is one title, and the labels are what make the pile useful.

| Label | Meaning |
|---|---|
| `platform: GP2X` / `Wiz` / `Caanoo` | Which handheld it targets |
| `status: playable` | Held 20 fps and the picture stood up to inspection (silence alone does not demote) |
| `status: ingame` | Real gameplay on screen with a notable gap: slow, flat, or wrong |
| `status: black` | Still running, but every frame sampled was black |
| `status: incompatible` | Never drew anything, usually died in the loader |
| `status: crashed` | Booted, then took the engine down with it |
| `group: ...` | The single thing that stopped it, like `mmio-spin` |
| `blocker: ...` | The specific culprit, like `0x90a` or `163 (mremap)` |
| `visual corruption` | The frame measured as sheared, duplicated, or noise |
| `flat fill` | Runs, but paints a single colour |

The `group:` labels are the ones to look at if you want something to fix. Each title lands in
exactly one group, picked by whatever actually stopped it first, so the number of issues carrying
a label is the number of games that one fix would unblock. Sorting by that count is a decent
to-do list.

`COMPATIBILITY.md` holds the same data as tables, including every title and the blockers ranked
by how many games they hold up.

## Where the numbers come from

Each title gets launched headlessly under the native engine, left alone for 25 seconds, and
watched through the shared framebuffer. START and A get tapped a few times on the way through so
the run has a chance of reaching a menu rather than sitting on a splash screen. A six second
window is recorded frame by frame for the clip. Meanwhile the engine writes a structured report of
everything it could not fully handle: syscalls it has no implementation for, symbols it failed to
resolve, device nodes it does not model, hardware registers it ignored.

The verdict comes from those two streams. Frame rate and frame count come off the framebuffer,
the blocker lists come out of the report, and the screenshot is whichever captured frame scored
highest on how much of it was drawn and how varied its colours were.

Then the chosen frame gets measured, because running and looking right are different questions. A
consistent offset between neighbouring scanlines means a stride or pitch mismatch, so the image
shears. Strong self-similarity at a large offset means the screen is holding more than one copy of
itself. Pixel-to-pixel noise far above what dithered artwork reaches means the frame is memory
being read as pixels rather than a picture. Anything the frame is drawn at other than 320x240 is
wrong on its face. A title that trips one of those is `ingame`, whatever its frame rate says.

Those checks are heuristics tuned against this corpus, deliberately set to flag rather than to be
certain. They are for pointing a human at the right titles.

Runs happen on ext4, never on a Windows mount. Going through drvfs costs around 20% of the frame
rate, which is more than enough to shove a title across the 20 fps line and into the wrong bucket.

## Things worth knowing before you trust a result

**A failing issue is not automatically our bug.** A good number of entries on the share are patch
archives, level packs, or partial dumps. The 59 with no executable at all are no longer graded
(their issues are closed: nothing to run means nothing to mark playable or unplayable), and the
engine ports that start up and then quit because the original game's data files were never there
carry `group: missing-game-data`. There is nothing on the emulator side to fix in either pile.

**Two copies of the same game are not the same game.** Payback is the clearest case. The build on
the share sits on its loading screen forever, while a different build of the same game runs at
28 fps. Each issue records the exact path that was tested, so check that before assuming a title
is broken everywhere.

**Screenshots are chosen by a script.** Usually you get a title screen or a menu. Sometimes you
get whatever was on screen when the timer ran out, which for a game stuck mid-load is a loading
screen. When a title is flagged for a visual fault the screenshot switches to the frame that
looked worst, since that is the one worth seeing. Titles that never drew anything worth looking at
have no screenshot at all.

**The clips are real motion.** Most issues lead with an animated GIF of six seconds of actual
play, recorded at 15 fps from about eight seconds into the run, so you can see scrolling, sprites
and animation rather than a slideshow. A few older entries fall back to a time-lapse stitched from
the sampled frames. Titles whose screen never changed get no clip at all.

Recording is a straight byte copy of the framebuffer, encoded to GIF afterwards, and a control run
with and without it confirmed the frame rates do not move. The clip is a recording of the run, not
a re-run, so it matches the numbers in the issue.

**25 seconds is not a playthrough.** A title marked playable booted, ran, sounded correct and
looked correct for as long as it was watched. Nobody finished it. Treat `playable` as "starts and
behaves", not as a promise about level 7.

**The visual checks only ever saw one frame.** A title that draws a clean menu and then falls
apart in gameplay still reads as playable here, because the frame that got measured was the menu.

**Some verdicts are softer than others.** `status: black` means the engine kept running and the
game kept advancing frames, but everything sampled came out black. That is usually a video path
problem rather than a dead game, and a few of those are probably closer to working than the label
suggests.

## Refreshing this

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
than filing duplicates. Stopping it partway and starting again is fine, and since GitHub throttles
bulk issue creation, that will probably happen.
