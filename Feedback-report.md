# Music: lyrics and Up Next panel animation is laggy

**Product:** Music (macOS)
**Area:** Main window > Lyrics / Up Next side panel
**Type:** Performance

## Environment
- macOS 27.3 (26C5035r)
- Music 1.7.3
- MacBook with M1 Max

## Problem
When I open or close the Lyrics panel on the right side of the Music window, the animation is slow and stutters. Up Next does the same thing. This has been like this for years across several macOS versions, so it isn't new in this release. Turning on Reduce Motion in Accessibility > Display changes nothing.

## Steps
1. Open Music and play a song that has lyrics.
2. Show the Lyrics panel, then hide it, and repeat a few times.
3. Do the same with Up Next.

## Expected
The panel opens and closes smoothly.

## Actual
The animation runs at a low frame rate and visibly stutters.

## What I found
I investigated this with the help of an AI coding assistant (Claude Code). It disassembled the Music binary and captured profiles of the app while I toggled the panel. I tested the workaround below myself. The code-level findings come from that static analysis and profiling, so treat the exact internals as a best-effort reading.

All recordings and profiles below are of the Lyrics panel only.

**Profile.** Claude Code captured two 6 second `sample` profiles of Music's main thread while I opened the Lyrics panel once, with playback paused (attached). The first has the default animation, the second has it turned off with the workaround below. `sample` was asked for a 1 ms interval but collected about one sample per 2 ms, so the counts are samples, not milliseconds. Compare them relatively, as a share of the main thread's samples.

| Main thread, one toggle | Animation on (2,892 samples) | Animation off (3,162 samples) |
|---|---|---|
| Busy (not idle) | 835 (29%) | 460 (15%) |
| `NSAnimationManager performAnimations` | 345 (12%) | 12 (0.4%) |
| View layout (`layoutSubtreeIfNeeded`) | 494 (17%) | 210 (7%) |
| SwiftUI (`ViewGraph` / AttributeGraph) | 464 (16%) | 221 (7%) |

The SwiftUI row is mostly nested inside the layout row, so the rows shouldn't be added together. With the animation on, most of the layout work (321 of 494 samples) happens inside the animation's per-frame callback (`NSAnimationManager performAnimations` > `layoutIfNeeded`). With the animation off, almost none does (6 of 210), and the layout that remains happens outside the animation. The share of main-thread samples that are busy drops by about half. Each condition is a single run, so the exact numbers are approximate.

Based on the profile and the binary, the panel appears to be an `NSSplitViewItem` collapsed through an animated `setCollapsed:`. My reading is that resizing the main content area during that animation forces repeated layout passes that include SwiftUI-hosted content.

**Workaround.** Music reads a hidden user default called `animate_inspector_expansion` to decide whether to animate this. With
`defaults write com.apple.Music animate_inspector_expansion -bool NO`
the Lyrics panel opens and closes instantly, with none of the stutter, as the attached recordings and profiles show. Up Next also felt much cleaner with the workaround when I used the app, but I did not record or profile that, so that part is only my own observation. It is a workaround, not a fix: it removes the animation instead of making it cheaper, and the profile above still shows some main-thread work (460 busy samples) when the panel opens with the animation off. The result suggests the animated path is a major contributor to the lag.

## Suggestion
Make the panel animation cheaper, for example by not re-laying out the window's content on every frame, so the animation can stay on by default.

## Attached
- `music-lyrics-animation-default.mov`: screen recording of toggling the Lyrics panel with the default animation (stutters)
- `music-lyrics-animation-workaround.mov`: same, with `animate_inspector_expansion` set to NO (instant)
- `Music-lyrics-one-toggle-animation-on.sample.txt`: 6 second `sample` of Music, one lyrics panel open, default animation
- `Music-lyrics-one-toggle-animation-off.sample.txt`: same, with `animate_inspector_expansion` set to NO
