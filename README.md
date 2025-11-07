# boi-wan.github.io

This repository contains the source for my personal blog built with Hugo and the PaperMod theme.

## About
- Static site generated with Hugo.
- Theme: PaperMod (clean, fast, and customizable Hugo theme).

## Requirements
- Hugo (recommended: Hugo Extended)
  - Install: https://gohugo.io/getting-started/installing/
- Git

## Quick start (local)
1. Clone the repo:
   git clone https://github.com/boi-wan/boi-wan.github.io.git
2. From the repo root run the dev server:
   hugo server -D
3. Open http://localhost:1313 to preview.

## Build
- Produce the production site:
  hugo --minify
- The generated site will be in the `public/` directory.

## Deploy
- You can deploy the contents of `public/` to GitHub Pages, Netlify, or any static host.
- Common options:
  - GitHub Actions to build and push to `gh-pages` or `main`/`gh-pages` depending on your Pages config.
  - Netlify: connect the repository and set the build command to `hugo --minify`.

## PaperMod notes
- PaperMod is a lightweight, configurable theme with many options (colors, fonts, layouts).
- Configuration is in `config.*` and theme params (see theme docs for available settings).
- If you need to customize styles, use Hugo's asset pipeline (SCSS) — Hugo Extended is required for SCSS.

## Useful links
- Hugo — https://gohugo.io/
- Hugo installation — https://gohugo.io/getting-started/installing/
- PaperMod theme — https://github.com/adityatelange/hugo-PaperMod
- PaperMod docs / demo — see the PaperMod repo README and demo site
- Netlify — https://www.netlify.com/

If you need help running or customizing the site, open an issue or reach out via my profile.
