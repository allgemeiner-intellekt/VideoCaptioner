---
name: video-transcribe
description: Download public online videos or accept local video/audio files, create one managed job folder per video under this workspace, and use VideoCaptioner to produce raw.srt and raw.txt transcription artifacts with logs and a manifest. Use when the user asks to download a video, transcribe a video/audio file, obtain subtitles, or prepare raw transcript materials for later cleanup.
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
  raw.srt
  raw.txt
  manifest.json
  logs/
    download.log
    transcribe-srt.log
    transcribe-txt.log
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
6. Generate both `raw.srt` and `raw.txt`.
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
3. Run `ffprobe`, then transcribe to `raw.srt` and `raw.txt`.
4. Write `manifest.json`.

Use the original file directly only when copying would be wasteful and the user has not asked for a self-contained job folder. In that case, record the external path in `manifest.json`.

## Validation

After transcription:

```bash
test -s "$JOB/raw.srt"
test -s "$JOB/raw.txt"
wc -l "$JOB/raw.srt" "$JOB/raw.txt"
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
  "asr": "bijian",
  "created_at": "YYYY-MM-DD",
  "notes": []
}
```

Use `source_type: "file"` for local media. Include `external_media_file` if the job uses a file outside the job folder.

## Handoff

End with the job folder path and the generated artifacts. Mention that `raw.srt` and `raw.txt` are raw transcription outputs ready for the transcript cleanup skill.
