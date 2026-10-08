---
name: recording-demo-clips
description: Record a short screen clip of a running app with Playwright and cut it into an MP4 for the blog and social posts, a GIF for a GitHub README, and a poster PNG. Use when a post needs a demo video, or when a project needs an animated README hero.
---

# Recording demo clips

A clip shows the one arc a reader should remember (here: question, spinner, red dot,
retry, green dot). Aim for 15-25 seconds. Everything is done headlessly: no screen
recorder, no system ffmpeg.

## What to record

- **Any locally running app works.** A hosted demo is not needed; start the app on a
  local port and drive that.
- **Make the run deterministic.** Scripted tools and canned model replies, not a live
  LLM, so a retake looks the same and costs nothing. If the app has no scripted mode,
  that is usually worth adding before recording.
- **Drive it, don't wait on wall-clock time.** Click the controls and poll the page
  for the end state, then hold about 2 seconds on the final frame.

## Tools

Both come from PyPI, so `uv run --with ...` is enough:

- **Playwright** (`playwright`, then `playwright install chromium` once). Use
  `new_context(viewport=..., record_video_dir=..., record_video_size=...)` with the
  same size for both. The `.webm` is written when the context is closed.
- **ffmpeg** via `imageio-ffmpeg`: `imageio_ffmpeg.get_ffmpeg_exe()` returns the path
  to a bundled binary.

## Cuts

From the one `.webm`, using the settings that have worked:

| output | for | ffmpeg essentials |
|---|---|---|
| MP4 | blog `<video>`, Reddit upload | `setpts=PTS/1.5`, `libx264 -preset slow -crf 24 -pix_fmt yuv420p -movflags +faststart`, `-an` |
| GIF | GitHub README (it won't play MP4 from markdown) | `setpts=PTS/2,fps=12,scale=900:-1:flags=lanczos`, two passes: `palettegen=stats_mode=diff`, then `paletteuse=dither=bayer:bayer_scale=3` |
| PNG | `poster` for the video, and the post's social card image | one representative frame: `-ss <t> -frames:v 1` |

Targets: MP4 under 1 MB, GIF under about 3 MB. If the GIF is too large, trim length
before lowering quality.

**Check before integrating.** Build a contact sheet (`fps=1/3,scale=440:-1,tile=4x2`,
`-frames:v 1`) and look at it. It shows in one image whether the arc is readable.

## Where the files go

- MP4 and PNG: `docs/assets/<post-slug>/` in this repo.
- GIF: the project's own repo (e.g. its `docs/`), referenced from its README.

Blog embed, with paths including the `/articles` baseurl:

```html
<video controls autoplay loop muted playsinline
       poster="/articles/docs/assets/<slug>/<clip>.png"
       aria-label="<one sentence describing what happens in the clip>">
  <source src="/articles/docs/assets/<slug>/<clip>.mp4" type="video/mp4">
  <img src="/articles/docs/assets/<slug>/<clip>.png" alt="<the final state>">
</video>
```

`muted` is required for autoplay. The poster doubles as the no-autoplay fallback.

For posting the clip, see the `social-media-launch` skill: upload the MP4 itself,
don't link it.
