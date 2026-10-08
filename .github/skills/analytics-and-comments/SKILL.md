---
name: analytics-and-comments
description: How this site measures readers (Google Analytics 4, Search Console) and hosts comments (giscus on GitHub Discussions), and what to do for each when a new post is published. Use after publishing a post, when checking traffic, or when debugging comments or indexing.
---

# Analytics and comments

Everything is configured in `_config.yml`; nothing here needs a code change for a new
post.

## Google Analytics 4

- Measurement id is `google_analytics` in `_config.yml`. minima's own snippet is the dead
  Universal Analytics one, so `_includes/google-analytics.html` overrides it with
  `gtag.js` and keeps a Do Not Track guard.
- It is only emitted when `JEKYLL_ENV=production`, which the Pages workflow sets.
  **Local previews send nothing**, and a browser with DNT on sends nothing either, so
  check the live page, not the preview.
- To confirm a post is tracked: the live HTML contains the `G-` id, and GA's
  **Realtime** report shows a visit from a browser without DNT.
- Property settings already done: data retention 14 months, enhanced measurement on
  (scroll depth, outbound clicks, downloads).
- After a social post, judge each venue by **Acquisition → Traffic acquisition**
  (session source) and engagement time. Upvotes alone are misleading.

## Search Console

- The property is the **URL prefix** `https://pslusarz.github.io/articles/`, verified by
  `google_site_verification` in `_config.yml`. A root or Domain property is not possible
  on `github.io`.
- `jekyll-sitemap` lists new posts automatically. In the sitemap box submit only
  `sitemap.xml`, because the property prefix is filled in already.
- For a new post: **URL Inspection → Request indexing** with its live URL. That is
  faster than waiting for the sitemap to be re-read.

## Comments: giscus

- Each post gets a "Comments and reactions" section backed by GitHub Discussions on
  `pslusarz/articles`, category **Announcements**. Only the owner and the giscus bot can
  open threads; visitors can only reply, which is the anti-spam choice.
- Mapping is `pathname` with `data-strict="1"`. The discussion **title** is the path
  without the leading `/` or the extension, e.g.
  `articles/2026/10/01/an-agent-you-can-interrupt`. The **body** must contain the SHA-1
  of that title, conventionally as `<!-- sha1: ... -->`.
- Changing a published post's date or slug changes its path and orphans its comments.
  If you must, retitle the discussion and update the hash in the same change.

## Optional: seed the discussion when a post goes live

Comments work without this: the first comment or reaction makes the giscus bot create
the thread. Seed it yourself only when you want an opening prompt from the author, or a
thread URL to share before anyone has commented:

1. Title = the path rule above. Hash: `printf '%s' "<title>" | shasum -a 1`.
2. Body: one or two sentences inviting a specific kind of feedback, a link to the post,
   and `<!-- sha1: <hash> -->`.
3. Create it with `gh api graphql` and the `createDiscussion` mutation, using
   `repo_id` and `category_id` from the `giscus` block in `_config.yml`.
4. Open the live post and confirm the widget shows that thread. A wrong title or hash
   makes giscus create a duplicate on the first comment.

This is public, so confirm the wording with the author before creating it.
