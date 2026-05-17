# overleaf-git-sync

Two-way sync between an [Overleaf](https://overleaf.com) project and a git repo (GitHub, GitLab, Bitbucket — anything).

- ✅ `git push` reaches **both** remotes atomically (Overleaf first; if that fails, the primary push is aborted, so the two never drift).
- ✅ `git pull` from your primary; `git olp` pulls from Overleaf.
- ✅ Survives Overleaf's exec-bit-stripping git mirror (trampoline-installed hook).
- ✅ Zero dependencies beyond `bash` and `git ≥ 2.9`.

## Why

- Overleaf is great for real-time collaborative editing and one-click PDF compile.
- Git hosts (GitHub/GitLab) are where code, CI, and reviews live.
- You want to edit on Overleaf **and** edit locally (with your editor / Claude Code / a script) **and** never manually copy files between the two.

This repo bundles three small pieces that make that work:
- `setup.sh` — one-time installer per clone (adds remote, hook, aliases).
- `.githooks/pre-push` — atomic push hook (Overleaf first → primary).
- A few git aliases (`olp`, `olf`, `ols`) for the Overleaf side.

## Quickstart

### Option A — start a new paper repo from this template

1. Click **"Use this template"** on GitHub → create your new paper repo.
2. Clone it locally and `cd` in.
3. Get an Overleaf git token: **Overleaf → Account Settings → Git Integration → Create Token**.
4. Get your Overleaf project ID from the project URL: `https://www.overleaf.com/project/<PROJECT_ID>`.
5. Run setup:

   ```bash
   bash setup.sh
   # paste project ID
   # paste token (hidden)
   ```
6. Commit the generated `.overleaf-sync` (project ID is not a secret; it lets collaborators skip the project-ID prompt):

   ```bash
   git add .overleaf-sync
   git commit -m "chore: pin overleaf project id"
   git push
   ```

### Option B — drop into an existing paper repo

If your paper already lives in a git repo, copy two files in:

```bash
cd /path/to/your/paper-repo
curl -fsSL https://raw.githubusercontent.com/<you>/overleaf-git-sync/main/setup.sh -o setup.sh
mkdir -p .githooks
curl -fsSL https://raw.githubusercontent.com/<you>/overleaf-git-sync/main/.githooks/pre-push -o .githooks/pre-push
chmod +x setup.sh .githooks/pre-push
bash setup.sh
```

Commit `setup.sh`, `.githooks/pre-push`, and `.overleaf-sync` so teammates can run `bash setup.sh` and only need to paste their token.

## Daily commands

| Command  | What it does                                                                 |
|----------|------------------------------------------------------------------------------|
| `git pull` | Pull from your primary remote (your `pull.rebase` setting is respected).   |
| `git olp`  | Pull from Overleaf — **always merge** (rebase over Overleaf's auto-commits creates false conflicts). |
| `git push` | Push to **both** remotes. Hook pushes Overleaf first; if it fails, the primary push is aborted (atomic). |
| `git ols`  | Show the last 10 commits on Overleaf's side.                              |
| `git olf`  | Fetch (only) from Overleaf — useful before `git log overleaf/master`.     |

**Recommended routine before editing:**

```bash
git pull && git olp
```

`git pull` alone won't pick up web edits (Overleaf is a separate remote). `git olp` catches you up.

## Conflicts

Standard git 3-way merge. Two common cases:

- **`git olp` says CONFLICT** — same line edited on web and locally. Resolve markers → `git add` → `git commit`.
- **`git push` fails with `Overleaf push failed. primary remote push ABORTED`** — Overleaf moved ahead while you were committing:
  ```bash
  git olp     # merge Overleaf changes
  # resolve any conflicts, commit
  git push    # retry
  ```

## What syncs and what doesn't

| Item                                | Via git? | Notes                       |
|-------------------------------------|:--------:|-----------------------------|
| `.tex`, `.bib`, `.sty`, images      | ✅       |                             |
| File add / delete / rename          | ✅       |                             |
| Overleaf Comments (sticky notes)    | ❌       | Web-only                    |
| Track Changes                       | ❌       | Web-only (Premium)          |
| Chat / live cursors                 | ❌       | Web-only                    |

Reviews / discussion live in Overleaf. In-line `% TODO:` comments live in `.tex`.

## How the atomic push works

A naive setup uses git's multi-push URL (`git remote set-url --add --push origin <overleaf>`). That is **not atomic** — if the Overleaf push fails (non-fast-forward), the primary push still proceeds, and the two remotes drift.

This repo's `pre-push` hook intercepts pushes to `origin`, attempts the Overleaf push first, and exits non-zero on failure — which aborts the rest of the `git push`. The result: either both remotes advance or neither does.

The hook lives in `.githooks/pre-push` (tracked, version-controlled). But Overleaf's git mirror **strips Unix mode bits** on every round-trip, so a hook installed via `core.hooksPath=.githooks` silently stops working the first time Overleaf saves. Workaround: `setup.sh` installs a tiny trampoline at `.git/hooks/pre-push` (untracked, stays executable) that `exec`s the tracked hook via `bash`.

## Files

| File                  | Purpose                                                  |
|-----------------------|----------------------------------------------------------|
| `setup.sh`            | One-shot installer (run per clone).                      |
| `.githooks/pre-push`  | Atomic-push hook. Read-only at runtime.                  |
| `.overleaf-sync`      | Persisted project ID. Tracked. Not a secret.             |
| [TESTING.md](TESTING.md) | Validation matrix to verify your install end-to-end. |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Common errors and recovery.            |

## License

MIT. See [LICENSE](LICENSE).
