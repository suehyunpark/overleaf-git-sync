# Troubleshooting

## Token rejected during `setup.sh`

> `error: token missing or doesn't start with 'olp_'`

Overleaf tokens always start with `olp_`. If yours doesn't, you copied the wrong string — re-create at **Account Settings → Git Integration → Create Token** and paste the **whole** token (it's long).

## `git push` fails with `Authentication failed`

The Overleaf token expired or was revoked. Re-run:

```bash
bash setup.sh
```

with a fresh token. (The script overwrites the `overleaf` remote URL.)

## Hook isn't firing — `git push` only updates origin

Symptom: pushing only goes to origin; no `[pre-push]` lines in the output.

Most likely cause: Overleaf round-tripped the repo and stripped the exec bit off `.githooks/pre-push` (this is exactly what the trampoline is supposed to handle). Check:

```bash
ls -l .git/hooks/pre-push
```

- If the file doesn't exist or isn't executable → re-run `bash setup.sh` (idempotent, installs the trampoline).
- If `.git/hooks/pre-push` exists and is executable but doesn't run → make sure you're on git ≥ 2.9 and that no other `core.hooksPath` is set:

  ```bash
  git config --get core.hooksPath   # should be empty
  git --version                      # should be ≥ 2.9
  ```

## `git push --no-verify` — should I use it?

Only as an escape hatch:

- Overleaf is down and you need to ship to the primary remote anyway.
- You know you're intentionally diverging the two remotes (rare).

After using `--no-verify`, run `git push overleaf master` (or wait for Overleaf to come back) to re-sync.

## `git olp` keeps producing huge merge commits

Overleaf creates one commit per web save (`Update on Overleaf.`). When you `git olp`, those bundle into a single merge commit on your side. The "size" you see is in the log, not in the diff — the merge commit itself usually has zero diff against the latest content. Ignore it.

If the history really bothers you, run periodic squashes from a maintainer branch. Out of scope for this tool.

## Conflict shows up on a file I never touched

That's an Overleaf auto-save vs. local-commit race. The `main.tex` mentioned in the conflict has changes from both sides. Standard resolution:

```bash
git status                          # see conflicted files
$EDITOR <file>                      # resolve <<<<<<< / >>>>>>> markers
git add <file>
git commit                          # finishes the merge
git push
```

## "I cloned the repo on a new machine — how do I sync?"

If `.overleaf-sync` is committed (it should be), just:

```bash
bash setup.sh
# enter token only — project ID auto-loaded
```

If `.overleaf-sync` isn't committed yet, run `bash setup.sh` with the project ID and commit the generated file.

## Removing the sync from a repo

```bash
git remote remove overleaf
rm .git/hooks/pre-push
git config --unset alias.olp
git config --unset alias.olf
git config --unset alias.ols
# optionally: rm setup.sh .githooks/pre-push .overleaf-sync
```
