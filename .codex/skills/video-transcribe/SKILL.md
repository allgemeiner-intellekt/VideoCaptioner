---
name: video-transcribe
description: Download public online videos or accept local video/audio files, create one managed job folder per video under this workspace, and use VideoCaptioner to produce raw subtitle/text transcription artifacts with logs and a manifest. For media longer than 45 minutes, split into parts before transcription to control memory pressure and retry cost. Use when the user asks to download a video, transcribe a video/audio file, obtain subtitles, or prepare raw transcript materials for later cleanup.
allowed-tools: Bash(videocaptioner *, ffprobe *, ffmpeg *, mkdir *, cp *, find *, test *, wc *, date *, printf *, python3 *)
---

# Video Transcribe

Use this skill to turn a public video URL or local media file into raw transcript artifacts. Keep the scope narrow: create the job folder, obtain media, transcribe it, validate outputs, and write metadata. Do not polish, summarize, translate, or restructure the transcript here; that belongs to the later subtitle/text cleanup skill.

## Workspace

Assume the current project is the video service workspace:

```text
videos/
  jobs/
  incoming/
  exports/
  tmp/
VideoCaptionerApp/
```

Create one folder per video under `videos/jobs/`. Use a short slug from the user, URL title, or filename. Prefix with the date when useful:

```text
videos/jobs/2026-05-24-short-slug/
  source.url              # URL jobs only
  source.<ext>            # local jobs or normalized downloaded media when practical
  media-info.txt
  raw.srt                 # short jobs only
  raw.txt                 # short jobs only
  parts/                  # long jobs only
    part-001.<ext>
    raw.part-001.srt
    raw.part-001.txt
  manifest.json
  logs/
    download.log
    transcribe-srt.log
    transcribe-txt.log
    split.log
    transcribe-part-001-srt.log
    transcribe-part-001-txt.log
  tmp/
```

If a target job folder already exists, do not overwrite it. Add a numeric suffix such as `-2`.

## Command Selection

Prefer the globally installed `videocaptioner` command.

Before running a command family for the first time in a session, check help:

```bash
videocaptioner download --help
videocaptioner transcribe --help
```

If `videocaptioner` is not on `PATH`, fall back to the local project copy:

```bash
cd VideoCaptionerApp
uv run videocaptioner ...
```

Default ASR is `bijian` for public Chinese/English videos. Use another ASR only if the user asks, the language requires it, or `bijian` fails.

## URL Workflow

1. Create the job folder with `logs/` and `tmp/`.
2. Save the original URL to `source.url`.
3. Run `videocaptioner download "$URL" -o "$JOB"` and tee output to `logs/download.log`.
4. Find the downloaded media file in the job folder. Prefer common video/audio extensions.
5. Run `ffprobe` and write `media-info.txt`.
6. If media duration is over 45 minutes, use the long media workflow. Otherwise generate both `raw.srt` and `raw.txt`.
7. Write `manifest.json`.
8. Report the important output paths.

Typical commands:

```bash
mkdir -p "$JOB/logs" "$JOB/tmp"
printf '%s\n' "$URL" > "$JOB/source.url"
videocaptioner download "$URL" -o "$JOB" 2>&1 | tee "$JOB/logs/download.log"
ffprobe -hide_banner "$MEDIA" 2>&1 | tee "$JOB/media-info.txt"
videocaptioner transcribe "$MEDIA" --asr bijian --format srt -o "$JOB/raw.srt" 2>&1 | tee "$JOB/logs/transcribe-srt.log"
videocaptioner transcribe "$MEDIA" --asr bijian --format txt -o "$JOB/raw.txt" 2>&1 | tee "$JOB/logs/transcribe-txt.log"
```

## Local Media Workflow

For a local video/audio path:

1. Create the job folder.
2. Copy the file into the job folder as `source.<ext>` unless the file is already there.
3. Run `ffprobe`.
4. If media duration is over 45 minutes, use the long media workflow. Otherwise transcribe to `raw.srt` and `raw.txt`.
5. Write `manifest.json`.

