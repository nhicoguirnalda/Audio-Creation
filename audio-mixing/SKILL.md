---
name: audio-mixing
description: Combines generated music, sound effects, ambience and voiceover into one balanced track with per-track gain, sidechain ducking, fades and final loudness. Use for "mix these tracks", "put music under the voice", "duck the bed under narration", "build the soundtrack for this scene", "combine music and sfx". Writes to output/mixes/. Do NOT use for single-file trims and fades (audio-editing) or for creating the elements (music-generation, sound-effects-generation).
---

# Audio Mixing

Mixes an arbitrary number of tracks into one file: gain staging, sidechain
ducking under a voiceover, fades, and a single final loudness pass. Output goes
to `output/mixes/` as WAV + MP3.

## Setup

```powershell
$PY = "D:\Downloads\OpenCode Skills\.venv\Scripts\python.exe"
$TK = "D:\Downloads\OpenCode Skills\audio-toolkit"
```

## The four-layer soundtrack

Build children's video audio in this order. Each layer has one job.

| Layer | Source skill | Level | Notes |
|---|---|---|---|
| Voiceover | your recording, in `output/voice/` | 0 dB (anchor) | always the loudest thing |
| Music bed | music-generation | -18 to -24 dB under voice | duck it, never fight it |
| Ambience | sound-effects-generation | -26 to -32 dB | glue, not content |
| SFX / stingers | sound-effects-generation | -8 to -12 dB under voice | transient only |

## Mix

```powershell
# music bed + one effect
& $PY "$TK\process.py" mix --in bgm.wav --in pop.wav --gain 0.8,0.25 --name scene1

# music under narration, auto-ducked, 20 s, fades in and out
& $PY "$TK\process.py" mix --in bgm.wav --in voice.wav --gain 0.35,1.0 `
    --duck-with voice.wav --duration 20 --fade-in 0.5 --fade-out 1.5 --name scene1

# full stack: bed + ambience + two effects, mixed with the voice
& $PY "$TK\process.py" mix --in bgm.wav --in rain.wav --in boing.wav --in voice.wav `
    --gain 0.3,0.15,0.45,1.0 --duck-with voice.wav --name scene1
```

`--duck-with` treats the first track(s) as the bed, compresses them against the
voice with a 6:1 ratio and 20 ms attack / 300 ms release, then sums voice and
ducked bed. Tune with `--threshold` (default 0.03 — lower is more sensitive) and
`--ratio`.

The final file is normalised to `--lufs` (default -16; use -14 for YouTube).

## Why ducking instead of just lowering the music

A static -20 dB bed disappears in quiet passages and gets buried in loud ones.
Sidechain compression keeps the music present between sentences and pushes it
down only while someone speaks. That is the difference between "amateur" and
"broadcast" in a 20-second clip, and it is one flag here.

## Crossfades between cues

Two-step, and always in the edit chain rather than in the mix:

```powershell
& $PY "$TK\process.py" edit --op fade --in escalation.wav --fade-in 1.0 --fade-out 1.0 --duration 8
& $PY "$TK\process.py" mix --in calm.wav --in escalation.wav --gain 0.8,0.8 --name transition
```

Overlap by 0.5-1.5 s. A crossfade shorter than 300 ms reads as a mistake; longer
than 2 s reads as a slow dissolve. For a hard cut, use a `whoosh` or `swell`
transition from **sound-effects-generation** instead of a fade.

## Checking the result

```powershell
& $PY "$TK\process.py" info --in "output\mixes\scene1_x.wav"
```

Accept when: peak below -1 dBFS, duration matches the picture, and the voice is
still intelligible at phone-speaker volume. If the mix peaks above 0 dBFS, lower
the bed gains rather than re-normalising the master.

## Pitfalls

- **Summing many loud tracks clips.** With `normalize=0` (used here) there is no
  automatic attenuation — that is deliberate, but it means gains are your job.
  Start every added layer 6 dB lower than feels right.
- **Do not normalise the stems before mixing.** It destroys the balance you are
  about to create.
- **Trim to the same length before mixing** if you want a fixed runtime; the
  mixer uses the longest input by default.
- **Order matters for `--gain`**: it maps to `--in` order, comma-separated.
- Mixing 48 kHz and 44.1 kHz sources is fine here, they are converted to 44.1 kHz
  internally. For video delivery see **ffmpeg-audio** and keep 48 kHz throughout instead.

## Hand-off

- Fix an element first → **audio-editing**
- Missing music or effects → **music-generation**, **sound-effects-generation**
- Mux the mix into an MP4, or export AAC instead → **ffmpeg-audio**
