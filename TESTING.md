# Sync Infrastructure Testing

Validation matrix for the Overleaf ↔ git sync (`setup.sh` + pre-push hook + aliases). Run these against a throwaway Overleaf project before relying on this in production.

## Preconditions

- `bash setup.sh` completed successfully (token + project ID accepted).
- `git remote -v` shows `origin` (your primary, fetch+push) and `overleaf` (fetch+push).
- `git config alias.olp` → `pull --no-rebase overleaf master`.
- `.git/hooks/pre-push` exists and is executable (the trampoline).

## Test matrix

### 1. Basic sync (solo)

| ID  | Scenario                          | Procedure                                                                 | Expected |
|-----|-----------------------------------|---------------------------------------------------------------------------|----------|
| T1  | Overleaf-only edit                | Edit `main.tex` on web → save → wait 30s → `git olp`                      | Merge commit; web changes reflected locally. |
| T2  | Local-only edit                   | Edit locally → commit → `git push`                                        | Both remotes updated. Output shows `[pre-push]` lines. |
| T6  | Same-line conflict                | Web + local both edit the same line → local commit → `git olp`            | CONFLICT + `<<<<<<<` markers. Resolve → commit → push. |
| T9  | Uncommitted local + Overleaf edit | Local uncommitted changes; run `git olp`                                  | "local changes would be overwritten". Recover: stash → pull → pop. |
| T12 | Local committed-not-pushed + Overleaf edit | Local commit; web edit; `git push`                                | **Hook detects Overleaf rejection → aborts primary push.** Recover: `git olp` → merge → `git push`. |

### 2. Hook-specific

| ID  | Scenario                       | Procedure                                                | Expected |
|-----|--------------------------------|----------------------------------------------------------|----------|
| H1  | Atomic push success            | Both remotes clean; local commit; `git push`             | Output order: `[pre-push] pushing to Overleaf first` → Overleaf OK → primary push. |
| H2  | Atomic push blocked            | Web edit + wait; local commit; `git push`                | `[pre-push] ERROR: Overleaf push failed. GitLab push ABORTED.` **Primary remote does not advance.** |
| H3  | No `overleaf` remote           | `git remote remove overleaf`; local commit; `git push`   | Hook warns + allows primary push (fallback). Recover with `bash setup.sh`. |
| H4  | Direct `git push overleaf`     | `git push overleaf master`                                | Hook doesn't fire (only intercepts pushes to `origin`). |
| H5  | `--no-verify` bypass           | `git push --no-verify`                                   | Hook skipped, primary-only push. Emergency escape hatch. |
| H6  | `setup.sh` re-run              | `bash setup.sh` again with same inputs                   | Idempotent — config overwritten, no errors. |

### 3. Two-person collaboration (smoke test post-deploy)

| ID  | Scenario                              | Expected |
|-----|---------------------------------------|----------|
| T3  | A edits on web, B edits a different file locally | B's `git olp` auto-merges; both edits survive. |
| T4  | A and B edit the same file, different paragraphs | Auto-merge succeeds (line-based). |
| T7  | A edits, B deletes the file           | modify/delete conflict on `git olp`. Resolve manually. |
| T16 | A edits, B renames the file           | Rename detection auto-merges; otherwise add/delete. |
| T19 | Both replace the same binary (image)  | Binary conflict — pick a side with `git checkout --theirs/--ours`. |

## H-scenario detailed procedures

### H1 — Atomic push success (baseline)

```bash
git pull
git olp

echo "% H1 test $(date)" >> main.tex
git add main.tex
git commit -m "test: H1 atomic push success"
git push
```

Expected output:

```
[pre-push] pushing to Overleaf first ...
To https://git.overleaf.com/...
   xxx..yyy  master -> master
[pre-push] Overleaf OK — proceeding to GitLab.
To <your primary remote>
   xxx..yyy  master -> master
```

Verify: `git log origin/master -1` == `git log overleaf/master -1` == local HEAD.

### H2 — Atomic push blocked (most important)

```bash
git pull && git olp

# On the Overleaf web UI: edit main.tex → save → wait 30s.

echo "% H2 local $(date)" >> main.tex
git add main.tex
git commit -m "test: H2 conflict"

before=$(git rev-parse origin/master)
git push || true                 # expect failure
git fetch origin
after=$(git rev-parse origin/master)
[[ "$before" == "$after" ]] && echo "✅ H2 PASS" || echo "❌ H2 FAIL"
```

Expected hook output:

```
[pre-push] pushing to Overleaf first ...
 ! [rejected]        master -> master (fetch first)
[pre-push] ERROR: Overleaf push failed. GitLab push ABORTED to prevent drift.
```

Recovery:

```bash
git olp
# resolve, commit
git push
```

### H3 — No overleaf remote (fallback)

```bash
git remote remove overleaf
echo "% H3 $(date)" >> main.tex
git add main.tex && git commit -m "test: H3 no overleaf"
git push
```

Expected: `[pre-push] WARN — 'overleaf' remote not configured. Run bash setup.sh.` + primary push proceeds.

Recover: `bash setup.sh`.

## Cleanup

Remove test markers:

```bash
sed -i '/^% [TH][0-9]/d' main.tex
git add main.tex && git commit -m "chore: remove sync test markers"
git push
```

## Pre-deploy checklist

- [ ] T1, T2, T6, T9, T12 pass (basic sync).
- [ ] H1, H2 pass (hook atomicity — most important).
- [ ] H3 pass (hook tolerates missing overleaf remote).
- [ ] H6 pass (`setup.sh` idempotent).
- [ ] At least one of T3/T4 with a second person (real-world smoke).

H2 is the load-bearing test — passing it means "`git push` alone never drifts the two remotes."

## Known limits

1. **Overleaf auto-commit noise**: every web save creates an `Update on Overleaf.` commit. `git olp` bundles them via merge but history stays noisy. Squash via scheduled CI if you care (out of scope here).
2. **Network failure**: Overleaf down → hook fails → primary push blocked. Emergency: `git push --no-verify` (H5).
3. **`--no-verify` bypasses the hook entirely.** Treat as escape hatch only.
