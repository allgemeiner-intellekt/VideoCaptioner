---
name: subtitle-to-text
description: Convert raw subtitles or transcript text from a video job into one polished transcript.md using AI reading and judgment, especially for Chinese lectures, talks, philosophy videos, and other spoken long-form content. Supports both single raw.srt/raw.txt jobs and split long-video jobs with parts/raw.part-*.srt or parts/raw.part-*.txt. Use when the user asks to clean up subtitles, turn SRT into readable text, organize a transcript, or produce a structured article-like transcript.
---

# Subtitle To Text

Use this skill to turn raw subtitles into one readable transcript. This is an AI reading task, not a mechanical cleanup task: read the subtitle content, understand the argument, then write a faithful structured text.

## Output

Produce exactly one main artifact unless the user asks otherwise:

```text
videos/jobs/<slug>/transcript.md
```

If the input is outside a job folder, write `transcript.md` beside the source file or in the user-specified destination.

## Default Style

Use the structured article transcript style:

- Keep the speaker's original claims, order of argument, tone, and emphases.
- Add clear `##` section headings based on the video's actual argumentative structure.
- Rebuild paragraphs so the text reads like a lecture article or study note.
- Remove subtitle numbers, timestamps, isolated filler words, obvious stutters, and meaningless repetitions.
- Preserve forceful language, jokes, rhetorical questions, and distinctive phrasing when they carry meaning.
- Do not summarize instead of transcribing. The output should still represent the full talk.
- Do not add outside arguments, explanations, citations, or interpretations that are not in the source.

## Reading Workflow

1. Locate the source, usually `raw.srt`, `raw.txt`, `parts/raw.part-*.txt`, `parts/raw.part-*.srt`, or a user-provided `.srt`.
2. Inspect the file size and length. For long files, read it in chunks.
3. If the job is split, read the part files in filename order: `raw.part-001`, `raw.part-002`, and so on.
4. Actually read the subtitle content. Do not generate the transcript by regex, script, or bulk text transformation.
5. Identify the main turns in the argument across the whole video, not separately per part.
6. Write `transcript.md` as one coherent Markdown document.
7. Re-read the beginning, middle, and end of the output against the source to check that the structure and ending are complete.

Shell commands may be used to view chunks, count lines, or inspect source locations. They must not be used to mechanically produce the final prose.

## Split Jobs

For long videos, the transcription skill may leave separate part files instead of merged subtitles:

```text
videos/jobs/<slug>/
  parts/
    raw.part-001.txt
    raw.part-002.txt
    raw.part-003.txt
```

Treat these as one continuous source:

- Read all parts in numeric filename order.
- Do not create separate final transcripts per part unless the user asks.
- Do not preserve artificial part boundaries as headings unless they match real topic boundaries.
- Smooth transitions across part boundaries when writing `transcript.md`.
- If overlap causes duplicated sentences at boundaries, remove the duplicate in the final text.
- If a part is missing or empty, stop and report the missing path instead of silently skipping it.

## SRT Handling

When reading SRT:

- Ignore numeric subtitle indices and time ranges while writing the final text.
- Merge short subtitle fragments into full sentences and paragraphs.
- Treat long pauses as possible paragraph or section boundaries, not automatic cuts.
- Use the surrounding context to decide whether a short subtitle is meaningful or just a filler.

## Editing Rules

Favor faithful readability:

- Add natural Chinese punctuation.
- Normalize repeated words only when they are accidental disfluency.
- Keep repeated words when they express emphasis, rhythm, anger, irony, or rhetorical escalation.
- Lightly repair grammar caused by speech-to-text fragmentation.
- Keep important oral markers such as "问题在于", "也就是说", "为什么", "所以", when they organize reasoning.

For domain terms, be conservative:

- Correct obvious ASR errors when the context is strong.
- For uncertain corrections, write a light inline marker such as `〔疑似：癔症〕`.
- Do not silently replace philosophical, political, technical, or foreign-language terms unless the context makes the correction clear.
- Preserve useful foreign terms when the speaker appears to say or discuss them.

## Sectioning

Choose headings from the content itself. Good headings name argumentative moves, for example:

```markdown
## 一、问题的提出：如何理解这句话
## 二、第一种错误：把社会理解为人的总和
## 三、第二种错误：经院式地保留人的本质
## 四、回到原句：人的本质作为社会关系的效果
```

Do not over-outline. Use enough headings to make the talk navigable, but keep the transcript continuous.

## Quality Bar

Before finishing, check:

- The output has one top-level title and coherent `##` sections.
- The beginning and ending of the source are represented.
- No major section of the talk has been reduced to a brief summary.
- Strong or controversial claims remain attributable to the speaker's transcript, not rewritten as your own essay.
- Uncertain ASR repairs are marked instead of hidden.

End by reporting the path to `transcript.md` and any important uncertainty about the source quality.
