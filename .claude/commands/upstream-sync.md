# Upstream Sync

Sync this fork with the upstream garrytan/gstack repo, resolve conflicts, push, and install locally.

## Step 0: Ensure clean working tree

```bash
git status --porcelain
```

If there are uncommitted changes, ask the user whether to stash them or abort.

## Step 1: Fetch upstream

```bash
git remote add upstream https://github.com/garrytan/gstack.git 2>/dev/null || true
git fetch upstream main
```

## Step 2: Record current version and preview incoming changes

Save the current version before merging so we can diff changelogs accurately:

```bash
cat VERSION
```

Remember this as OLD_VERSION.

Show what's new from upstream before merging:

```bash
git log --oneline HEAD..upstream/main
```

If there are no new commits, tell the user "Already up to date with upstream" and skip to Step 7.

## Step 3: Merge upstream

```bash
git merge upstream/main --no-edit
```

If the merge succeeds with no conflicts, skip to Step 4 (gbrain pin audit).

### Conflict resolution

If there are merge conflicts, resolve them using these rules from CLAUDE.md:

1. **Generated SKILL.md files:** accept either side, then run `bun run gen:skill-docs` to regenerate.
2. **office-hours/SKILL.md.tmpl:** accept upstream's version, then re-remove Phase 4.5 (Founder Signal Synthesis), Phase 6 (Handoff — Founder Discovery), and YC branding.
3. **URL-only files** (README, bin/gstack-update-check, gstack-upgrade/SKILL.md.tmpl): keep our `donovan-yohan/gstack-adfree` URLs.
4. **gbrain repo URL refs:** keep `donovan-yohan/kbrain` everywhere — never accept upstream's `garrytan/gbrain` URL. See Step 4 for the audit.
5. **Everything else:** review the conflict and resolve sensibly — prefer upstream's logic changes, keep our fork-specific customizations.

After resolving all conflicts:
```bash
bun run gen:skill-docs
git add -A
git commit --no-edit
```

## Step 4: Verify gbrain refs still pin to kbrain fork

This fork repoints gbrain federation at `donovan-yohan/kbrain`. Upstream commits can re-introduce the original `garrytan/gbrain` URL (new docs, new bin scripts, regenerated templates). Always audit after merging.

```bash
# Should return NOTHING. Any output means a ref needs re-pinning.
grep -rEn "https?://github\.com/garrytan/gbrain" \
  --include="*.md" --include="*.ts" --include="*.tmpl" --include="*.json" --include="*.sh" \
  . 2>/dev/null | grep -v node_modules | grep -v "\.gbrain/"
```

If any matches appear, edit each file to replace `garrytan/gbrain` → `donovan-yohan/kbrain` (preserve `.git` suffix where present). Then regenerate and re-test:

```bash
bun run gen:skill-docs
bun test test/gbrain-detect-install.test.ts
git add -A
git commit -m "chore: re-pin gbrain refs at donovan-yohan/kbrain after upstream sync"
```

The canonical pin sites (six files as of v1.26.0.0):
- `bin/gstack-gbrain-install` — `GBRAIN_REPO_URL=`
- `setup-gbrain/SKILL.md.tmpl` + regenerated `setup-gbrain/SKILL.md`
- `README.md` + `USING_GBRAIN_WITH_GSTACK.md` — GBrain link
- `test/gbrain-detect-install.test.ts` — clone-URL assertion

Note: `PINNED_COMMIT` in `bin/gstack-gbrain-install` (currently `08b3698e...`, gbrain v0.18.2) is unchanged by upstream syncs. The kbrain fork must contain that SHA or install fails. Bump the pin only if kbrain diverges.

## Step 5: Validate

Run the free test suite to make sure nothing broke:

```bash
bun run gen:skill-docs
bun test test/skill-validation.test.ts
```

If tests fail, fix the issues before proceeding.

## Step 6: Push to origin

```bash
git push origin main
```

## Step 7: Install locally via /gstack-upgrade

Run the `/gstack-upgrade` skill to pull the latest from our fork into the local skill install at `~/.claude/skills/gstack/`.

## Step 8: Summary

Read CHANGELOG.md and find ALL version entries between OLD_VERSION (from Step 2) and
the current version after merge. This may span multiple releases if we haven't synced
in a while.

Provide a summary with:
- Version jump (e.g. "v0.15.2.0 → v0.17.0.0")
- How many commits were merged from upstream
- **All changes across every version since OLD_VERSION**, grouped by theme (not by version).
  Read each `## [x.y.z.0]` section in CHANGELOG.md that's newer than OLD_VERSION.
- Any conflicts that were resolved and how
- Whether tests passed
- Whether the local install was updated
