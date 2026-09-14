# Campaign Wiki Setup Guide: Obsidian → Quartz → GitHub Pages

## What you'll end up with

```
campaign-wiki/                  (GitHub repo root)
├── vault/                      (your actual Obsidian vault — edit here)
│   ├── .obsidian/
│   ├── NPCs/
│   ├── Locations/
│   └── ...
├── quartz/                     (the site generator)
│   ├── content -> ../vault      (symlink, so Quartz reads your vault directly)
│   ├── quartz.config.ts
│   └── ...
└── .github/workflows/deploy.yml
```

- Live site at `https://yourusername.github.io/campaign-wiki/`
- You keep writing in Obsidian exactly as before
- Players suggest edits via GitHub pull requests on the same repo

---

## Before you start

Open a terminal (macOS/Linux terminal, or Git Bash / PowerShell on Windows) and check:

```bash
node -v      # Quartz now requires v22 or higher
npm -v
git --version
```

If anything's missing: Node.js from nodejs.org (LTS installer), Git from git-scm.com.

You'll also need a GitHub account, and to know exactly where your current Obsidian vault folder lives on disk.

---

## Phase 1 — Decide what players are allowed to see (do this first)

This matters more here than for a normal wiki: your vault almost certainly holds GM secrets — monster stats, plot twists, an NPC's true motive.

The setup below defaults to **nothing is published unless you say so**. You'll do this with frontmatter: any note without `publish: true` at the top never makes it into the built site, no matter what. That gets wired up in Phase 7 — just keep in mind as you go that every note starts private.

---

## Phase 2 — Create the GitHub repository

