# CLAUDE.md — ttrpg-terra

Quartz **v5** site that publishes the D&D campaign wiki (Karpathy-style, Claude Code–maintained) at `strakul.com/ttrpg-terra`.

This repo is a fork of `jackyzha0/quartz` holding **only site config/engine**. Wiki content lives in a separate repo and is pointed at with `-d` at build time — never copied, symlinked, or committed here.

Full background notes: `C:\Users\strak\Dropbox\Personal\Projects\General Tech\TTRPG Coding\Quartz Setup for TTRPG Wiki.md`

## Repos

| Repo | Local path | Remote | Branch |
| --- | --- | --- | --- |
| Quartz site (this) | `C:\Users\strak\Projects\ttrpg-terra` | `origin` = `dr-rodriguez/ttrpg-terra`, `upstream` = `jackyzha0/quartz` | `v5` |
| Wiki content | `C:\Users\strak\Projects\DND LLM Wiki` | `dr-rodriguez/dnd-llm-wiki` (public) | `main` |

Only `wiki/` of the content repo is published. `raw/`, `.agents/`, `AGENTS.md` are never built (but are already public on GitHub).

Wiki `wiki/` currently has: `index.md`, `log.md`, `Base Tables/` (`.base` files), `Characters/`, `Images/`, `Locations/`, `Lore/`, `Quests/`, `Relationships/`, `Sessions/`.

## Quartz v5, not v4

- Config is the single `quartz.config.yaml` (no `quartz.config.ts` / `quartz.layout.ts`). `quartz.ts` just loads the YAML — don't edit it.
- `quartz.config.default.yaml` is upstream's default — reference only.
- Features are plugins from the `quartz-community` org, listed under `plugins:` in the YAML and pinned in `quartz.lock.json`. Installed into `.quartz/` (gitignored).
- Many online guides describe v4. Check the bundled v5 docs in `docs/` first (`docs/configuration.md`, `docs/cli/*.md`, `docs/advanced/*.md`, `docs/features/*.md`), then https://quartz.jzhao.xyz/.

## Commands

Run from repo root. Environment: Windows, native (not WSL). Node 22+ (`.node-version`), npm 10.9.2+.

```bash
npm ci                                                 # install deps
npx quartz plugin install                              # install plugins from quartz.lock.json into .quartz/
npx quartz build -d "../DND LLM Wiki/wiki" --serve     # local preview at http://localhost:8080
npx quartz build -d "../DND LLM Wiki/wiki"             # build only -> public/
npm run check                                          # tsc + prettier check
```

- Local preview and CI use the same config; only the `-d` path differs. CI path will be `dnd-llm-wiki/wiki`.
- Don't use `npx quartz create` again — already done (`--template ttrpg --strategy new --links shortest --baseUrl strakul.com/ttrpg-terra`).
- `content/index.md` is the placeholder from `--strategy new`. Not used for the real site; leave it.
- Don't create junctions/symlinks in `content/` pointing at `C:\...` — breaks CI.

## Current config highlights (`quartz.config.yaml`)

- `pageTitle: DnD Wiki`, `baseUrl: strakul.com/ttrpg-terra`, `locale: en-US`.
- `ignorePatterns`: `private`, `templates`, `.obsidian`.
- Theme: `@quartz-themes/core` with `its-theme` / `ttrpg-dnd` variation, both modes.
- Enabled extras: `bases-page` (renders `.base` files), `canvas-page`, `note-properties`, `remove-draft` (`draft: true` hides pages), `unlisted-pages`, `alias-redirects`, `github:Requiae/quartz-leaflet-bases-plugin`, `obsidian-plugin-excalidraw`.
- Footer links: GitHub, `strakul.com/blog`.
- `baseUrl` must match the repo name (Pages serves project repos at `strakul.com/<repo>`). Rename repo → update `baseUrl`. RSS/sitemap depend on it.

## Committed vs ignored

- Commit: `quartz.config.yaml`, `quartz.lock.json`, `package.json`, `package-lock.json`, `CLAUDE.md`, workflows.
- Gitignored: `.quartz/`, `public/`, `node_modules/`, `.quartz-cache`, `private/`.
- Never commit wiki content into this repo.

## Publishing plan (NOT set up yet)

Hosting: GitHub Pages on `dr-rodriguez/ttrpg-terra`. `dr-rodriguez.github.io` already has `CNAME` → `strakul.com`, so any project repo with Pages enabled serves at `strakul.com/<repo-name>` (same as `/blog`).

Chosen approach: **two-checkout workflow** in this repo — check out this repo + sparse-checkout `wiki/` from `dnd-llm-wiki`, build with `-d`.

