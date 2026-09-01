# Context for whoever (human or AI) picks this project up next

Written assuming no memory of any prior conversation about this repo. The
README documents what this is and how to install it in a repo; this file
documents why it exists, what it replaced, and the mistakes that shaped it.

---

## Why this exists

This is the extracted, documented version of a workflow originally built
directly inside two other repos —
[nas-dashboard](https://github.com/codemastervy/nas-dashboard) and
[smb-web-client](https://github.com/codemastervy/smb-web-client) — where it
runs in production. It replaced a **continuous Dependabot + auto-merge**
setup that caused a real incident on both of those repos, which is worth
understanding in full before touching this workflow, because every design
choice here is a direct reaction to a specific way that first attempt broke.

## The incident that led here

Dependabot was configured, and a workflow was added that called
`gh pr merge --auto` on every PR Dependabot opened. Branch protection
requiring the CI checks to pass was configured **after** auto-merge was
already live — a plain sequencing mistake. The consequence: GitHub's
auto-merge has nothing to hold a merge open *for* when no check is marked
required, so every Dependabot PR — including a bump that put `react-dom` on
a major version ahead of `react` itself, a genuine breaking mismatch —
merged to `main` completely unvetted, within minutes of the workflow being
enabled. Disabling the repo-level "allow auto-merge" setting did **not**
retroactively stop PRs that already had auto-merge armed on them
individually; several more merged before that was understood. All of it was
reverted with `git revert` (not a history rewrite), and only then was branch
protection actually configured and *verified* — by pushing a deliberately
broken dependency and confirming GitHub itself refused the merge with "3 of
3 required status checks are expected," not just trusting that the
configuration looked right.

**The lesson this workflow is built around:** auto-merge is only as safe as
the required checks it's gated on, and the gating has to be *configured and
proven* before the automation that depends on it goes live — not the other
way around, and not assumed to work from reading the settings page.

## Why this shape, specifically

**Every 2 months, not continuous.** The original Dependabot setup opened a
rolling stream of individual PRs — one per outdated package, whenever it
happened to notice. That's *more* individually-vettable, but it's also what
made the incident possible: many small, independently-merging PRs, each too
small to seem worth a second look, collectively landing a breaking change.
This workflow deliberately does the opposite: **one consolidated PR**,
bundling everything outdated at once, on a schedule the person approving it
actually controls.

**A GitHub Environment for approval, not a custom email service.** The
alternative considered was a fully custom, branded email via a third-party
mail provider (SendGrid/Mailgun) — rejected because it needs API keys/SMTP
credentials to be provisioned and managed, for a cosmetic improvement over
what GitHub already does for free: a job targeting an Environment with a
required reviewer genuinely pauses, and GitHub's own notification system
emails the reviewer a link to a real Approve/Reject action on
`github.com`. No secrets, no third-party dependency, and it was possible to
verify the whole mechanism end to end (see below) without needing to wait
for an actual scheduled run or receive a real email.

**Reuses the target repo's own CI rather than testing anything itself.**
This workflow has no opinion about what "passing" means for your project —
it just needs your CI's check names to exist and be marked required on
`main`. This keeps it genuinely portable across repos with different stacks,
at the cost of the setup needing that one manual, easy-to-forget step
(README step 4) — which is precisely the step whose absence caused the
original incident, so it's called out with extra emphasis rather than
buried in a numbered list.

## A bug found after shipping, by actually using it as a reference repo

This repo originally shipped with the workflow file live at
`.github/workflows/bimonthly-dependency-update.yml` — the same path where
it needs to live in a *consuming* repo to actually run. The oversight: that
means it was also live **here**, in the repo that has no
`backend/requirements.txt`, no `frontend/package.json`, no `Dockerfile` —
nothing for its own check steps to find. Left as shipped, it would
eventually have fired on its own schedule against this repo and failed,
emailing about a phantom problem in a repo that was never meant to run it.
Fixed by moving the workflow file to `examples/`, inert until someone
copies it into a repo that actually has dependencies to check — see the
README's copy command for the current (corrected) path. Worth remembering
if you ever add a *second* reference file here: anything under
`.github/workflows/` in this repo runs against this repo.

## Verified, not just built

Every mechanism here was actually exercised via `workflow_dispatch` and the
GitHub API, not just reasoned about from the YAML:

- **The pause is real**, confirmed via the API showing the `await-approval`
  job in a genuine `waiting` state, not just "should pause per the docs."
- **Approving it (via the same
  `POST /repos/{owner}/{repo}/actions/runs/{id}/pending_deployments`
  endpoint GitHub's own UI button calls) resumes the workflow correctly**,
  and the resulting PR contains exactly the version bumps the check step
  found.
- **A genuinely broken update — a real major TypeScript bump that produces
  two real type errors — correctly leaves the PR sitting `BLOCKED`,
  unmerged**, confirmed via `mergeStateStatus` on the PR, not assumed from
  "CI failed so it probably didn't merge."
- **Commits pushed by the workflow's own `GITHUB_TOKEN` do not trigger
  other workflows** — a deliberate GitHub Actions loop-prevention rule that
  is easy to not know about until it silently breaks something. The PR this
  workflow opens would never have gotten CI to run on it at all — sitting
  open forever, `auto-merge` armed but permanently waiting on checks that
  could never appear — without the explicit `gh workflow run ci.yml --ref
  "$BRANCH"` call right after pushing. Found by watching a real PR sit with
  zero check-runs before this was added.
- **A separate, easy-to-miss repo setting** ("Allow GitHub Actions to
  create and approve pull requests," off by default) blocks `gh pr create`
  outright with a permission error distinct from anything to do with
  required checks — found the same way, by watching the `apply` job
  actually fail.

## What this doesn't do, on purpose

- It doesn't try to detect *how risky* an update is (patch vs. minor vs.
  major) — everything found gets bundled into the same PR, and the gate is
  entirely "does your CI pass," not a heuristic about version numbers.
- It doesn't retry or auto-fix a failing update. A broken bump just sits
  open, unmerged, for a human to look at — matching the explicit design
  goal ("check whether something breaks before it updates") rather than
  attempting to also *fix* what breaks.
- It doesn't manage its own required-check configuration. Branch protection
  is set up once, by hand, outside this workflow — deliberately, since
  letting the automation configure its own gate is close to the shape of
  the original incident.

## If you're adapting this to a new repo

Read the README's "Gotchas found by actually running this" section before
you assume it'll just work by copying the two files — every gotcha listed
there broke a real run at least once, and none of them are obvious from
reading the workflow YAML alone.
