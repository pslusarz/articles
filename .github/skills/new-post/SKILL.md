---
name: new-post
description: Start, preview, publish and verify a new post on this Jekyll site, from draft to live page, then hand off to the media, analytics and social skills. Use when the author wants to write up a new project or investigation.
---

# Writing and publishing a new post

Writing conventions (deep-dive blocks, colour palette, side-by-side panels, inline SVG)
and the local preview setup are in `README.md`. Read it first.

## Start

- Draft in `_drafts/YYYY-MM-DD-slug.md`. The URL comes from the `date` in front matter,
  so pick the date and slug up front; changing them after publication orphans comments.
- Front matter:
  - `title`
  - `date`
  - `description`: one sentence of about 150-160 characters. It is the search snippet
    and the social preview text.
  - `image`: a 1200×630 PNG under `docs/assets/<slug>/`. `jekyll-seo-tag` turns it into
    the `og:image` card shown when the link is shared.
- Supporting files go in `docs/assets/<slug>/`.
- The house shape so far: a **Summary**, **Motivation** and **Audience** paragraph at the
  top, then a linked **Contents** list, then **Main findings** as bullets, then the
  sections. When an existing tool solves part of the problem, include a section that
  rebuilds the idea on that tool, as a control.
- If the project has a working app, record a clip (`recording-demo-clips`), and make a
  1200×630 social card for `image` as described there.

## Preview

`bundle exec jekyll serve --drafts --livereload` with Homebrew Ruby on the PATH (see
`README.md`). Check that every Contents anchor resolves and that media load.

## Publish

1. Move the draft to `_posts/` under the same filename, commit and push to `main`.
2. The Pages workflow builds with `JEKYLL_ENV=production`. Watch it with `gh run watch`,
   or poll the live URL until the new content appears.
3. Verify the live page: title, media URLs return 200, Contents anchors, the GA id is
   present, the comments section renders.

Pushing publishes. Get the author's go-ahead before every push to `main`.

## After publishing

1. `analytics-and-comments`: request indexing; seed the discussion only if wanted.
2. The project repo: its README links to the post, it has a licence, and it has an
   animated hero if there is a clip.
3. `social-media-launch`: one venue at a time.
