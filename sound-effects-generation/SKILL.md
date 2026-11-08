---
name: sound-effects-generation
description: Generates cartoon sound effects, ambient beds and transition stingers procedurally with NumPy, plus free AI options when a real-world recording is needed. Use for "add a boing", "cartoon pop", "whoosh transition", "rain ambience", "kids video sound design", "scene transition sound". Writes to output/sfx/cartoon/, output/sfx/ui/, output/ambient/ and output/transitions/. Do NOT use for music (music-generation) or for mixing a full scene (audio-mixing).
---

# Sound Effects, Ambience & Transitions

Everything here is synthesised from scratch with NumPy: no downloads, no GPU,
no licence questions, deterministic under `--seed`. Perfect for cartoon and
children's content where clean, readable sounds beat realism.

## Setup

```powershell
$PY = "D:\Downloads\OpenCode Skills\.venv\Scripts\python.exe"
$TK = "D:\Downloads\OpenCode Skills\audio-toolkit"
& $PY "$TK\generate.py" list
```

## Cartoon SFX

```powershell
& $PY "$TK\generate.py" sfx --type boing
& $PY "$TK\generate.py" sfx --type pop --wide
& $PY "$TK\generate.py" sfx --type twinkle --seed 3
```

| Type | Sounds like | Typical use |
|---|---|---|
| `boing` | springy pitch wobble | squash/stretch, jump |
| `pop` | tiny mouth pop | bubble, appear |
| `whoosh` | filtered noise sweep | object passes camera |
| `zap` | fast sawtooth drop | laser, magic hit |
| `splat` | gated noise burst | paint, food, mud |
| `twinkle` | ascending bell arpeggio | magic reveal, success |
| `bell` | single FM bell | notification, ding |
| `click` | 60 ms transient | UI tap |
| `slide` | slide-whistle gliss down | falling, embarrassment |
| `thud` | low body thump | landing, impact |
| `honk` | filtered sawtooth | car, clown |
| `bubble` | short rising blips | underwater, potion |
| `sparkle` | clustered shimmering bells | fairy dust, reward |
| `swoosh` | reversed whoosh | enter frame |
| `riser` | accelerating sweep | build to reveal |

`--wide` adds Haas stereo widening. `click`, `pop` and `bell` default to
`output/sfx/ui/`; force `--force-sfx` to keep them in `sfx/cartoon/`.

## Ambient beds

```powershell
& $PY "$TK\generate.py" ambient --type rain --dur 30 --loop
& $PY "$TK\generate.py" ambient --type forest --dur 60 --loop
& $PY "$TK\generate.py" ambient --type magic --dur 20
```

Available: `rain wind ocean birds crickets forest night stream room magic`.

`room` is near-silent brown noise — use it to glue cuts so the soundtrack never
drops to digital silence. `magic` is a slow, detuned pad for storybook moments.
Always pass `--loop` for beds that repeat under a long scene.

Beds are delivered at -24 LUFS (not the -16 used for music) because they sit
under dialogue; `room` is -45 LUFS so it stays inaudible until you turn it up.
Override with `--lufs`. The procedural chiptune engine snaps music to whole bars,
so `--dur 20` may return 17-22 s — trim or loop it if the exact length matters.

## Transitions

```powershell
& $PY "$TK\generate.py" transition --type riser
& $PY "$TK\generate.py" transition --type whoosh
& $PY "$TK\generate.py" transition --type downlifter
```

`riser downlifter whoosh sting swell glitch bubble`.

Use `riser` into a reveal, `downlifter` out of one, `whoosh` for a hard cut,
`sting` for a title card, and `glitch` only for tech/countdown moments.

## Rules of thumb for children's video

- One effect per action, not per frame. Silence between effects is what makes them read.
- Keep SFX 6 dB under narration; see **audio-mixing** for ducking.
- Cartoon SFX should be shorter than the action they describe (0.2-0.7 s).
- Layer at most two: a bright transient (`pop`, `click`) plus a body (`thud`, `splat`).
- Do not reuse the exact same file twice within 3 s — reroll `--seed`.

## When synthesis is not enough

Ker-plop, dog bark, door slam, real footsteps — these need recordings or a model:

1. **Record it yourself** (phone, 48 kHz). Best quality per minute of effort, and free.
2. **CC0 libraries**: Freesound filtered to CC0, Kenney Audio, Pixabay (check each
   cover's licence). Download manually and drop files into `output/voice/` or a
   project folder, then process with **audio-editing**.
3. **Open-weights audio models** — Stable Audio Open or AudioLDM2, if VRAM allows.
   See `audio-toolkit/references/external-engines.md`. MusicGen is a music model:
   it makes poor one-shot effects.

Never use a paid credit-based generator; the whole pipeline here stays free.

## Hand-off

- Trim dead air off a recording → **audio-editing**
- Place effects against music and voice → **audio-mixing**
- Convert, concat, or embed into video → **ffmpeg-audio**
