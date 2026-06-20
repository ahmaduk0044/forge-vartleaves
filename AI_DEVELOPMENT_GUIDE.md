# AI Development Guide — FORGE by Vartleaves

**Status: Permanent, binding rules for any AI assistant (or human) making changes to this codebase.**

These aren't aspirational best practices — every rule below exists because of a specific, real incident in this project's history. They are listed with that history so a future session understands *why*, not just *what*.

---

## The 8 Rules

### 1. Never rewrite working code
Patch the smallest possible surface. If a function works, leave its working parts alone — add to it, wrap it, or call it from somewhere new. Don't restructure something that wasn't broken as a side effect of fixing something that was.

*Why this rule exists:* every successful change in this project's recent history (Search & Filters, Reports Module, Backup reversibility, Health Monitor, security patches) was done by adding new functions and making small, surgical edits to existing ones — never by restructuring a working module. The System Status tab is the clearest example: the Health Monitor was added by wrapping the existing render function in a thin async dispatcher, with the original stat-cards code moved verbatim, untouched, into the new structure.

### 2. Always create a backup before changes
Before touching anything, capture the current good state somewhere durable.

*Why this rule exists:* a fully-built, fully-tested 9-item feature batch was lost entirely because it only existed in a temporary sandbox container that got discarded before deployment. Nothing was recoverable. This is the single most expensive mistake in this project's history, and every backup-related feature built since (the 3-way source backup policy, `ForgeBackupService`'s auto-snapshot-before-restore) exists to make sure it can't happen again.

### 3. Always create a Git tag
Before starting risky work, tag the current commit. After finishing a milestone, tag the new one.

*Why this rule exists:* tags are what make rule 5 (rollback) possible at all — a rollback target only exists if it was tagged. This project uses two conventions: `PRE_*` tags before risky work (`PRE_FINAL_AUDIT`, `PRE_PRODUCTION_LOCK`), and `vX.Y-description` tags for milestones (`v1.4-backup-reversibility`, `v1.6-security-hardening`). Both are lightweight — a single API call — so there's no excuse to skip this.

### 4. Always test before deploy
Run the full verification sequence before anything goes live: syntax-check every script block individually, regression-test the actual new logic with real inputs (not just "it parses"), and diff the working copy against the last deployed commit to confirm the change is exactly what was intended — nothing more.

*Why this rule exists:* a past deploy was once claimed successful without verification, and it wasn't — wasting time tracking down the gap. Since then, every deploy in this project follows the same discipline, documented in full in `DEPLOYMENT_GUIDE.md`. One specific gotcha this testing caught: a JS string containing a literal `</script>` tag silently truncated an entire script block in the browser, even though it parsed fine in isolation — only a structural read of the code caught it, not the automated checks. Tests reduce risk; they don't replace a careful read.

### 5. Always roll back on failure
If a deploy doesn't verify cleanly — if the re-fetched live file doesn't match byte-for-byte, or a real-browser check (not just the Node test harness) turns something up — the safety tag from rule 3 means reverting is a normal, calm action, not an emergency. Push the previous good commit's content back to `main`, exactly like any other deploy.

*Why this rule exists:* a rollback is only routine if it was planned for in advance. Treating "go back to the last good version" as a normal, well-rehearsed action (not a crisis) is what tags and backups are *for*.

### 6. Never create duplicate architecture
If a capability already exists (a service, a data store, a UI pattern), extend it. Don't build a second, parallel version because it's easier than understanding the first one.

*Why this rule exists:* this project has one pattern for cross-cutting services — a self-contained IIFE module with an `init(db)` call and a small public API (`ForgeFirebaseService`, `ForgePermissionService`, `ForgeNotificationService`, `ForgeBackupService`, `ForgeHealthMonitor`, `ForgeSessionGuard`). Every one of these was added without touching the others, and none of them duplicate what another already does. The one real architectural inconsistency on record — the Backup module claiming to cover a `tasks` Firestore collection that doesn't actually exist (tasks are local-storage only) — happened because two pieces of the system were built at different times without being reconciled. That's the failure mode this rule prevents.

### 7. Never remove existing functionality
A patch's diff should be additive or surgical. If something existing disappears from the diff that wasn't the explicit target of the fix, stop and investigate before deploying.

*Why this rule exists:* every deploy in this project is verified with an explicit diff-scope check before pushing — confirming line-for-line that nothing outside the intended change was touched. This has been checked and confirmed clean on every single deploy so far, including ones that touched authentication, role logic, and rendering code shared across multiple tabs.

### 8. Prefer extension over replacement
When in doubt between modifying something in place and adding something new alongside it, add. A new function, a new hook into an existing callback, a new service — these carry far less risk than rewriting something that already works.

*Why this rule exists:* this is *how* rules 1, 6, and 7 actually get followed in practice. The session security work is the clearest example: inactivity timeout and role re-verification were added as a brand-new `ForgeSessionGuard` module, integrated into the existing sign-in/sign-out flow with exactly two single-line additions (`.start()` on sign-in, `.stop()` on sign-out) — the sign-in and sign-out functions themselves were never touched.

---

## The standard workflow (how these rules get applied together)

This is the concrete, repeatable sequence used for every change in this project:

1. **Verify current state.** Confirm the live commit matches what's expected before touching anything — don't assume, check.
2. **Tag a safety checkpoint** (Rule 3) before starting.
3. **Pull the exact live file**, pinned to that commit SHA — never branch-ref, which can be momentarily stale.
4. **Make the smallest change that satisfies the request** (Rules 1, 6, 8).
5. **Syntax-check** every script block individually (Rule 4).
6. **Regression-test** the actual new/changed logic with real inputs, not just confirming it parses (Rule 4).
7. **Diff the result against the last deployed commit** — confirm the change is in scope and nothing existing was removed (Rule 7).
8. **Deploy.**
9. **Re-fetch immediately, by exact commit SHA, and diff byte-for-byte against the working copy.** A deploy isn't "done" until this is zero lines different.
10. **Tag the new milestone** (Rule 3).
11. **If step 9 fails, or a human later reports something broken: roll back** (Rule 5) using the tag from step 2, then re-investigate calmly rather than patching forward under pressure.

## A known limitation of this workflow, stated honestly

Steps 5–7 run in a Node.js test harness, not a real browser. This has already caught real bugs (e.g. silent script-tag truncation) but has also produced at least one **false failure** caused by a Node `vm` quirk — top-level `const` bindings inside a script can't be reliably overridden from outside that same `vm` context, which can make a perfectly correct piece of code look like it failed a mock-based test. When this happens, the fix is to extract the specific logic into an isolated, standalone function and test *that* directly — not to weaken the test or skip verification. Automated testing here is a strong signal, not a perfect one; a human confirming real-browser behavior is still the final check before a feature is considered fully done.

## What these rules do *not* mean

- They don't mean every request must be minimized to the point of being useless. "Minimal" means *no larger than the request requires* — a genuine new feature (like the Health Monitor) is still a substantial, complete piece of work; it's just built additively rather than by disturbing what already exists.
- They don't mean skipping honest disclosure of limitations. If something can't be verified (e.g. actual deployed Firestore Security Rules, which live outside this codebase entirely and aren't visible from here), say so plainly rather than presenting an assumption as a verified fact.
- They don't mean refusing to ever recommend a larger change. If a real architectural gap is found (e.g. `tasks` claiming Firestore coverage it doesn't have), it should be surfaced clearly — just not silently "fixed" by rewriting things beyond what was asked.
