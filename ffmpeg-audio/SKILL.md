---
name: ffmpeg-audio
description: Copy-paste FFmpeg recipes for audio - probe files, convert WAV/MP3/AAC/Opus, two-pass loudness normalisation, limiters, concat and crossfade, extract or replace audio in video, batch processing, metadata and cover art. Use for "ffmpeg command for", "convert to mp3", "extract audio from mp4", "replace the audio track", "normalise loudness properly", "why does my audio clip", "mux audio into video". This is the low-level layer; prefer audio-editing and audio-mixing for routine work.
---

# FFmpeg Audio

The raw tool layer. The other four skills call FFmpeg through
`audio-toolkit/`, but when a recipe does not exist yet, write the command here.

This machine has **FFmpeg 9.0.2 full build** (gyan.dev): `libmp3lame`,
`libopus`, `aac`, `librubberband`, `libsoxr`, `loudnorm`, `ebur128` are all
available. Nothing needs installing.

```powershell
$FF = "ffmpeg"
ffmpeg -version | Select-Object -First 1
```

## Probe before touching

```powershell
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bit_rate `
        -show_entries format=duration,size -of default=noprint_wrappers=1 "in.mp3"
```

Or `& $PY "$TK\process.py" info --in in.mp3` for a plain-language summary.

## Format and codec decisions

| Goal | Command |
|---|---|
| Master / intermediate | `-c:a pcm_s16le -ar 44100` (WAV) |
| Music and SFX delivery | `-c:a libmp3lame -b:a 192k` (VBR `-q:a 2` is smaller) |
| Video mux | `-c:a aac -b:a 192k -ar 48000` |
| Web / game loop | `-c:a libopus -b:a 96k -ar 48000` |
| Phone ringtone-ish tiny | `-c:a libmp3lame -b:a 96k -ar 32000` |

Rules: **WAV 44.1 kHz** for music, **AAC 48 kHz** for anything going into MP4,
**MP3 192 kbps** for distribution. Keep WAV until the last step — every MP3
encode is lossy and lossy-to-lossy re-encoding compounds artefacts.

```powershell
ffmpeg -i in.wav -c:a libmp3lame -b:a 192k out.mp3
ffmpeg -i in.wav -ac 1 -ar 44100 mono.wav
ffmpeg -i in.wav -af "aformat=channel_layouts=stereo" stereo.wav
```

## Loudness, correctly

One-pass `loudnorm` is fine for short clips. For anything delivered, use two
passes — measure, then apply the measurement.

```powershell
# pass 1: measure
ffmpeg -i in.wav -af loudnorm=I=-16:TP=-1.5:LRA=11:print_format=json -f null -

# pass 2: apply the numbers from pass 1
ffmpeg -i in.wav -af "loudnorm=I=-16:TP=-1.5:LRA=11:measured_I=-17.2:measured_TP=-1.1:measured_LRA=6.4:measured_thresh=-28.0:offset=0.3:linear=true" `
       -ar 44100 out.wav
```

Targets: **-14 LUFS** web video, **-16 LUFS** podcast/game, **-23 LUFS** broadcast.
Always cap true peak at -1 dBTP (or -1.5 for delivery with lossy encoding).

Catch overs after any processing:

```powershell
ffmpeg -i in.wav -af "alimiter=limit=0.891:level=false" -ar 44100 out.wav
```

`0.891` is -1 dBFS. `level=false` prevents the limiter from auto-boosting.

Verify:

```powershell
ffmpeg -i out.wav -filter_complex ebur128=peak=true -f null - 2>&1 | Select-String "I:|Peak:"
```

## Fades and crossfades

```powershell
# fade in 0.5 s, fade out 2 s starting at 18 s
ffmpeg -i in.wav -af "afade=t=in:st=0:d=0.5,afade=t=out:st=18:d=2" out.wav

# equal-power crossfade between two cues, 1.5 s overlap
ffmpeg -i a.wav -i b.wav -filter_complex `
  "[0:a][1:a]acrossfade=d=1.5:c1=tri:c2=tri[out]" -map "[out]" mix.wav
