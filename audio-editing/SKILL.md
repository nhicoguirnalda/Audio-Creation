---
name: audio-editing
description: Non-destructive edits on existing audio files - trim, fade, normalise loudness, seamless loop, pitch, speed, reverse, silence strip, channel and format conversion. Use for "trim this clip", "fade out the music", "make it loop", "normalise to -16 LUFS", "cut the silence", "convert to mp3", "pitch this down". Writes results to output/edits/. Do NOT use for generating new audio (music-generation, sound-effects-generation) or for combining tracks (audio-mixing).
---

# Audio Editing

One CLI, one operation at a time, FFmpeg underneath. Every result is written to
`output/edits/` as WAV + MP3 unless told otherwise, and loudness-normalised to
-16 LUFS (pass `--lufs -99` to keep the original level).

## Setup

```powershell
$PY = "D:\Downloads\OpenCode Skills\.venv\Scripts\python.exe"
$TK = "D:\Downloads\OpenCode Skills\audio-toolkit"
& $PY "$TK\process.py" info --in "output\music\childrens\bgm-playful_x.wav"
```

Check `info` before and after an edit: duration, sample rate, channel count and
true peak tell you whether the edit did what you meant.

## Operations

```powershell
# trim to a section
& $PY "$TK\process.py" edit --op trim --in in.wav --start 1.2 --end 4.8

# fades (fade-out needs --duration, or the file length is used)
& $PY "$TK\process.py" edit --op fade --in in.wav --fade-in 0.2 --fade-out 1.5 --duration 20

# loudness for a platform
& $PY "$TK\process.py" edit --op normalize --in in.wav --lufs -14

# seamless loop of a short bed
& $PY "$TK\process.py" edit --op loop --in bgm.wav --duration 60 --xfade 1.0

# pitch without changing tempo
& $PY "$TK\process.py" edit --op pitch --in in.wav --semitones -3

# tempo without changing pitch
& $PY "$TK\process.py" edit --op speed --in in.wav --factor 1.15

# strip leading/trailing silence from a recording
& $PY "$TK\process.py" edit --op silence --in vo.wav --threshold -45

# small level nudge, channel fixes, format conversion
& $PY "$TK\process.py" edit --op gain --in in.wav --db -3
& $PY "$TK\process.py" edit --op stereo --in mono.wav
& $PY "$TK\process.py" edit --op convert --in in.wav --format mp3 --bitrate 192k
```

`--op reverse` for reversed cymbals and reverse-whooshes.
`--op mono` to fold to mono before a phone-speaker review.

## Loop correctly

`--op loop` tiles the file and then crossfades the tail over the head, so the
loop point has no click. It is not the same as repeating a file: a naive repeat
usually produces a click at every seam because the end and start do not match.
If the source is musical, make it an exact number of bars first (`--op trim`),
then loop.

## Loudness targets

| Deliverable | Integrated | True peak |
|---|---|---|
| YouTube / Web video | -14 LUFS | -1 dBTP |
| Broadcast-style / TV | -23 LUFS | -1 dBTP |
| Game / app SFX bed | -16 LUFS | -1.5 dBTP |
| Podcast / narration | -16 LUFS | -1 dBTP |
| Music-only file for further mixing | leave alone (`--lufs -99`) | -1 dBTP |

Normalise **once, at the end**. Normalising, then mixing, then normalising again
just raises the noise floor.

## Pitfalls

- **Do not re-encode MP3 repeatedly.** Each pass loses quality. Work in WAV and
  export MP3 once, last.
- **MP3 adds ~50 ms of padding** at the start. Trim in WAV if frame-accurate sync
  matters; for SFX placed by ear it does not.
- **Big speed changes** (`--factor` below 0.5 or above 2) are chained internally
  and can smear transients. Prefer pitch-preserving `--op pitch`.
- **Clipping after a gain or mix** is invisible until playback: always check the
  `info` peak afterwards.
- FFmpeg's `-ss` before `-i` is fast but keyframe-seeking; this tool places it
  correctly for audio. See **ffmpeg-audio** if writing commands by hand.

## Hand-off

- Combine the edited result with other tracks → **audio-mixing**
- Need new material first → **music-generation**, **sound-effects-generation**
- Hand-written FFmpeg, codecs, video muxing → **ffmpeg-audio**
