# frogbot-central

Proof of concept for **centralized Frogbot** — one repository per organization
that scans every *other* repository, instead of committing a workflow file into
each one.

Target repositories contain **no Frogbot workflow, no secrets and no
configuration**. Nothing is installed in them.

## CVE Auto-Fix (centralized)

Target repositories must **not** contain `jfrog-auto-fix.yml`. The only auto-fix workflow is this repo's `.github/workflows/jfrog-auto-fix.yml`.

XSC `fix_component` → central-integration `repository_dispatch` `jfrog-auto-fix` on **this** repo, with:

| payload | meaning |
|---|---|
| `target_owner` / `target_repo` | git repo of the scanned artifact (`vcs.url`) |
| `component_name` | e.g. `com.fasterxml.jackson.core:jackson-databind` |
| `affected_version` / `fix_version` | from Xray remediations |
| `branch` | `vcs.branch` |

The action must checkout `repository: target_owner/target_repo`. Default `GITHUB_REPOSITORY` is `frogbot-central`, so without that input the SHA/PR land on the dispatcher.

Org secrets: `JF_GIT_TOKEN` (or `AUTO_FIX_TOKEN`), `JF_URL`, `JF_ACCESS_TOKEN`.

Full wiring (GitHub App, ngrok → `:8046`, tenant map, JAS, artifact VCS props) is listed on the companion draft PRs. Not for review.

## Status: working

| Capability | Result |
|---|---|
| `scan-pull-request` on another repo | ✅ resolved source/target from the PR number alone, downloaded both branches via API, found 21 SCA vulnerabilities, posted PR comments |
| `scan-repository` on another repo | ✅ 31 SCA vulnerabilities, opened a real fix PR on the target |
| Fan-out across the org | ✅ matrix over all non-archived repos, one dispatch |
| Merge gating | ✅ commit status on the target PR head; blocked → clean proven end to end |

Verified against `centralized-frogbot-poc/demo-vulnerable-npm`, which has no
Frogbot workflow of its own.

## Workflows

| Workflow | Trigger | Covers |
|---|---|---|
| `scan-repository.yml` | `repository_dispatch` (`jfrog-scan-repository`) + daily cron + manual dispatch | single target from payload, or fans out across every non-archived repo |
| `scan-pull-request.yml` | `repository_dispatch` (`jfrog-scan-pull-request`) + manual dispatch | one PR in one target repo (`target_repo`, `pr_number`), plus commit status gate |
| `jfrog-auto-fix.yml` | `repository_dispatch` (`jfrog-auto-fix`) | centralized auto-fix against `target_owner`/`target_repo` (POC) |

`repository_dispatch` is the production-shaped path (XSC / App webhook → central workflows). Manual `workflow_dispatch` remains for POC testing.

Organization secrets: `JF_URL`, `JF_ACCESS_TOKEN`, `JF_GIT_TOKEN`. Auto-fix also uses `AUTO_FIX_TOKEN` (falls back to `JF_GIT_TOKEN`).

## What we learned

### 1. Frogbot's Go core needs no changes — the Action wrapper does

`scan-pull-request` is given only the repo and the PR number; it calls
`GetPullRequestByID` to resolve the source and target owner/repo/branch, then
downloads both branches itself. There is no reference to `GITHUB_EVENT_PATH` or
`GITHUB_REPOSITORY` anywhere in the Frogbot Go codebase.

But **`jfrog/frogbot@v3` cannot be used for this.** `setFrogbotEnv()` in
`action/src/utils.ts` overwrites three variables unconditionally with the
*running* repository's context:

```ts
core.exportVariable('JF_GIT_OWNER', githubContext.repo.owner);              // overwrites
core.exportVariable('JF_GIT_REPO', ...);                                    // overwrites
core.exportVariable('JF_GIT_PULL_REQUEST_ID', githubContext.issue.number);  // overwrites
```

In the same function `JF_GIT_TOKEN`, `JF_GIT_API_ENDPOINT` and
`JF_GIT_SERVER_URL` *are* guarded with `if (!process.env.X)`. Only these three
are not. The symptom is silent: the run scans the central repo instead of the
target, reports success, finds nothing, and emits `"manifests": null`.

These workflows therefore invoke the `frogbot` binary directly. The upstream fix
is three `if (!process.env...)` guards — patch in the POC notes
(`frogbot-action-respect-env.patch`), ~14 lines.

