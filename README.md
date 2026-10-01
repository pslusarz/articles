# articles

Source for <https://pslusarz.github.io/articles/> — a Jekyll site where I write up
things I have built or investigated.

Posts live in `_posts/`, named `YYYY-MM-DD-slug.md`. Work in progress goes in
`_drafts/`, which is only served with `--drafts`. A draft still takes its URL from the
`date` in its front matter rather than from a bare slug, so it previews at
`/articles/YYYY/MM/DD/slug.html` and publishing it is a move into `_posts/` under the
matching filename.

Supporting material for a post (scans, raw tool output, data) goes in
`docs/assets/<post-slug>/` so it can be linked from the article and inspected directly.

## Local preview

Jekyll 4 needs Ruby >= 2.7 and macOS ships 2.6, so the system Ruby will not do:

```sh
brew install ruby
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"   # brew installs it unlinked
bundle install
bundle exec jekyll serve --drafts --livereload
```

The site is then at <http://127.0.0.1:4000/articles/>. `--livereload` refreshes the
browser on save, which is most of the value while a post is still moving.

## Writing conventions

- **Collapse the depth rather than cutting it.** Supporting detail goes in
  `<details class="deep-dive" markdown="1">` behind a `<summary>` line. Kramdown needs
  that `markdown="1"` or the contents render as literal HTML.
- **One palette throughout a post.** `assets/main.scss` defines `.tok-user`,
  `.tok-call`, `.tok-result` and `.tok-wait`. When a diagram and a transcript refer to
  the same thing, give it the same colour in both; `pre.wire` is the plain transcript
  block they sit in.
- **Two-up comparisons.** `<div class="sbs">` lays `<figure class="panel">` elements
  side by side and wraps them on narrow screens. Code inside is scaled down to fit a
  half-width column, so keep those lines under roughly 50 characters.
- **Diagrams are inline SVG.** No build step and no JavaScript, so GitHub Pages serves
  them unchanged and they stay legible when scaled.
