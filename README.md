# &lt;username&gt;.github.io

Personal content hub, published with GitHub Pages (Jekyll + minima).

## What this is

A growing body of technical writing and reference material on agentic AI, Power Platform, and governance on the Microsoft stack. Articles are the body of work; short teasers go to LinkedIn and link back here.

## How it is published

GitHub Pages, building from the default branch root with Jekyll.

One-time setup on GitHub:

1. Create a public repository named exactly `<username>.github.io` (replace with your GitHub username).
2. Push these files to the default branch (`main`).
3. In the repo, open Settings, then Pages.
4. Under "Build and deployment", set Source to "Deploy from a branch", branch `main`, folder `/ (root)`. Save.
5. Wait about a minute. The site goes live at `https://<username>.github.io/`.

Before you push, edit `_config.yml` and replace `<username>` in the `url` line.

## Adding an article

Drop a finalized article into `_posts/` named `YYYY-MM-DD-slug.md` with this front matter:

    ---
    layout: post
    title: "Your title"
    date: YYYY-MM-DD
    tags: [tag-one, tag-two]
    ---

This is the format the weekly `github-content` task produces, so its drafts in `Vault/GitHub/Drafts` move straight in after your review.

## Preview locally (optional)

With Ruby installed:

    bundle install
    bundle exec jekyll serve

Then open `http://127.0.0.1:4000/`.

## Theme

Using `minima`. To change it, edit `theme` in `_config.yml`. Browse supported themes at https://pages.github.com/themes/ or use any gem-based or remote theme.