### 2. `scan-repository` does not clone the target — it copies the cwd

`switchToTempWorkingDir()` does `CopyDir(os.Getwd(), tempWd)` and then
`SetCurrentWdAsLocalGitRepository()`. So `scan-repository` requires the target to
already be checked out in the working directory; it produced an empty SBOM until
we added an explicit `actions/checkout` of the **target** repo.

This is the one place the two commands genuinely differ:

| | Gets its source by | Needs a checkout? |
|---|---|---|
| `scan-pull-request` | `DownloadRepoToTempDir` — VCS API | **no** |
| `scan-repository` | copies the current working directory | **yes** |

### 3. Frogbot produces no check run or commit status — so nothing gates a merge

Per-repo Frogbot gating works only because the *workflow run in the target repo*
produces a check. Run centrally, that check lands on the central repo's commit,
not the PR head — so gating silently disappears. Frogbot posts comments and
uploads SARIF, but creates no check run or status: after a successful scan the PR
head had `total_count: 0` check runs and no statuses.

`scan-pull-request.yml` adds it via the Statuses API. (The Checks API is
restricted to GitHub Apps, so a PAT cannot use it; commit statuses work with a
PAT and are equally valid as required checks. Production should use check runs
from the App for the richer 64KB markdown output and file/line annotations.)

### 4. A required status that is never posted blocks the PR forever — proven

With a ruleset requiring `JFrog Frogbot / scan-pull-request`:

| PR | Status | `mergeable_state` |
|---|---|---|
| #1 — scanned | `success` | `clean` |
| #2 — never scanned | `pending` | **`blocked`** |

Dispatching the scan for #2 moved it to `clean`. Two things follow:

- Coverage *is* enforced from day one — a commit the status has never been posted
  on is gated, not silently skipped. That is the desired behaviour.
- The liveness risk is real, not theoretical. Whatever dispatches scans must
  cover **every** PR, and the workflow must **always** resolve the status —
  hence the `if: always()` publish step and the up-front `pending`.

The trap this surfaced: **PR #2 was Frogbot's own fix PR, blocked by Frogbot's
own gate**, because nothing dispatched a scan for it. Auto-fix PRs would be
unmergeable in production unless the dispatch path covers PRs that Frogbot opens.

### 5. Organization-level rulesets require GitHub Team

`POST /orgs/{org}/rulesets` returns `403 Upgrade to GitHub Team to enable this
feature.` on a Free org. The org-wide gating mechanism is plan-gated at **Team**,
not only Enterprise. Repository-level rulesets work on Free for public repos,
which is what this POC used.

### 6. Minor findings

- The GitHub SBOM-snapshot upload uses the **running** repo's `sha`/`ref`, so it
  reported the central repo's commit — a second, separate leak of the running
  repo's context, in the snapshot path rather than the scan path.
- Snapshot upload also fails with `404 Dependency graph is disabled` until the
  target repo enables it.
- The gate is **vacuous without a watch/policy**: `Found 0 active watches` means
  Frogbot exits 0 even with 31 vulnerabilities, so the status goes green. Real
  gating needs a watch on the git-repo resource.
- The JFrog tenant's default config profile in use was a leftover from unrelated
  work (`profile-browser-repro-...`), which affects which scanners run.

## Remaining gaps

1. **Stored cross-repo token.** `JF_GIT_TOKEN` is a stored PAT with broad scope,
   and PR comments are attributed to a human user. Production should mint a
   1-hour installation token scoped to the single target repo, fetched at runtime
   (job presents its Actions OIDC token to XSC, which validates the `sub` claim
   and returns a scoped token) so nothing sits at rest in the customer's org.
2. **Dispatch wiring.** Workflows now accept `repository_dispatch` (POC stand-in
   for the App webhook → XSC → central run path). Must still cover Frogbot's own
   fix PRs (see finding 4). Auto-fix is centralized here as a POC.
3. **Splitting scan from fix.** `scan-repository` opens fix PRs, so it is not a
   scan-only path — it runs package managers and does depend on the build
   environment. A purely centralized detective capability needs the split.
4. **No-lockfile repos** cannot be scanned by V3 static SCA at all. They need a
   first-class coverage state and a non-blocking terminal status.
5. **Migration.** Repos that already carry a committed Frogbot workflow would be
   scanned twice. Needs detection and suppression of one path.