```

`acrossfade` needs both inputs the same sample rate and channel layout — insert
`aformat=sample_rates=44100:channel_layouts=stereo` on each first.

## Concat

Same codec parameters → stream copy, instant and lossless:

```powershell
"file 'a.wav'" | Set-Content list.txt
"file 'b.wav'" | Add-Content list.txt
ffmpeg -f concat -safe 0 -i list.txt -c copy joined.wav
```

Different codecs, rates or layouts → decode and re-encode:

```powershell
ffmpeg -i a.mp3 -i b.wav -filter_complex "[0:a][1:a]concat=n=2:v=0:a=1[out]" -map "[out]" joined.wav
```

Stream copy across mismatched formats produces silence or garbage. When in
doubt, use the filter form.

## Audio in video

```powershell
# extract the audio unchanged
ffmpeg -i clip.mp4 -vn -c:a copy audio.m4a

# extract to WAV for editing
ffmpeg -i clip.mp4 -vn -ar 48000 -ac 2 audio.wav

# replace the audio track, keep video untouched
ffmpeg -i clip.mp4 -i mix.wav -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -b:a 192k -shortest out.mp4

# add music under the existing audio (keeps both)
ffmpeg -i clip.mp4 -i bgm.wav -filter_complex `
  "[0:a]volume=1.0[vo];[1:a]volume=0.25[bed];[vo][bed]amix=inputs=2:normalize=0[a]" `
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k -shortest out.mp4

# mute the video entirely
ffmpeg -i clip.mp4 -c:v copy -an out.mp4
```

For MP4 always output **AAC at 48 kHz**. MP3-in-MP4 plays everywhere in browsers
but breaks in some players, and the extra encoder delay makes sync drift.

## Trim and re-time without breaking sync

```powershell
# trim 1.0 s .. 4.5 s, re-encode audio
ffmpeg -i in.wav -ss 1.0 -to 4.5 -c:a pcm_s16le out.wav

# trim video + audio together, frame accurate
ffmpeg -ss 1.0 -to 4.5 -i clip.mp4 -c:v libx264 -c:a aac -b:a 192k out.mp4
```

`-ss` **before** `-i` is fast seeking; **after** `-i` is accurate but slow.
For audio-only work either is fine. For video, put it before `-i` and accept
re-encoding, or use the `trim` filter when frame accuracy is mandatory.

## Silence

```powershell
# find silent sections (reports start/duration in seconds)
ffmpeg -i vo.wav -af silencedetect=noise=-40dB:d=0.4 -f null - 2>&1 | Select-String "silence_"

# trim leading silence only
ffmpeg -i vo.wav -af "silenceremove=start_periods=1:start_threshold=-45dB" out.wav
```

## Batch

```powershell
Get-ChildItem *.wav | ForEach-Object {
  ffmpeg -hide_banner -loglevel error -i $_.FullName -c:a libmp3lame -b:a 192k `
         ($_.FullName -replace '\.wav$', '.mp3')
}
```

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Output is silent after `-c copy` | mismatch between inputs | drop `-c copy`, use the `concat` filter |
| `Filtergraph ... has an unconnected output` | missing `-map "[label]"` | map every labelled output |
| Audio drifts from video | MP3 encoder delay in MP4 | use AAC 48 kHz |
| `Output file #0 does not contain any stream` | `-vn` with no audio input | check the source actually has audio |
| Loudness measured but quieter after apply | `linear=true` refused out of range | let it run dynamic (drop `linear`) |
| Clipping after `amix` | `normalize` default behaviour changed across versions | set `normalize=0` and manage gain yourself |
| `Unknown filter 'rubberband'` | minimal FFmpeg build | this build has it; another machine may need `asetrate+aresample+atempo` |

## Hand-off

- You want operations without writing commands → **audio-editing**, **audio-mixing**
- You still need material → **music-generation**, **sound-effects-generation**
