---
name: social-media-launch
description: Plan and draft the promotion of a published post and its project on Reddit, Hacker News and similar venues - venue order, per-forum scouting, rules, post shape, and the first hours after posting. Use when the author wants to share a post, or asks where and how to post it.
---

# Launching a post on social media

## Strategy: one venue at a time

Never post everywhere at once.

- **You can only be in one comment section.** The first 60-90 minutes of replies are
  most of the value, and most of the ranking signal.
- **Each venue tests the framing.** Learn which title and opening land in a low-stakes
  venue, then carry that into the next.
- **Hacker News goes last.** HN dedupes URLs, so a submission that sinks mostly burns that
  link. You get roughly one good shot.

Typical order: a mid-size, on-topic subreddit, then a second subreddit matched to the
post's angle, then HN (Show HN if there is something to try, linking to the thing itself
with the write-up as the first comment). Venues used so far: r/LLMDevs first;
r/ClaudeAI was next in line because the Claude SDK comparison was the hook; lobste.rs is
invite-only. r/MachineLearning and r/programming were judged poor fits for engineering
write-ups.

## Before the first post

- **The project repo has a FOSS licence.** Many subreddits allow self-promotion only for
  open-source projects (r/LLMDevs rule 5). Without a licence, a public repo is still all
  rights reserved.
- **Every link returns 200:** the post, the repo, the demo if there is one, the video
  file.
- **A hosted demo** (if there is one): load-test it with a few hundred concurrent
  sessions before HN. Measure; don't guess.
- **The post has a social card:** an `image:` in its front matter (see `new-post`).
- **Optionally, a seeded discussion thread** (see `analytics-and-comments`), if you want
  a GitHub URL to point people to for a longer exchange.

## Scout each venue before drafting

For every venue, separately:

1. Read the sidebar or posting rules in full and quote the ones that bind
   (self-promotion, low-effort, misleading claims, flair).
2. Look at the top posts of the past month, and record which post *type* scores. On
   r/LLMDevs the best video post scored 428 and the best link post 56. That is why the
   clip is uploaded as media (see `recording-demo-clips`).
3. Note the title patterns that win. So far: first person, concrete, a number or a
   comparison, "open source".

## Post shape

- **Title:** "I built X that does Y", with the specific detail that makes it unusual.
  Avoid abstract or academic titles.
- **Body, written in the thread rather than a bare link:** the everyday problem, what
  you built, two or three findings, the comparison with existing tools, and links at
  the bottom (demo, write-up, code with its licence). End with an invitation to discuss
  the most surprising finding.
- **Claim only what was measured.** Say plainly that it is your project.
- **Prepare replies in advance** to the two or three questions that are certain to be
  asked (usually "how is this different from <framework>?"), each linking to the
  relevant section anchor of the post.

Give the author the full post as copy-paste blocks: link, title, body, flair, timing,
and the prepared replies. They will edit the wording.

## After posting

- Be present for the first 90 minutes and answer everything.
- Record which framing and objections came up. They shape the next venue's title, and
  sometimes a section of the post.
- Compare venues with GA traffic acquisition (see `analytics-and-comments`), not by
  score alone.
