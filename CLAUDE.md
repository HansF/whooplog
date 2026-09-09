# whooplog — working notes for Claude

Hugo + Hextra documentation site about a 1S FPV tinywhoop. Deployed to GitHub Pages at
`https://hansf.github.io/whooplog/`.

## Critical: the site lives on a subpath

`baseURL` is `https://hansf.github.io/whooplog/` (trailing slash required). `hugo server`
preserves the path, so local dev runs at `http://localhost:1313/whooplog/`, not `/`. This is
deliberate — it means subpath bugs surface locally.

**Two rules for content:**

1. Never write raw HTML with root-absolute paths. `<img src="/x.png">` and
   `<a href="/docs/">` bypass Hextra's render hooks and 404 in production. Use markdown
   syntax instead — the hooks pass those through `relURL`. If raw HTML is unavoidable, use
   `{{ "x.png" | relURL }}`.
2. Never hardcode `/whooplog/...` in content either. That breaks `hugo server` and any
   future domain move. Write `/docs/...` and let the hook prepend the base.

Verify before pushing:

```bash
hugo --gc --minify --cleanDestinationDir --printPathWarnings
grep -ro 'href="/[a-z]' public --include=*.html | grep -v '/whooplog/'
grep -ro 'src="/[a-z]'  public --include=*.html | grep -v '/whooplog/'
```

Both greps must return nothing.

## Theme

Hextra, pinned as a git submodule at `themes/hextra` (tag `v0.12.1`). It ships **precompiled
Tailwind CSS and contains no SCSS**, so CI needs no Dart Sass and no `npm install` — Hugo
extended alone is enough. Don't add a dart-sass step; that advice comes from Hextra's
pre-0.10 SCSS era.

`markup.highlight.noClasses = false` is required — Hextra's chroma CSS is class-based, and
without it code blocks ignore the light/dark toggle.

Layouts map automatically by section: `/blog` gets the theme's blog layout, `/log` and
`/reference` fall through to the docs-style list layout with a sidebar. No `cascade` needed.

## Blackbox log policy

Raw `.bbl`, decoded `.csv` and `.event` files are **gitignored**. Log pages carry the
analysis — tables, ratios, dB figures — and name their source file in frontmatter
(`log_file:`). `/docs/decode-and-analyze-blackbox/` documents the command that reproduces
the numbers from a local copy.

If committing logs ever becomes worthwhile, use **GitHub Release assets** — not Git LFS.
`actions/checkout` does not fetch LFS objects without `lfs: true`, so LFS would silently
deploy 130-byte pointer files in place of every log, and LFS bandwidth is quota-metered.

**Privacy:** this craft has no GPS, so its logs contain no location data. A GPS-equipped
craft would need home-point coordinates scrubbed before any log is committed to a public
repo.

## Content conventions

- YAML frontmatter in content; archetypes emit YAML too.
- Section pages: `title`, `description`, `lead`, `weight`, `toc`.
- `hugo new log/YYYY-MM-DD-name.md --kind log` gives the flight-log skeleton.
- Tone is reference voice, not diary. State findings as findings — "ratios ranged
  1.69–1.78, a spread under 5%", not "phew, it held up". First person only where an action
  was taken.
- Use callouts heavily: `warning` for traps, `error` for open issues, `info` for scope
  caveats. The site's value is largely in the gotchas.

## Scope discipline

Everything documented is specific to one airframe on alpha firmware. Don't generalise
settings into recommendations. Third-party reference material (Betaflight wiki mirrors,
tuning transcripts bundled with the betaflight-mcp plugin) may be **cited and linked, never
copied** into `content/`.

## Related

`/home/hans/Projects/betamcp` is a scratch workspace holding upstream clones (betaflight,
betaflight-configurator, betaflight-mcp, blackbox-log-viewer) and working log files. It is
not a git repo and contains no code of ours — this site is the actual output.
