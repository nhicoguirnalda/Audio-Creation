---
name: music-generation
description: Generates background music locally with MusicGen Small, or procedurally when no GPU is available. Use for children's background music, cartoon underscore, lullabies, playful learning-video beds, jingles, and loops. Trigger on "make background music", "generate a music track", "kids bgm", "cartoon music", "lullaby", "loop this bed". Produces WAV + MP3 into output/music/childrens/. Do NOT use for sound effects (sound-effects-generation) or for trimming/mixing existing audio (audio-editing, audio-mixing).
---

# Music Generation

Local, free, offline music generation. Primary engine is **MusicGen Small**
(`facebook/musicgen-small`, MIT-licensed weights, 300M params) through
Hugging Face `transformers`. Fallback engine is a built-in chiptune synthesiser
that needs no model at all.

## Setup

```powershell
$PY = "D:\Downloads\OpenCode Skills\.venv\Scripts\python.exe"
$TK = "D:\Downloads\OpenCode Skills\audio-toolkit"
```

Check the stack once per session — never install anything before this passes:

```powershell
& $PY "$TK\generate.py" status
```

Expected on this machine: `cuda True`, `gpu NVIDIA GeForce GTX 1650`, `vram_gb 4.3`.
If it reports `torch None`, the venv is missing — run
`& "$TK\setup.cmd"` (bypasses PowerShell's script policy) before generating.

## Generate

```powershell
& $PY "$TK\generate.py" music --preset playful --dur 20
& $PY "$TK\generate.py" music --preset bedtime --dur 30 --loop
& $PY "$TK\generate.py" music --preset silly --dur 15 --seed 7
& $PY "$TK\generate.py" music --prompt "toy piano and glockenspiel, 100 bpm, gentle" --dur 20
```

| Flag | Meaning |
|---|---|
| `--preset` | one of `playful bedtime adventure silly magical learning lullaby celebration` |
| `--prompt` | extra description appended to the preset prompt |
| `--dur` | seconds (default 20) |
| `--engine` | `auto` (default) / `musicgen` / `procedural` |
| `--seed` | reproducible take; reroll by changing it |
| `--loop` | crossfade the tail into the head for a seamless loop |
| `--chunks` | required for `--dur` over 30 s (see below) |
| `--formats` | `wav,mp3` (default), `wav`, or `mp3` |
| `--guidance` | prompt adherence, default 3.0; **1.0 lowers VRAM a lot** |

Output lands in `output/music/childrens/<name>_<timestamp>.wav|.mp3`, loudness-normalised
to -16 LUFS, and every file is appended to `output/_manifest.jsonl`.

## The 30-second ceiling

MusicGen produces 50 tokens per second, so a single pass stops at ~30 s.
For longer beds pass `--chunks`; the toolkit generates consecutive chunks and
crossfades them over 2 s. Expect drift in melody — for anything over ~60 s,
generate one loop with `--loop` and repeat it in the edit instead. That sounds
better and is faster.

Measured on this machine: about 5x realtime once the model is warm — 8 s of
music in ~25 s, a 30 s bed in about 2.5 minutes. The first run after a reboot is
slower (CUDA warm-up). The procedural engine takes about one second for the same
length.

## 4 GB VRAM rules

The GTX 1650 has 4 GB and Windows already holds ~1 GB.

- Keep `--dur` at or under 30 s per pass.
- On out-of-memory: add `--guidance 1.0` (skips classifier-free guidance, roughly
  halves activation memory) and close other GPU apps.
- Set `$env:PYTORCH_CUDA_ALLOC_CONF="expandable_segments:True"` if fragmentation errors appear.
- On CPU (`cuda False`) MusicGen still works but is ~20-40x slower: use
  `--engine procedural` for anything iterative, MusicGen only for the final bed.

## When to skip the model

Use `--engine procedural`. It is instant, deterministic, and writes a proper
stereo loop (chord pad + bass + melody + percussion) in the mood of the preset.
For 15-second children's clips this is often indistinguishable from a model take
and costs nothing. Prefer it while blocking out a video, then replace the final
bed with MusicGen.

## Writing prompts that work

MusicGen responds to instrumentation + tempo + mood + "no vocals". It ignores
negations poorly, so describe what you *want*, not what you don't.

```
cheerful children's background music, bright xylophone and ukulele,
hand claps, bouncy 120 bpm, major key, clean mix, no vocals
```

Keep underscores quiet: aim for gentle dynamics, avoid dense drums under
narration. For a series, reuse one `--seed` and only vary `--dur` so cues feel
related. Full recipe list: `references/prompt-recipes.md`.

## External engines

When local generation is impractical (VRAM exhausted, need >2 min, need
non-music audio), see `audio-toolkit/references/external-engines.md` — all
options there are free, open-weights, and self-hosted. No paid APIs, no credits.

## Hand-off

- Loop or trim the result → **audio-editing**
- Put it under narration with ducking → **audio-mixing**
- Raw FFmpeg conversions, loudness standards, muxing into video → **ffmpeg-audio**
