# opencode-audio-skills

Audio generation and production skills for [OpenCode](https://opencode.ai) that run
entirely on your own machine. No API keys, no subscriptions, no credits, no uploads.

Five skills cover the whole chain: generate music, design sound effects, edit, mix,
and talk to FFmpeg directly when you need something the toolkit doesn't wrap.

```powershell
& $PY "$TK\generate.py" music --preset playful --dur 20 --loop
& $PY "$TK\generate.py" sfx --type boing --seed 4
& $PY "$TK\process.py" mix --in bed.wav --in voice.wav --duck-with voice.wav --lufs -14
```

## Skills

| Skill | What it does | Writes to |
|---|---|---|
| **music-generation** | Background music via MusicGen Small on your GPU, with an instant procedural fallback | `output/music/childrens/` |
| **sound-effects-generation** | 15 cartoon SFX, 10 ambience beds, 7 transitions — synthesised from scratch in NumPy | `output/sfx/`, `output/ambient/`, `output/transitions/` |
| **audio-editing** | Trim, fade, normalise, seamless loop, pitch, speed, reverse, silence strip, convert | `output/edits/` |
| **audio-mixing** | Multi-track mixing, sidechain ducking under narration, final loudness | `output/mixes/` |
| **ffmpeg-audio** | Reference layer: codecs, two-pass loudnorm, concat, crossfade, muxing audio into video | — |

Skills are written for children's content and short-form video, but nothing is
specific to that — the presets and level targets are just sensible defaults.

## Requirements

- **Python 3.12** — 3.13+ has no prebuilt CUDA wheels for this stack
- **FFmpeg** with `libmp3lame` (a gyan.dev full build works)
- **NVIDIA GPU** with 4 GB+ VRAM for MusicGen. Everything else runs on CPU
- ~5 GB of disk for the venv, plus ~2 GB once MusicGen weights are cached

Verified on: Ryzen 5 3400G, 16 GB RAM, GTX 1650 4 GB, Windows 11, FFmpeg 9.0.2,
torch 2.7.1+cu126.

## Install

```powershell
git clone https://github.com/<you>/opencode-audio-skills
cd opencode-audio-skills

# creates .venv, installs torch + numpy + soundfile + transformers, verifies CUDA
& .\audio-toolkit\setup.cmd
```

`setup.cmd` is idempotent and skips anything already present. It calls
`setup.ps1` with an execution-policy bypass, so it works on default Windows
configurations.

Then link the skills into OpenCode:

```powershell
$dst = "$env:USERPROFILE\.config\opencode\skills"
foreach ($n in "music-generation","sound-effects-generation","audio-editing","audio-mixing","ffmpeg-audio") {
  New-Item -ItemType Junction -Path (Join-Path $dst $n) -Target (Join-Path $PWD "skills\$n")
}
```

Restart OpenCode. The skills now load in every project, and edits to this repo
take effect immediately — no copying.

Set these two for the examples below:

```powershell
$PY = ".\\.venv\\Scripts\\python.exe"
$TK = ".\\audio-toolkit"
```

## Usage

You don't have to run the scripts yourself. Describe what you want and the
matching skill loads.

### Music

```powershell
& $PY "$TK\generate.py" music --preset bedtime --dur 30 --loop
& $PY "$TK\generate.py" music --preset silly --seed 7
& $PY "$TK\generate.py" music --prompt "toy piano and glockenspiel, 100 bpm, gentle" --dur 20
& $PY "$TK\generate.py" music --preset learning --dur 15 --engine procedural
```

Presets: `playful bedtime adventure silly magical learning lullaby celebration`.

MusicGen emits 50 tokens/second, so one pass stops at 30 s. `--loop` crossfades
the tail into the head for a seamless loop, which is usually better than
generating three minutes of audio. Expect roughly 5x realtime on a GTX 1650
(30 s of music in about 2.5 minutes); the procedural engine takes one second.

### Sound effects, ambience, transitions

```powershell
& $PY "$TK\generate.py" sfx --type boing --seed 4 --wide
& $PY "$TK\generate.py" ambient --type forest --dur 60 --loop
& $PY "$TK\generate.py" transition --type riser
& $PY "$TK\generate.py" list
```

SFX: `boing pop whoosh zap splat twinkle bell click slide thud honk bubble sparkle swoosh riser`
Ambient: `rain wind ocean birds crickets forest night stream room magic`
Transitions: `riser downlifter whoosh sting swell glitch bubble`

All of it is deterministic under `--seed`, needs no model, and takes milliseconds.

### Edit

```powershell
& $PY "$TK\process.py" info --in track.wav
& $PY "$TK\process.py" edit --op trim --in track.wav --start 1.2 --end 4.8
& $PY "$TK\process.py" edit --op loop --in bed.wav --duration 60 --xfade 1.0
& $PY "$TK\process.py" edit --op normalize --in track.wav --lufs -14
```

Ops: `trim fade normalize loop pitch speed reverse silence convert gain mono stereo`.

### Mix

```powershell
& $PY "$TK\process.py" mix --in bgm.wav --in rain.wav --in boing.wav --in voice.wav `
    --gain 0.3,0.15,0.45,1.0 --duck-with voice.wav --lufs -14 --name scene1
```

`--duck-with` compresses the bed against the voice (6:1, 20 ms attack, 300 ms
release) so the music stays present between sentences and drops only while
someone speaks.

### FFmpeg

`ffmpeg-audio` is documentation, not code: probing, codec choice, two-pass
`loudnorm`, limiters, concat, `acrossfade`, extracting or replacing audio in a
video, batch loops, and the failure modes that cost people an afternoon. Ask for
it when you need a command the toolkit doesn't wrap.

## Full pipeline

A 20-second clip with narration, start to finish.

```powershell
$PY = ".\\.venv\\Scripts\\python.exe"
$TK = ".\\audio-toolkit"

function Newest($pattern) {
    (Get-ChildItem $pattern | Sort-Object LastWriteTime | Select-Object -Last 1).FullName
}

& $PY "$TK\generate.py" music --preset playful --dur 20 --loop --name scene1-bgm
& $PY "$TK\generate.py" sfx --type boing --seed 4 --name scene1-boing
& $PY "$TK\generate.py" ambient --type room --dur 20 --loop --name scene1-room
& $PY "$TK\generate.py" transition --type whoosh --name scene1-whoosh

# record or drop narration into output/voice/vo.wav, then:
& $PY "$TK\process.py" edit --op silence --in "output\voice\vo.wav" --threshold -45

& $PY "$TK\process.py" mix `
    --in (Newest "output\music\childrens\scene1-bgm_*.wav") `
    --in (Newest "output\ambient\scene1-room_*.wav") `
    --in (Newest "output\edits\silence-vo_*.wav") `
    --gain 0.3,0.12,1.0 --duck-with (Newest "output\edits\silence-vo_*.wav") `
    --duration 20 --fade-out 1.5 --lufs -14 --name scene1-final
```

## Output layout

```
output/
  music/childrens/   scene bed music
  sfx/cartoon/       cartoon and action effects
  sfx/ui/            clicks, pops, notification bells
  ambient/           looping beds
  transitions/       risers, whooshes, stingers
  edits/             processed single files
  mixes/             finished multi-track mixes
  voice/             drop narration recordings here
  _manifest.jsonl    every generated file, with prompt and seed
```

Files land as WAV + MP3 (192 kbps), loudness-normalised: music and SFX at -16
LUFS, ambience at -24 (room tone at -45), mixes at whatever `--lufs` you pass.
The manifest makes a session reproducible — prompt, seed and engine per file.

## How it works

```
audio-toolkit/
  audiotk.py    folders, WAV/MP3 export, loudness, seamless loop, manifest
  synth.py      procedural SFX, ambience, transitions, chiptune (pure NumPy)
  musicgen.py   MusicGen Small: fp16, chunked generation, 32 kHz -> 44.1 kHz
  generate.py   CLI: music | sfx | ambient | transition | list | status
  process.py    CLI: info | edit | mix
  selftest.py   renders every generator and asserts it is sane
```

Signals are built from oscillators, filtered noise and envelopes rather than
samples, so there are no licence questions and no download. MusicGen is the only
model in the pipeline; everything else is NumPy plus FFmpeg.

## Testing

```powershell
& $PY "$TK\selftest.py"          # synthesis only
& $PY "$TK\selftest.py" --files  # also round-trips WAV + MP3 through the exporter
```

It renders every SFX, transition, ambience bed and music preset, checks each is
non-silent and the right length, and verifies the loop seam is continuous.

## Limits worth knowing

- **MusicGen Small weights are CC-BY-NC 4.0** — free for personal and
  non-commercial use, not for monetised uploads. The procedural engine has no
  restrictions.
- One MusicGen pass is 30 s. `--chunks` goes longer but the melody drifts;
  loop-and-repeat sounds better.
- The procedural chiptune engine snaps to whole bars, so `--dur 20` may give
  17-22 s. Trim or loop if the exact length matters.
- MusicGen is a music model. It makes poor one-shot effects — use the
  procedural generators or record the real thing.
- No paid and no credit-based services anywhere, by design.
  `audio-toolkit/references/external-engines.md` lists the free open-weights
  options (Stable Audio Open, Demucs, Piper) if you need more than this pipeline
  provides.

## Licence

Code in this repository is MIT. Generated procedural audio is yours with no
attribution. MusicGen weights and FFmpeg binaries carry their own licences —
see above.
