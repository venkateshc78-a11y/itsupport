---
description: Security-scan, write the README, push to a GitHub repo, set the About section, and stand up GitHub Pages via Actions
argument-hint: <github-repo-url-or-owner/repo>
---

# Publish to GitHub + GitHub Pages

Goal: take the current project and publish it to the GitHub repo the user names,
with a real README, a GitHub Pages deployment wired through GitHub Actions, and
the repo's About section pointing at the live Pages URL — but only after
confirming nothing sensitive is about to leave this machine.

The target repo is: $1
(If empty, ask the user for the GitHub repo URL or `owner/repo` before doing anything else — do not guess or invent one.)

Do the steps below **in order**. Do not skip the security scan step or push before it passes.

## 1. Establish repo state

- Run `git status` and `git remote -v`. If this isn't a git repo yet, `git init` it.
- Normalize the target from `$1` into both a full clone URL and an `owner/repo` slug
  (accept either form as input).
- If `origin` already points somewhere else, tell the user and confirm before changing it
  — don't silently repoint an existing remote.

## 2. Security scan — must pass before anything is pushed

Before staging or pushing *any* file, scan the full working tree (tracked, untracked, and
about-to-be-added files — not just what's already staged) for anything that shouldn't leave
the machine:

- **Filename denylist**: `.env`, `.env.*`, `*.pem`, `*.key`, `id_rsa*`, `credentials.json`,
  `secrets.*`, `.npmrc`, `.aws/credentials`, `*.p12`, `*.pfx`, any `.git-credentials`.
- **Content patterns** (grep across all files about to be committed): AWS access keys
  (`AKIA[0-9A-Z]{16}`), GitHub tokens (`ghp_`, `gho_`, `ghs_`, `github_pat_`), private key
  blocks (`-----BEGIN ... PRIVATE KEY-----`), Slack tokens (`xox[baprs]-`), generic
  `password\s*=`, `api[_-]?key\s*[:=]`, `secret\s*[:=]`, `token\s*[:=]` assignments with a
  real-looking value (not a placeholder like `<your-key-here>` or `process.env.X`), and
  database connection strings containing embedded credentials.
- **Look for real personal data** hardcoded in source (real emails/phone numbers/addresses
  that look like actual people's data rather than obvious placeholders) — flag it even if
  it isn't a "secret" in the credential sense.

If anything matches: **stop, do not push**, list exactly what was found and where (file +
line), and ask the user how to proceed (remove it, `.gitignore` it, or confirm it's a safe
placeholder). Only continue once the user has resolved every flagged item or explicitly
confirms a specific match is a false positive.

Report to the user that the scan ran and passed before moving on, so it's visible this step
actually happened rather than being silently skipped.

## 3. README

Check whether `README.md` already has real content (not just a placeholder like `# reponame`).

- If it's missing or a placeholder, write a proper one: what the project is, how to run/open
  it, and its structure — based on what's actually in the repo, not invented features. Match
  the style already used in this project's `CLAUDE.md` if one exists.
- If it already has real content, edit it to stay accurate rather than replacing it wholesale.

## 4. Commit and push

- Stage specific files by name (never a blanket `-A`/`.` without reviewing `git status` first).
- Commit with a message describing what's being published.
- Check `gh auth status`. If `gh` isn't installed or isn't authenticated:
  - Prefer a real `gh` install/login if the user can do it interactively.
  - Otherwise, download the portable `gh` release tarball/zip for this machine's OS/arch from
    `https://api.github.com/repos/cli/cli/releases/latest` into the scratchpad, run it from
    there (no sudo/Homebrew required), and use `gh auth login --hostname github.com --git-protocol https --web`
    (device-code flow: show the user the code and the URL, wait for their confirmation).
  - A PAT the user pastes directly is also fine — never echo it back or store it outside of
    it being used for the push.
- Set `origin` to the target repo and push. If push is rejected with something like
  *"refusing to allow an OAuth App to create or update workflow ... without `workflow` scope"*,
  run `gh auth refresh -h github.com -s workflow` (another device-code confirmation) and push again.

## 5. GitHub Pages via GitHub Actions

- Add (or update) `.github/workflows/deploy-pages.yml` using the standard
  `actions/checkout` → `actions/configure-pages` → `actions/upload-pages-artifact` →
  `actions/deploy-pages` flow, triggered on push to the default branch plus
  `workflow_dispatch`, with `permissions: pages: write, id-token: write, contents: read`.
  If a Pages/deploy workflow already exists in the repo, update it in place instead of
  adding a duplicate.
- Check if Pages is already enabled: `gh api repos/<owner>/<repo>/pages`. If it 404s, create
  it with `gh api --method POST repos/<owner>/<repo>/pages -f build_type=workflow`. If it
  exists but isn't set to `build_type: workflow`, update it.
- Push the workflow file (same auth/scope handling as step 4).
- Watch the run to completion: `gh run list --limit 1` then `gh run watch <id> --exit-status`.
  If it fails, read the failing step's logs and fix the workflow rather than reporting success.
- The resulting Pages URL is `https://<owner>.github.io/<repo>/` (or a custom domain if one
  is already configured) — confirm it from the `gh api repos/<owner>/<repo>/pages` response.

## 6. GitHub About section

Once the Pages URL is live, set the repo's About panel:

```
gh repo edit <owner>/<repo> --description "<one-line description>" --homepage "<pages-url>"
```

Use a real one-line description of what the project is (not a placeholder), derived from the
README/CLAUDE.md.

## 7. Report back

Summarize for the user: what was scanned and cleared, what was pushed, the README status,
the live Pages URL, and the About section update. Call out anything that still needs their
attention (e.g., a flagged secret they chose to keep, or a workflow step that needed a manual
scope refresh).