Use the original file directly only when copying would be wasteful and the user has not asked for a self-contained job folder. In that case, record the external path in `manifest.json`.

## Long Media Workflow

If `ffprobe` shows duration over 45 minutes, split before transcription. This is the default for long media because it keeps memory pressure, temporary files, and failure retries bounded.

Use `parts/` for all split media and per-part transcripts:

```text
parts/
  part-001.<ext>
  raw.part-001.srt
  raw.part-001.txt
  part-002.<ext>
  raw.part-002.srt
  raw.part-002.txt
```

Rules:

- Prefer 25-35 minute parts; 30 minutes is the default target.
- Avoid a final tiny part when practical. For example, split a 50 minute video into about 25+25 rather than 30+20 only if that is easy; do not over-engineer this.
- A 5-10 second overlap is acceptable if needed for speech continuity, but do not spend effort merging duplicate text later.
- Do not merge part subtitles into `raw.srt` or `raw.txt` by default. The cleanup skill can read part files in order.
- If the user explicitly asks for no splitting, follow the user's instruction and note the increased resource risk.

Typical split approach:

```bash
mkdir -p "$JOB/parts"
ffmpeg -hide_banner -i "$MEDIA" -vn -ac 1 -ar 16000 -f segment -segment_time 1800 -reset_timestamps 1 "$JOB/parts/part-%03d.wav" 2>&1 | tee "$JOB/logs/split.log"
```

Then transcribe each part independently:

```bash
videocaptioner transcribe "$PART" --asr bijian --format srt -o "$JOB/parts/raw.part-001.srt" 2>&1 | tee "$JOB/logs/transcribe-part-001-srt.log"
videocaptioner transcribe "$PART" --asr bijian --format txt -o "$JOB/parts/raw.part-001.txt" 2>&1 | tee "$JOB/logs/transcribe-part-001-txt.log"
```

Use the actual part number in each output path. If one part fails, retry only that part after inspecting its log.

For long media, the successful output is the ordered set of `parts/raw.part-*.srt` and `parts/raw.part-*.txt`, not a merged `raw.srt`/`raw.txt`.

## Validation

After transcription:

```bash
test -s "$JOB/raw.srt" || test -s "$JOB/parts/raw.part-001.srt"
test -s "$JOB/raw.txt" || test -s "$JOB/parts/raw.part-001.txt"
wc -l "$JOB"/raw.* "$JOB"/parts/raw.part-* 2>/dev/null
```

Treat empty or extremely short output as a failed transcription unless the media itself is very short. Check `logs/transcribe-*.log` before retrying. If online download fails, tell the user the exact failure and suggest placing the video in `videos/incoming/`.

## Manifest

Write `manifest.json` with at least:

```json
{
  "status": "transcribed",
  "source_type": "url",
  "source_url": "",
  "media_file": "",
  "raw_srt": "raw.srt",
  "raw_txt": "raw.txt",
  "split": false,
  "parts": [],
  "asr": "bijian",
  "created_at": "YYYY-MM-DD",
  "notes": []
}
```

Use `source_type: "file"` for local media. Include `external_media_file` if the job uses a file outside the job folder.

For split jobs, set `split: true`, omit or null `raw_srt`/`raw_txt`, and list parts in order:

```json
{
  "split": true,
  "split_threshold_minutes": 45,
  "part_target_minutes": 30,
  "parts": [
    {
      "media_file": "parts/part-001.wav",
      "raw_srt": "parts/raw.part-001.srt",
      "raw_txt": "parts/raw.part-001.txt"
    }
  ]
}
```

## Handoff

End with the job folder path and the generated artifacts. For short jobs, mention `raw.srt` and `raw.txt`. For long split jobs, mention `parts/raw.part-*.srt` and `parts/raw.part-*.txt`. These raw transcription outputs are ready for the transcript cleanup skill.
