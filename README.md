# crw-brew

Homebrew packaging for **[crw](https://github.com/us/crw)** — a Rust web
scraper, crawler and MCP server that turns URLs into clean Markdown or JSON —
for **macOS and Linux** (arm64 + x86_64).

Upstream ships pre-built binaries but no Homebrew formula. This repo generates
one and keeps it current in
[`jakobhviid/homebrew-tap`](https://github.com/jakobhviid/homebrew-tap).

## Install

```sh
brew install jakobhviid/tap/crw
crw example.com          # page as markdown on stdout
```

## No account needed

`crw` works fully locally. Scraping, crawling, mapping and the MCP server need
no key, no sign-up, and no network service beyond the site being read:

```sh
crw example.com                  # markdown on stdout
crw scrape example.com -f json   # structured
crw crawl example.com --depth 3
crw mcp                          # embedded MCP server, for opencode / Claude Code
crw doctor                       # read-only diagnostics
```

`crw search` is the one subcommand that needs a backend — either a local one or
the hosted CRW Cloud via `crw setup`, which is a paid service with a free credit
allowance. Nothing else here touches it.

## What it does and doesn't get through

`crw` fetches over HTTP first, and when a page comes back thin it escalates to
its own JS renderer, saying so on stderr:

```
info: HTTP returned thin content (10 chars markdown), escalating to JS renderer...
```

That escalation is what makes it useful on JavaScript-rendered sites — a Reddit
thread that a plain HTTP fetch reduces to an empty shell comes back as the full
post and comments.

It does **not** clear an active bot challenge. A Cloudflare managed challenge
(`cf-mitigated: challenge`) or DataDome (`x-datadome: protected`) returns the
interstitial — "Just a moment…", "Performing security verification" — as the
page content, because those checks want a real browser session rather than a
renderer. Read the output, not just the exit code: a challenge page is a
*successful* fetch of the wrong thing.

## How it works

- **macOS (and non-x86_64 Linux) download straight from upstream** — the
  formula's `url` points at `https://github.com/us/crw/releases/download/…`.
  Homebrew on macOS always has `clang` (via the required Command Line Tools), so
  it installs the upstream binary directly.
- **x86_64 Linux pours a bottle.** Homebrew-on-Linux requires a **C compiler**
  for any non-bottled formula — a problem on atomic/immutable distros (Bazzite,
  Silverblue) that don't ship one. A *bottle* is the only thing Homebrew pours
  without a compiler, so we repackage upstream's own Linux binary into an
  `x86_64_linux` bottle (no modification) and host it on this repo's GitHub
  releases. Same pattern amdl/grove/pwtune use.
- **Checksums:** upstream publishes a `SHA256SUMS` manifest alongside the
  release — already the hash Homebrew wants — so [`update.sh`](update.sh)
  verifies each downloaded asset against it and reuses the value verbatim.
- A daily GitHub Action ([`.github/workflows/update.yml`](.github/workflows/update.yml))
  runs `update.sh`; when upstream ships a new version it releases the new bottle
  and pushes the formula to the tap's `auto/formula-crw` branch, where the tap's
  gate installs and tests it and lands it on `main` only if it passes (opening
  an issue there if not). No-op when nothing changed. GitHub disables
  a repo's cron after 60 days of inactivity, so the job also pushes a throwaway
  `keep-alive` commit only when this repo is ≥50 days idle — enough to keep the
  schedule alive without daily noise.

`update.sh` avoids bash 4 features so it runs on stock macOS (bash 3.2) as well
as in CI, which is what makes a local bump actually work.

## Setup (one time)

The updater pushes to the tap over SSH, so add the tap's deploy key as a secret
in **this** repo:

- `TAP_DEPLOY_KEY` — the same Ed25519 deploy key used by your other tool repos
  to write to `jakobhviid/homebrew-tap`.

Then run the workflow once (`Actions → update-formula → Run workflow`) to
publish a new version, or bump it locally:
`./update.sh && cp crw.rb …/homebrew-tap/Formula/`.

## Notes

- Single self-contained binary (~13 MB); no runtime dependencies. The MCP server
  is the same binary (`crw mcp`), not a second install.
- Linux uses the glibc build; Windows is intentionally skipped (Homebrew targets
  glibc on Linux and doesn't do Windows).
- Upstream also publishes `crw-server` and `crw-mcp` as separate binaries. They
  aren't packaged here because `crw serve` and `crw mcp` are subcommands of the
  one binary this formula installs.

## License

crw is licensed **AGPL-3.0-only** (upstream declares the deprecated bare
`AGPL-3.0`; its LICENSE is AGPLv3 with no "or later" clause). This packaging is
MIT.

A Homebrew formula is a build recipe, not the program — for macOS and arm64
Linux the user downloads from upstream, so nothing here conveys crw. The
`x86_64_linux` bottle *is* a conveyed binary, so it ships unmodified, keeps its
license, and the formula names the exact upstream release tag it was built from,
which is where its corresponding source lives.
