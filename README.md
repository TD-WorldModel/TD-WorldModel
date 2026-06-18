# Contrastive World Models

> Contrastive is all you need

A single-page Jekyll project website for the paper, styled like
[DINO-WM](https://dino-wm.github.io/). **You only ever write Markdown** — the
HTML layout is set up once and you don't touch it.

## What you edit

- **`index.md`** — the whole page.
  - The **front matter** (between the `---` lines at the top) holds structured
    data: title, subtitle, authors, affiliations, link buttons, teaser, BibTeX.
  - **Everything below** is plain Markdown — Abstract, Method, Results, and any
    other `##` sections. It's one long scrolling page; add as many sections as
    you want.
- **`static/images/`** — drop figures/videos here, then reference them:
  ```markdown
  ![caption]({{ "/static/images/method.png" | relative_url }})
  ```
  (The `relative_url` filter keeps links working under the repo's base path.)

## What you don't touch

`_layouts/` + `_includes/` (the HTML shell) and `static/css/index.css`
(styling). Set up once; ignore them.

## Install (one time)

You need a modern Ruby. On macOS with [Homebrew](https://brew.sh):

```bash
brew install ruby
```

Add Homebrew's Ruby to your PATH (paste into `~/.zshrc`, then open a new terminal):

```bash
export PATH="/opt/homebrew/opt/ruby/bin:/opt/homebrew/lib/ruby/gems/4.0.0/bin:$PATH"
```

Then install the site's dependencies (run in this folder):

```bash
bundle install
```

Check it worked: `ruby -v` should print 4.x (not 2.6).

## Preview locally (optional)

```bash
bundle exec jekyll serve --livereload
# open http://localhost:4000/contrastive-wm-draft/
```

Edit `index.md`, save, and the page reloads automatically. Stop with `Ctrl-C`.

No Ruby/Jekyll? Skip all this — just push and GitHub builds the site for you.

## Publish

```bash
git add -A && git commit -m "Update site" && git push origin main
```

Once, in the repo: **Settings → Pages → Source: "Deploy from a branch" →
`main` / `(root)`**. Published at `https://7peng.github.io/contrastive-wm-draft/`.

> `baseurl` in `_config.yml` is `/contrastive-wm-draft` to match the repo name.
> Change it if you rename the repo or use a custom domain.
