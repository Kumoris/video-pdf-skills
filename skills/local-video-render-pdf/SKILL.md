---
name: local-video-render-pdf
description: Generate a professional, detailed, figure-rich LaTeX course note and final PDF from a local video file or local folder of video assets. Use when the user provides a local video path such as .mp4, .mov, .mkv, .webm, .m4v, .avi, or asks Codex to understand and process a local lecture, tutorial, screen recording, interview, meeting recording, or technical talk by combining local metadata, sidecar subtitles or transcripts, optional slides or images, audio transcription, direct visual frame inspection, key-frame extraction, chaptering, and a rendered PDF deliverable.
---

# Local Video Render PDF

Use this skill to understand a local video and turn it into a complete, compileable `.tex` note and a rendered PDF.

This skill is the local-file counterpart of the online video render workflows. It does not assume a platform URL, official cover image, platform chapters, or downloadable subtitle metadata. Build the source bundle from local files, local media streams, and direct visual inspection.

## Goal

Produce a professional Chinese lecture note from a local video file.

The output must:

- use the video's actual teaching content rather than transcript text alone
- combine speech, on-screen visuals, slides, code, demos, UI states, diagrams, formulas, and any provided sidecar files
- include high-value key frames or crops as figures, without adding redundant screenshots
- use a user-provided cover image when available; otherwise select a visually inspected representative frame and label it honestly as a representative frame
- end with a final synthesis section that includes the speaker's substantive closing discussion when present and your own distilled takeaways
- be structurally organized with `\section{...}` and `\subsection{...}`
- be a complete `.tex` document from `\documentclass` to `\end{document}`
- be compiled successfully to PDF as part of the final delivery unless the user explicitly requests a non-PDF output

If the user asks for another local-video processing output, such as a summary, chapter list, transcript cleanup, key-frame package, clip plan, or extraction plan, use the same source-understanding workflow but deliver the requested artifact instead of forcing a full PDF.

## Input Handling

Accept any of the following:

- a local video file path, such as `.mp4`, `.mov`, `.mkv`, `.webm`, `.m4v`, `.avi`
- a local directory containing one or more videos plus sidecar files
- a video file with matching sidecars, such as `.srt`, `.vtt`, `.txt`, `.md`, `.pdf`, `.pptx`, `.png`, `.jpg`
- a screen recording, meeting recording, interview, podcast with slides, coding demo, or lecture capture

Do not overwrite the source video or sidecar files.

Create a work directory in the current workspace or beside the requested output, for example:

```bash
mkdir -p "<video-stem>_video_notes_work"/{metadata,subtitles,audio,frames,figures,tex}
```

If the user gives a directory with several plausible videos, inspect filenames and durations first. Ask the user which one to process only when there is no clear intended target.

## Local Source Acquisition

### Metadata Inspection

Inspect local metadata before writing.

Use `ffprobe` or `ffmpeg` to record:

- container format
- duration
- resolution
- frame rate
- audio streams and languages when available
- subtitle streams when available
- creation time or title metadata when available

Example:

```bash
ffprobe -hide_banner -show_format -show_streams -print_format json "<video-path>" > "<work>/metadata/ffprobe.json"
```

If metadata is sparse, derive a working title from the filename and record the local path in the `.tex` metadata. Do not invent a platform channel, publication date, or official cover.

### Sidecar Discovery

Look for files near the video before using transcription.

Prefer matching-stem files first:

- subtitles: `<stem>.srt`, `<stem>.vtt`, `<stem>.ass`
- transcripts or notes: `<stem>.txt`, `<stem>.md`
- slides or handouts: `<stem>.pdf`, `<stem>.pptx`
- cover or thumbnail: `<stem>_cover.*`, `cover.*`, `thumbnail.*`

Use sidecar material as evidence, but do not let it replace video understanding. Important claims and figures should still be aligned with video timestamps or visually confirmed frames.

### Subtitle Acquisition

Use this fallback order:

