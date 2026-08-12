# magiceyes compatibility

How far every GP2X, Wiz, and Caanoo game gets when you run it under
[magiceyes](https://github.com/zdiemer/magiceyes). One issue per title, a thousand of them.
If you want to know whether some particular game works, search the issues.

No code lives here. This is the scoreboard, kept apart from the main repo so its issue list can
stay about the emulator instead of disappearing under a thousand game reports.

## Where things stand

1,031 titles booted headlessly on 11 August 2026. Another 60 entries on the share were skipped
because they are firmware images, SDKs, or loose readmes rather than games.

| Platform | Titles | Playable | Renders | Black | Incompatible |
|---|--:|--:|--:|--:|--:|
| GP2X | 673 | 224 | 76 | 134 | 239 |
| Wiz | 153 | 26 | 7 | 25 | 95 |
| Caanoo | 205 | 21 | 21 | 15 | 148 |
| **All** | **1,031** | **271** | **104** | **174** | **482** |

So a bit over a quarter run properly, and a bit over half get far enough to put something on
screen. Nothing crashed the engine, which is the one number worth being smug about.

The biggest single lead is a register. 25 of the 27 titles that hang forever are sitting in a
tight loop reading MMSP2 register `0x90a`, tens of millions of times a second, waiting for a value
that never changes. That same register comes up as unhandled in 409 titles overall. It looks like
one fix with a very long tail.

## How to read it

Every issue is one title, and the labels are what make the pile useful.

| Label | Meaning |
|---|---|
| `platform: GP2X` / `Wiz` / `Caanoo` | Which handheld it targets |
| `status: playable` | Held 25 fps or better, drew real frames, made sound |
| `status: renders` | Drew real frames, but ran slow or came out silent |
| `status: black` | Still running, but every frame sampled was black |
| `status: incompatible` | Never drew anything, usually died in the loader |
| `status: crashed` | Booted, then took the engine down with it |
| `group: ...` | The single thing that stopped it, like `mmio-spin` |
| `blocker: ...` | The specific culprit, like `0x90a` or `163 (mremap)` |

The `group:` labels are the ones to look at if you want something to fix. Each title lands in
exactly one group, picked by whatever actually stopped it first, so the number of issues carrying
a label is the number of games that one fix would unblock. Sorting by that count is a decent
to-do list.

`COMPATIBILITY.md` holds the same data as tables, including every title and the blockers ranked
by how many games they hold up.

## Where the numbers come from

Each title gets launched headlessly under the native engine, left alone for 25 seconds, and
watched through the shared framebuffer. START and A get tapped a few times on the way through so
the run has a chance of reaching a menu rather than sitting on a splash screen. Meanwhile the
engine writes a structured report of everything it could not fully handle: syscalls it has no
implementation for, symbols it failed to resolve, device nodes it does not model, hardware
registers it ignored.

The verdict comes from those two streams. Frame rate and frame count come off the framebuffer,
the blocker lists come out of the report, and the screenshot is whichever captured frame scored
highest on how much of it was drawn and how varied its colours were.

Runs happen on ext4, never on a Windows mount. Going through drvfs costs around 20% of the frame
rate, which is more than enough to shove a title across the 25 fps line and into the wrong bucket.

## Things worth knowing before you trust a result

**A failing issue is not automatically our bug.** A good number of entries on the share are patch
archives, level packs, or partial dumps. 59 have no executable in them at all, and another 106 are
engine ports that start up and then quit because the original game's data files were never there.
Those are `group: no-executable` and `group: missing-game-data`, and there is nothing on the
emulator side to fix.

**Two copies of the same game are not the same game.** Payback is the clearest case. The build on
the share sits on its loading screen forever, while a different build of the same game runs at
28 fps. Each issue records the exact path that was tested, so check that before assuming a title
is broken everywhere.

**Screenshots are chosen by a script.** Usually you get a title screen or a menu. Sometimes you
get whatever was on screen when the timer ran out, which for a game stuck mid-load is a loading
screen. Titles that never drew anything worth looking at have no screenshot at all.

**25 seconds is not a playthrough.** A title marked playable booted, ran, and sounded correct for
as long as it was watched. Nobody finished it. Treat `playable` as "starts and behaves", not as a
promise about level 7.

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
        --shots-base-url https://github.com/zdiemer/magiceyes-compat/blob/main/shots
```

Issues carry a hidden marker keyed to the title, so a second run edits what is already here rather
than filing duplicates. Stopping it partway and starting again is fine, and since GitHub throttles
bulk issue creation, that will probably happen.