1. Go to `github.com/new` while logged in
2. Repository name: `campaign-wiki` (or anything you like — it becomes part of the site's URL)
3. Visibility: **Public** (needed for free GitHub Pages, and for the fork-based edit workflow in Phase 14)
4. Leave "Add a README," ".gitignore," and "license" all **unchecked** — you want a completely empty repo
5. Click **Create repository**
6. Copy the HTTPS URL shown (looks like `https://github.com/yourusername/campaign-wiki.git`) — you'll need it next

---

## Phase 3 — Set up the local project folder

```bash
mkdir ~/campaign-wiki
cd ~/campaign-wiki
git init
git remote add origin https://github.com/yourusername/campaign-wiki.git
mkdir vault
```

(Use your actual repo URL from Phase 2, step 6.)

---

## Phase 4 — Move your existing vault into place

1. Copy the entire contents of your current vault into the new `vault/` folder:

   **macOS/Linux:**
   ```bash
   cp -R "/path/to/your/existing/vault/." ~/campaign-wiki/vault/
   ```
   **Windows (PowerShell):**
   ```powershell
   Copy-Item -Path "C:\path\to\your\existing\vault\*" -Destination "C:\campaign-wiki\vault" -Recurse
   ```
   Copy, don't move, at first — leave your original vault intact until everything's confirmed working.

2. In Obsidian: **File → Open another vault → Open folder as vault** → select `campaign-wiki/vault`
3. Confirm it opens normally — links, tags, and attachments should resolve fine, since `.obsidian` came along with everything else
4. From now on, write here

---

## Phase 5 — Install Quartz

```bash
cd ~/campaign-wiki
git clone https://github.com/jackyzha0/quartz.git
cd quartz
npm i
npx quartz create
```

Quartz moved to a template-based setup wizard (Quartz 5). Answer the prompts as they come:

- **Template** → **TTRPG** — it builds on the Obsidian template (full wikilink/callout support) and adds an interactive map plugin plus a D&D-flavored theme. This is the one built for exactly your use case
- **Content strategy** → **Symlink**
- **Source path** → `../vault` — must be **relative**, not an absolute path. An absolute path (like `C:\Users\Caleb\campaign-wiki\vault`) works fine locally but produces an empty, broken site once GitHub Actions builds it, since the runner has no access to your PC's filesystem. This is a known gotcha, not a hypothetical one.
- **Base URL** → `yourusername.github.io/campaign-wiki` (no `https://`, no trailing slash)
- The link-resolution prompt is skipped — TTRPG locks it to shortest path automatically

Once it finishes, install the plugins the TTRPG template references:
```bash
npx quartz plugin install --from-config
```

Check the symlink worked:
```bash
ls content
```
You should see your vault's actual folders and notes.

**On Windows, verify it's a real symlink, not a copy** — creating symlinks needs Developer Mode or Administrator rights, and Quartz can silently fall back to copying files instead if neither is available:
```powershell
(Get-Item content).LinkType
```
This should print `SymbolicLink`. If it prints nothing, it's a plain folder with copied files — edits to your vault won't propagate. Fix: enable Developer Mode (**Settings → Privacy & Security → For Developers**), then:
```powershell
Remove-Item -Recurse -Force content
New-Item -ItemType SymbolicLink -Path content -Target ..\vault
```

---

## Phase 6 — Configure the site

Quartz's config file is now `quartz/quartz.config.yaml` — YAML, not TypeScript, as of Quartz 5. `baseUrl` should already be set correctly from the Phase 5 wizard; worth a quick check that it matches your Pages URL exactly, since a mismatch breaks internal links.

Under `configuration:`, set:
- `pageTitle` → your campaign's name, e.g. `"The Ashfall Chronicles"`

Save the file.

---

## Phase 7 — Turn on the publish filter

This is the step that keeps GM secrets out of players' hands, so it's worth getting exactly right rather than approximately right.

The `ExplicitPublish` filter still exists in current Quartz and works the same way conceptually: a note is only built if its frontmatter has `publish: true`. What changed is the config file format (YAML now, not TypeScript) — and rather than hand you guessed YAML syntax for something this sensitive, open `quartz/quartz.config.yaml`, find the `plugins:` list, and share what's there with me. I'll give you the exact line to add for your actual file instead of risking a wrong filter that silently publishes everything.

Once it's wired up, add this to the top of every note you want players to see:
```
---
title: NPC Name or Page Title
publish: true
---
```
Adding an explicit `title:` avoids Quartz falling back to awkward auto-generated titles. Everything else — session prep, secret stat blocks, plot notes — stays untouched and, once the filter's confirmed working, won't exist on the built site.

Whatever the exact syntax turns out to be: always verify locally in Phase 8, before ever pushing, by confirming un-published notes genuinely don't show up.

---

## Phase 8 — Test locally before anything goes live

```bash
npx quartz build --serve
```
Open `http://localhost:8080`. Click through:
- Only `publish: true` notes should appear
- Links between notes should resolve
- Images/attachments should load

Press `Ctrl+C` in the terminal to stop the local server when done.

---

## Phase 9 — Push everything to GitHub

Quartz was cloned as its own git repo, so remove its inner `.git` folder first, or it'll conflict with yours:

**macOS/Linux:**
```bash
cd ~/campaign-wiki/quartz
rm -rf .git
cd ~/campaign-wiki
```
**Windows (PowerShell):**
```powershell
cd ~\campaign-wiki\quartz
Remove-Item -Recurse -Force .git
cd ~\campaign-wiki
```
Then commit:
```bash
git add .
git commit -m "Initial campaign wiki setup"
git branch -M main
git push -u origin main
```

This first push to a brand-new repo will trigger a GitHub sign-in. On Windows, Git's credential manager usually handles this with a browser window or tab — which can open behind your terminal or in the taskbar rather than front and center. If the terminal looks stuck right after `git push`, check for that window before assuming something's wrong.

---

## Phase 10 — Set up the deploy workflow

GitHub only runs workflows from the repo root's `.github/workflows/`, but Quartz's bundled one is nested at `quartz/.github/workflows/deploy.yml`. Move it up:

**macOS/Linux:**
```bash
cd ~/campaign-wiki
mkdir -p .github/workflows
mv quartz/.github/workflows/deploy.yml .github/workflows/deploy.yml
```
**Windows (PowerShell):**
```powershell
cd ~\campaign-wiki
New-Item -ItemType Directory -Force -Path .github\workflows
Move-Item quartz\.github\workflows\deploy.yml .github\workflows\deploy.yml
```

Then open `.github/workflows/deploy.yml` and replace its contents with this — adapted so every step knows Quartz lives in a subfolder, not the repo root:

```yaml
name: Deploy Quartz site to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
      - name: Install dependencies
        working-directory: ./quartz
        run: npm ci
      - name: Build site
        working-directory: ./quartz
        run: npx quartz build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./quartz/public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

`fetch-depth: 0` matters — Quartz reads git history to show "last edited" dates on pages.

Commit and push:
```bash
git add .github
git commit -m "Add deploy workflow"
git push
```

---

## Phase 11 — Turn on GitHub Pages

1. On GitHub.com, open your repo → **Settings** → **Pages** (left sidebar)
2. Under "Build and deployment" → **Source** dropdown → select **GitHub Actions** (not "Deploy from a branch")
3. Go to the **Actions** tab — you should see the workflow from Phase 10's push running. Click in to watch it. A green checkmark means it worked

---

## Phase 12 — Verify the live site

Settings → Pages will now show the live URL: `https://yourusername.github.io/campaign-wiki/`

Visit it and click around. Then check secrecy actually works: try guessing the URL of a note you deliberately left un-published — it should 404.

---

## Phase 13 — Your ongoing writing loop

Whenever you add or update campaign notes:
1. Write in Obsidian as normal, inside `campaign-wiki/vault/`
2. Add `publish: true` to any new note meant for players
3. In terminal:
   ```bash
   cd ~/campaign-wiki
   git add .
   git commit -m "describe what changed"
   git push
   ```
4. GitHub Actions rebuilds and redeploys automatically within a minute or two

---

## Phase 14 — Let players suggest edits

**Option A — Add players as collaborators** (best for a small, trusted table)
1. Repo → **Settings** → **Collaborators and teams** → **Add people** → enter their GitHub username/email → role: **Write**
2. They accept the email invite
3. Lock down `main` so their edits become proposals, not instant changes: **Settings → Branches → Add branch protection rule** → branch name pattern `main` → check **"Require a pull request before merging"** → Create
4. Now, when a player opens a file on GitHub.com and clicks the pencil (edit) icon, GitHub prompts them to create a new branch and open a pull request instead of committing straight to `main`
5. Review it under the repo's **Pull requests** tab, comment if needed, then **Merge pull request** — this triggers an automatic redeploy

**Option B — Public repo, no invites needed** (closer to how Wikipedia itself works)
1. Repo is already Public from Phase 2
2. Any logged-in GitHub user can click the pencil icon on a file; GitHub automatically forks the repo behind the scenes and opens a PR back to yours when they hit "Propose changes"
3. Review and merge exactly as in Option A

---

## Troubleshooting

- **Links broken on the live site** → `baseUrl` in `quartz.config.ts` doesn't exactly match your Pages URL
- **`content/` looks empty in the Actions build log** → the symlink was made with an absolute path instead of `../vault`; redo Phase 5
- **A note marked `publish: true` still isn't showing** → check for a typo in the frontmatter key, and that the `---` fences are each on their own line
- **Action fails at the build step** → open the failed run's log under the Actions tab; usually a missing `title:` in some note's frontmatter, or occasionally a deprecated action version — if GitHub flags `upload-pages-artifact` or `deploy-pages` as outdated, bump the version tag shown in its Marketplace listing
- **Dataview queries or Canvas files** → not supported by Quartz; replace with static text/tables, or a screenshot for Canvas boards
- **Build fails with `Cannot find module '@quartz-themes/its-theme/ttrpg-dnd'`** → known bug in the TTRPG template's default theme entry. Set `enabled: false` on the `@quartz-themes/core` plugin block in `quartz.config.yaml` to unblock the build — purely cosmetic, doesn't affect content or the publish filter
- **Local preview shows zero pages** → expected if no note has `publish: true` yet (the explicit-publish filter is working correctly); add it to one note to confirm the pipeline end-to-end