1. **Sidecar subtitles**: use local `.srt` or `.vtt` if present.
2. **Embedded subtitles**: extract subtitle streams from the video container.
3. **Sidecar transcript**: use local `.txt` or `.md`, then align coarsely to video time using chapters, scene changes, or audio when possible.
4. **Whisper speech-to-text**: extract audio and transcribe to timestamped SRT.
5. **Visual-only mode**: if audio is absent, unusable, or transcription is unavailable, build the notes from visual inspection and any sidecar files.

Embedded subtitle extraction example:

```bash
ffmpeg -i "<video-path>" -map 0:s:0 "<work>/subtitles/embedded.srt"
```

Whisper fallback example:

```bash
ffmpeg -i "<video-path>" -map 0:a:0 -vn -ac 1 -ar 16000 "<work>/audio/audio.wav"
whisper "<work>/audio/audio.wav" --model medium --language zh --output_format srt --output_dir "<work>/subtitles"
```

Choose the Whisper language from the user's request or the video's audio. If the language is uncertain, use auto-detection or run a short sample first.

Preserve timestamps. Do not flatten subtitles into plain text before locating figures.

### Cover or Representative Frame

Local videos often do not have an official cover.

Use this order:

1. user-provided cover image
2. sidecar `cover.*` or `thumbnail.*`
3. embedded attached picture stream, if present
4. visually inspected title slide or representative frame extracted from the video

When using an extracted frame, call it a representative frame rather than an original cover.

Example:

```bash
ffmpeg -ss 00:00:10 -i "<video-path>" -frames:v 1 -q:v 2 "<work>/figures/representative_frame.jpg"
```

## Video Understanding Workflow

Build a source map before drafting:

- timeline boundaries from subtitles, visible slide changes, scene changes, or topic changes
- core concepts, definitions, formulas, diagrams, code, results, demos, and examples
- visually important segments that need key-frame extraction
- uncertain audio or visual regions that need reinspection
- supplied sidecar files and how they relate to the video

For screen recordings and software demos, track:

- user goal and setup state
- commands, code, UI actions, and resulting output
- important intermediate states and final state
- errors, corrections, or debugging branches that teach something

For interviews, meetings, and podcasts, track:

- speaker roles when inferable
- high-information exchanges worth preserving briefly
- claims, examples, caveats, disagreements, and closing synthesis

## Long Video Strategy

For longer videos, do not rely on a single monolithic pass.

- If the video is longer than 20 minutes, or the subtitle file contains more than 300 subtitle entries, split the work into smaller segments.
- Prefer natural boundaries: chapters, title slides, slide changes, scene changes, topic transitions, or subtitle ranges.
- Keep a small overlap between neighboring segments when explanations cross boundaries.
- When subagents are available and the user explicitly allows parallel agent work, assign segments to subagents and require each one to return: the segment's teaching goal, core claims, important formulas or code, required figures with time provenance, and ambiguities that need integration-time resolution.
- The main agent must integrate segment outputs into one unified outline and one coherent final narrative. The final PDF must read like a single lecture note, not a concatenation of chunk summaries.

## Teaching Content Rules

Build the notes from all of the following when available:

- video filename, local metadata, inferred or supplied title, and any chapter structure
- local subtitles, transcripts, slides, handouts, screenshots, or reference files
- on-screen diagrams, formulas, tables, plots, dashboards, architecture slides, and UI states
- speech explanations, examples, verbal emphasis, and speaker-provided caveats
- short high-signal original dialogue segments in interview, panel, podcast, or meeting videos, when exact wording adds presence, humor, intuition, or unusually compact information
- code snippets shown or described in the video

Skip content that does not contribute to the actual lesson:

- setup chatter
- greetings
- routine back-and-forth that does not add information, tension, humor, intuition, or teaching value
- recording logistics
- closing pleasantries

Keep the speaker's closing discussion when it carries actual teaching value, such as synthesis, limitations, future work, tradeoffs, advice, or open questions.

## Writing Rules

1. Write the notes in Chinese unless the user explicitly requests another language.

2. Organize the document with `\section{...}` and `\subsection{...}`.
   Reconstruct the teaching flow when needed; do not blindly mirror subtitle order.
   Each section should answer, in order when applicable: what problem is being solved, why simpler views are insufficient, what the core idea is, how it works, and what the reader should retain.

   Avoid overusing the "不是……而是……" sentence pattern.
   Use it only when the video itself establishes a real contrast and that contrast materially clarifies the mechanism.

   Do not use vague or overly abstract phrasing.
   Ground claims in concrete mechanisms, examples, variables, steps, observed phenomena, timestamps, figures, or speaker-provided evidence whenever possible.