Planned `.github/workflows/deploy.yaml` (start from https://quartz.jzhao.xyz/hosting):

- Triggers: `push` to `v5`, `workflow_dispatch`, `repository_dispatch: types: [wiki-updated]`.
- Permissions: `contents: read`, `pages: write`, `id-token: write`. Concurrency group `pages`.
- Steps: checkout (fetch-depth 0) → checkout `dr-rodriguez/dnd-llm-wiki` to `path: dnd-llm-wiki`, `sparse-checkout: wiki`, `fetch-depth: 0` → setup-node 24 → cache `~/.npm` (key `package-lock.json`) and `.quartz/plugins` (key `quartz.lock.json`) → `npm ci` → `npx quartz plugin install` → `npx quartz build -d dnd-llm-wiki/wiki` → `upload-pages-artifact` (`path: public`) → `deploy-pages` job.
- Fallback if nested `-d` misbehaves: `rm -rf content && cp -r dnd-llm-wiki/wiki content && npx quartz build`.
- Match action versions to upstream's existing workflows (they use `actions/checkout@v6`, `actions/setup-node@v6`, `actions/cache@v5`).

Rebuild on wiki change: workflow in `dnd-llm-wiki` on push to `main` with `paths: ["wiki/**"]` running
`gh api repos/dr-rodriguez/ttrpg-terra/dispatches -f event_type=wiki-updated` with `GH_TOKEN: ${{ secrets.QUARTZ_DISPATCH_TOKEN }}`.
Token = fine-grained PAT scoped only to `ttrpg-terra`, **Contents: read and write**, with expiry; stored as secret in `dnd-llm-wiki`. (Setup note says `repos/dr-rodriguez/ttrpg/...` — repo is actually `ttrpg-terra`.)

### Upstream workflows in `.github/workflows/`

`ci.yaml`, `deploy-v5.yaml`, `build-preview.yaml`, `deploy-preview.yaml`, `docker-build-push.yaml`, `dependabot-automerge.yaml` are upstream Quartz's own. All are guarded by `if: github.repository == 'jackyzha0/quartz'` so their jobs no-op here (still show as skipped runs). Consider deleting them, plus `.github/dependabot.yml` (not guarded — will open dependency PRs on this repo) and `FUNDING.yml`. Our deploy workflow should be a new file.

### Pre-publish checklist

- [ ] **`cname` plugin is enabled** — it writes `public/CNAME` containing `strakul.com` (hostname from `baseUrl`). For a project repo under the `dr-rodriguez.github.io` custom domain this is unwanted; disable it (`enabled: false`) before first deploy.
- [ ] `analytics: provider: plausible` is set — confirm a Plausible site exists for strakul.com or remove it.
- [ ] Remove/neutralize upstream workflows (above); add `deploy.yaml`.
- [ ] Repo Settings → Pages → Source: **GitHub Actions**.
- [ ] Verify `strakul.com/ttrpg-terra` loads; check Cloudflare isn't overriding routing.
- [ ] Verify page dates come from the `dnd-llm-wiki` git history, not this repo.
- [ ] Wiki audit: no `[[links]]` into `raw/` (render as unresolved), all embedded images under `wiki/` (e.g. `wiki/Images/`), `draft: true` on anything private/spoiler-y, no players' real names.
- [ ] Root homepage: `wiki/index.md` must exist (lowercase) or `/` 404s.
- [ ] Set up dispatch workflow + PAT in `dnd-llm-wiki`.
- [ ] Link from main site (`dr-rodriguez/dr-rodriguez.github.io`, HTML5 UP Dimension template).

## Wiki content conventions (for agent maintaining `dnd-llm-wiki`)

- Obsidian `[[wikilinks]]`, `![[embeds]]`, callouts render natively. Links resolve by `shortest` path.
- Entity frontmatter acts as an infobox (`note-properties` plugin shows it; Bases filter on it):
  ```yaml
  title: Sildar Hallwinter
  type: npc
  campaign: phandelver
  status: alive
  first_seen: "[[Session 03]]"
  tags: [npc, ally]
  aliases: [Sildar]
  ```
- `draft: true` → excluded from site. Folders in `ignorePatterns` → excluded.
- Don't name a top-level folder `index`; don't use `Index.md` (capital I).

## Gotchas

- Plugins must be installed before build (`npx quartz plugin install`) — CI too.
- Paths with spaces (`DND LLM Wiki`) need quoting.
- Upgrading Quartz: `upstream` remote is `jackyzha0/quartz`; see `docs/cli/upgrade.md`. Avoid touching `quartz/` engine source unless necessary — keeps upgrades clean.
