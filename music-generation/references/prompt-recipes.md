# MusicGen prompt recipes for children's content

MusicGen Small reads a plain-English description. It is not instruction-following:
every extra clause dilutes the ones before it. Four elements, in this order, work
best:

```
[instrumentation], [tempo + key feel], [mood / use case], [mix note], no vocals
```

Keep prompts under ~20 words. `no vocals` matters — the model happily sings.

## Ready-made prompts (used by `--preset`)

| Preset | Prompt |
|---|---|
| `playful` | cheerful children's background music, bright xylophone, ukulele and hand claps, bouncy 120 bpm, major key, clean mix, no vocals |
| `bedtime` | gentle bedtime lullaby, soft music box with warm pads, slow 70 bpm, dreamy, very calm, no vocals |
| `adventure` | playful adventure cartoon score, pizzicato strings, light percussion, 130 bpm, heroic but cute, no vocals |
| `silly` | goofy cartoon chase music, tuba, slide whistle and kazoo, comedic, fast tempo, no vocals |
| `magical` | magical fairy tale music, celesta, harp and shimmering bells, dreamy, no vocals |
| `learning` | friendly kids learning-video music, soft marimba and claps, steady 110 bpm, simple and unobtrusive, no vocals |
| `lullaby` | very slow bedtime lullaby, music box melody, warm and quiet, no vocals |
| `celebration` | happy kids celebration music, ukulele, whistles and hand claps, upbeat, no vocals |

## Scene-specific additions

Pass these as `--prompt "..."` on top of a preset.

**Opening / logo stinger**
```
short and punchy, ends with a clean button, 8 seconds
```

**Counting / learning segment**
```
simple repeated motif, sparse arrangement so numbers stay audible, no drums under 200 Hz
```

**Building a tower / stacking**
```
steadily rising pitch, one new instrument added every bar, playful
```

**Oops / mistake moment**
```
comic trombone wah, descending, brief, playful not sad
```

**Resolution / moral of the story**
```
warm and reassuring, strings and piano, resolves to the tonic, gentle
```

**Outro / subscribe card**
```
bright, friendly, one bar hook, ends cleanly with no reverb tail
```

**Background under dialogue**
```
minimal, mid-frequency space left free for narration, no melody in the 300-3000 Hz band
```

That last one is the most useful prompt in the list: it is the difference
between music that supports a voiceover and music that fights it.

## Parameter guides

| Want | Change |
|---|---|
| Closer to the prompt | `--guidance 4-6` (more VRAM, more literal) |
| More surprising melodies | `--temp 1.2-1.5` |
| Safer, more generic | `--temp 0.7-0.9` |
| Same family of results | reuse one `--seed`, vary `--dur` |
| Different take | change `--seed` by 1 |

`--guidance 1.0` disables classifier-free guidance — less literal but roughly
half the memory. Use it when the 4 GB card is struggling.

## Instrument vocabulary that works

Warm and child-friendly: `xylophone, glockenspiel, celesta, music box, marimba,
ukulele, pizzicato strings, hand claps, whistles, toy piano, soft bells`.

Avoid in prompts: `epic, cinematic, dubstep, EDM drop, heavy bass, distorted`.
They pull the model toward adult trailer music, which is exactly wrong here.

## Loop-friendly generation

Generate 15-25 s, then:

```powershell
& $PY "$TK\generate.py" music --preset learning --dur 20 --loop --name learn-bed
```

`--loop` crossfades the tail over the head so the seam is inaudible. MusicGen
does not produce exact bar lengths, so a crossfade is more reliable than trying
to cut on a beat.

## Quality triage

Listen for these before shipping a take:

- A melody that resolves early and then wanders → regenerate with a shorter `--dur`.
- Sudden instrument changes at ~15 s → that is the chunk boundary; use one pass or `--chunks`.
- Pumping or clipping → lower `--guidance`, then normalise with **audio-editing**.
- Vocals appearing despite `no vocals` → add `instrumental` and regenerate.