3. Start from `assets/notes-template.tex`.
   Fill in the metadata block, including the local source path and either a cover image path or representative frame path, and replace the body content block with the generated notes.

4. Use figures whenever they materially improve explanation.
   Include as many figures as are necessary for teaching clarity, even if that means many figures across the document.
   Do not optimize for a small figure count; optimize for explanatory coverage and readability.
   Good figures are key formulas, diagrams, tables, plots, visual comparisons, pipeline schedules, architecture views, demo states, and stage-by-stage visual progressions.

5. Do not place images inside custom message boxes.

6. When a mathematical formula appears:
   first explain in plain Chinese what the formula is trying to express and why it appears
   show it in display math using `$$...$$`
   then immediately follow with a flat list that explains every symbol

7. When code examples appear:
   explain the role of the code before the listing and summarize the expected behavior after it when useful
   wrap them in `lstlisting`
   include a descriptive `caption`

8. Highlight teaching signals deliberately and repeatedly when the content justifies it:
   use `importantbox` for core concepts the reader must walk away with, including formal definitions, central claims, key mechanism summaries, theorem-like statements, critical algorithm steps, and compact restatements of the main idea after a dense explanation
   use `knowledgebox` for background and side knowledge that improves understanding without being the main thread, including prerequisite reminders, historical lineage, engineering context, design tradeoffs, terminology comparisons, and intuition-building analogies
   use `warningbox` for common misunderstandings and failure points, including notation overload, hidden assumptions, misleading heuristics, easy-to-make implementation mistakes, causal confusions, off-by-one style reasoning errors, and places where the speaker contrasts a wrong intuition with the correct one
   use `dialoguebox` only for conversation-heavy videos when a brief original dialogue segment is high-information, funny, vivid, or especially intuitive, and preserving the speaker's wording gives the reader a stronger sense of being present in the discussion
   keep `dialoguebox` snippets short: preserve speaker labels and a concrete timestamp or interval, lightly clean obvious ASR errors only when confident, and follow the box with prose that explains why the dialogue segment matters
   do not use `dialoguebox` for greetings, filler, long transcript dumps, or dialogue that would be clearer as ordinary summarized exposition
   figures must stay outside `importantbox`, `knowledgebox`, `warningbox`, and `dialoguebox`

9. End every major section with `\subsection{本章小结}`.
   Add `\subsection{拓展阅读}` when there are one or two worthwhile external links or local reference files.

10. End the document with a final top-level section such as `\section{总结与延伸}`.
    That final section must include:
    - the speaker's substantive closing discussion, excluding routine sign-off language
    - your own structured distillation of the core claims, mechanisms, and practical implications
    - your expanded synthesis, including conceptual compression, cross-links between sections, and any careful generalization that stays faithful to the video
    - concrete takeaways, open questions, or next steps when the material supports them

11. Do not emit `[cite]`-style placeholders anywhere in the LaTeX.

## Figure Handling

Select figures by necessity and teaching value, not by an arbitrary quota or a bias toward keeping the document visually sparse.

Frame understanding must come from direct visual inspection.

- Use the `view image` tool to inspect candidate frames and crops before deciding what they show, how they should be described, and whether they are complete enough to include.
- Do not use OCR tools such as `tesseract` as a substitute for visual understanding of a frame.
- Do not infer a frame's semantic content only from nearby subtitles, filenames, or timestamps without checking the image itself.
- Contact sheets, montages, and tiled strips are good for recall, but final keep-or-reject decisions and semantic naming must be based on actual image inspection with `view image`.

### Candidate Extraction

Prefer dense local extraction before down-selection.

Timestamped single-frame extraction:

```bash
ffmpeg -ss 00:12:31 -i "<video-path>" -frames:v 1 -q:v 2 "<work>/frames/raw_00-12-31.jpg"
```

Interval sampling:

```bash
ffmpeg -ss 00:12:00 -to 00:13:00 -i "<video-path>" -vf "fps=1,scale=1600:-1" "<work>/frames/segment_%04d.jpg"
```

Low-rate whole-video sampling for recall:

```bash
ffmpeg -i "<video-path>" -vf "fps=1/5,scale=1280:-1" "<work>/frames/scan_%05d.jpg"
```

Contact sheet example:

```bash
magick montage "<work>/frames/scan_*.jpg" -tile 5x -geometry 320x180+8+8 "<work>/frames/contact_sheet.jpg"
```

### Frame Selection Checklist

Before inserting any video frame, inspect several nearby candidates from the same subtitle-aligned or visually aligned interval and apply this checklist:

- Relevance: the frame must directly support the exact concept discussed in the surrounding paragraph or subsection.
- Required content visible: every visual element referenced in the text must already be visible in the frame.
- Fully revealed state: when slides, whiteboards, animations, dashboards, terminals, or UI demos build progressively, use the final fully populated readable state rather than an intermediate state.
- Best nearby candidate: compare multiple nearby frames and prefer the one that is both most complete and most readable.
- Readability: text, formulas, labels, code, terminal output, and diagram structure must be legible enough to justify inclusion.

### Frame Naming

- Use neutral timestamp-based names for raw candidate frames.
- Rename a frame semantically only after visually confirming what is fully visible in the image.
- The semantic filename must describe the frame's actual visible content, not a guess based on subtitles, nearby narration, or the intended paragraph topic.
- If the frame is partially revealed, transitional, or ambiguous, keep searching and do not lock in a semantic name yet.

## Figure Time Provenance

Whenever the `.tex` or PDF references a specific video frame, or a crop derived from a video frame, record its source time interval on the same page as a bottom footnote.

- The footnote must show the concrete time interval, for example `00:12:31--00:12:46`.
- The interval should come from the subtitle-aligned, audio-aligned, or visually aligned segment used to locate the figure, not from a vague chapter-level estimate.
- If the figure is a crop, the footnote still refers to the original video time interval of the source frame or subtitle span.
- If several nearby frames in one figure all come from the same source interval, one clear footnote is enough.
- Keep the figure and its time footnote anchored to the same page; prefer layouts such as `[H]`, a non-floating block, or another stable placement when ordinary floats would separate them.

## Visualization

For concepts that remain hard to explain with only screenshots and prose, add accurate visualizations.

Two acceptable routes:

- generate LaTeX-native visualizations with TikZ or PGFPlots
- generate figures ahead of time with scripts and include them as images

For script-generated illustrations, prefer Python tools such as `matplotlib` and `seaborn` when they are the clearest way to produce an accurate teaching figure.

When a visualization is generated externally rather than drawn natively in LaTeX:

- export the figure as `pdf` so it can be inserted into the `.tex` without rasterization loss
- prefer vector output for plots, charts, and schematic illustrations
- avoid `png` or `jpg` for script-generated teaching figures unless the content is inherently raster

Do not add decorative graphics that do not teach anything.

## Final Checklist

Before delivery, verify all of the following:

- the original local video and sidecar files were not overwritten
- the selected subtitle, transcript, or visual-only fallback is identified
- no important teaching content has been dropped, and no concrete but critical detail has been lost during condensation, restructuring, or summarization
- the text and figures are aligned: each inserted frame supports the surrounding explanation, necessary crops have been applied, and the chosen frame shows the fullest relevant information rather than a transitional or incomplete state
- every video-derived figure has a concrete time provenance footnote
- the document is visually rich enough for teaching
- the `.tex` compiles successfully to PDF

## Delivery

Deliver all of the following when produced:

- the source-map or chapter notes used to organize the video, if useful
- the selected subtitle file, including Whisper-generated SRT if speech-to-text was used
- the cover image or representative frame referenced on the front page
- any extracted or generated figure assets referenced by the document
- the final `.tex` file and the compiled `.pdf` file, using a reasonable filename derived from the local video title, for example `[视频标题]_notes.tex` and `[视频标题]_notes.pdf`

## Asset

- `assets/notes-template.tex`: default LaTeX template to copy and fill for local video notes
