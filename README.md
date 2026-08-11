# frogbot-central

Proof of concept for **centralized Frogbot** — one repository per organization
that scans every *other* repository, instead of committing a workflow file into
each one.

Target repositories contain **no Frogbot workflow, no secrets and no
configuration**. Nothing is installed in them.

## Frogbot's core needs no changes. The GitHub Action wrapper does.

Frogbot's Go core was never coupled to running inside the repository it scans.
It takes its context from environment variables and fetches the source itself:

| What it needs | Where it comes from here |
|---|---|
| Which repo | `JF_GIT_OWNER` / `JF_GIT_REPO` |
| Which branch | `JF_GIT_BASE_BRANCH` |
| Which pull request | `JF_GIT_PULL_REQUEST_ID` |
| The source code | Frogbot clones/downloads it itself — `actions/checkout` is never used |

`scan-pull-request` is given only the repo and the PR number; it calls
`GetPullRequestByID` to resolve the source and target owner, repo and branch,
then downloads both branches into temp dirs. There is no reference to
`GITHUB_EVENT_PATH` or `GITHUB_REPOSITORY` anywhere in the Frogbot Go codebase.

**But `jfrog/frogbot@v3` cannot be used for this.** The action's
`setFrogbotEnv()` (`action/src/utils.ts`) overwrites three variables
unconditionally, with the *running* repository's context:

```ts
core.exportVariable('JF_GIT_OWNER', githubContext.repo.owner);          // overwrites
core.exportVariable('JF_GIT_REPO', /* githubContext.repo.repo */);      // overwrites
core.exportVariable('JF_GIT_PULL_REQUEST_ID', githubContext.issue.number); // overwrites
```

Note the inconsistency: in the same function `JF_GIT_TOKEN`,
`JF_GIT_API_ENDPOINT` and `JF_GIT_SERVER_URL` *are* guarded with
`if (!process.env.X)`. Only these three are not. The observable symptom is that
a centralized run silently scans the central repo instead of the target — it
reports success, finds nothing, and emits `"manifests": null`.

So these workflows **invoke the `frogbot` binary directly**. The upstream fix is
to guard those three the same way the other three already are; see
`../frogbot-action-respect-env.patch` in the POC notes.

## Workflows

| Workflow | Trigger | Covers |
|---|---|---|
| `scan-repository.yml` | daily cron + manual dispatch | fans out across every non-archived repo in the org via a matrix |
| `scan-pull-request.yml` | manual dispatch (`target_repo`, `pr_number`) | one PR in one target repo |

## Configuration

Organization secrets:

| Secret | Purpose |
|---|---|
| `JF_URL` | JFrog platform URL |
| `JF_ACCESS_TOKEN` | JFrog platform access token |
| `JF_GIT_TOKEN` | GitHub token with access to the **target** repositories |

`JF_GIT_TOKEN` is the one thing with no free equivalent here. A workflow's
built-in `GITHUB_TOKEN` is scoped to the repository it runs in, so it cannot read
a target repo or comment on its pull requests. In production this should be a
short-lived GitHub App installation token, minted per run and scoped to the
single target repo — not a stored credential.

## Known gaps in this POC

1. **Stored cross-repo token.** `JF_GIT_TOKEN` is a stored PAT. Production should
   mint a 1-hour installation token scoped to one repo, fetched at runtime by the
   job (e.g. the job presents its Actions OIDC token to XSC, which validates the
   `sub` claim and returns a scoped token). Nothing then sits at rest in the org.
2. **Dispatch is manual.** In production the `pull_request` App webhook drives
   this; here the PR scan is dispatched by hand.
3. **No merge gate.** Making this block merges needs an organization ruleset
   requiring a status check from the JFrog App — a separate mechanism from these
   workflows, and the next thing to prove.
4. **`scan-repository` also opens fix PRs.** Not a scan-only path, so it runs
   package managers and does depend on the build environment. Splitting scan from
   fix is a prerequisite for a purely centralized detective capability.
5. **Repos with no committed lock file** cannot be scanned by V3 static SCA at
   all. They need to be a first-class coverage state rather than a silent pass.
