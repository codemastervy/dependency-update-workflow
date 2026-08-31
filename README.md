# dependency-update-workflow

A GitHub Actions workflow that checks a repo's dependencies (pip, npm, Docker
base images) every 2 months, **pauses for your approval**, and only then
applies the updates — through a normal pull request that still has to pass
your test suite before it can merge.

No Dependabot, no third-party email service, no credentials to manage. The
approval step is a real GitHub Environment with a required reviewer; GitHub's
own notification system emails you a link to an Approve/Reject button.

Extracted from [nas-dashboard](https://github.com/codemastervy/nas-dashboard)
and [smb-web-client](https://github.com/codemastervy/smb-web-client), where
it's used in production. This repo is the documented reference copy — drop
the two files into another repo to adopt it there.

---

## What it does

```
schedule (every 2 months, or run manually)
        │
        ▼
   check for updates ──── nothing outdated? ──▶ done, no email sent
        │
        │ something's outdated
        ▼
  PAUSE — GitHub Environment requires your approval
   (this is what triggers the email)
        │
        │ you click Approve
        ▼
  apply the bumps, open ONE pull request
        │
        ▼
  your existing CI must pass (backend tests, frontend build,
  docker build, whatever your repo already requires)
        │
   ┌────┴────┐
  pass       fail
   │           │
   ▼           ▼
 merges    stays open, unmerged, for you to look at
```

If a bump breaks something, it does not get force-fixed or reverted — it
simply never merges. That's the point.

---

## Setup (per repo you want this in)

### 1. Copy the two pieces in

```bash
cp examples/bimonthly-dependency-update.yml  <target-repo>/.github/workflows/bimonthly-dependency-update.yml
cp -r scripts/                                <target-repo>/scripts/
```

The workflow file lives under `examples/`, not `.github/workflows/`, on
purpose — this repo has no dependency files of its own to check, so a live
copy sitting in `.github/workflows/` here would eventually fire on its
schedule and fail, emailing you about it. It only starts actually running
once you copy it into a repo that has dependencies to check.

### 2. Make sure the paths match your repo

The scripts assume:

| Path | Used for |
|---|---|
| `backend/requirements.txt` | pip packages, pinned as `name==version` |
| `frontend/package.json` + `frontend/package-lock.json` | npm packages |
| `Dockerfile` (repo root) | `FROM python:...` / `FROM node:...` base images |

If your repo is laid out differently, edit the paths in
`scripts/apply_pip_updates.py`, `scripts/apply_docker_update.py`, and the
`check` job's working directories in the workflow file.

### 3. Your repo needs its own CI workflow

This workflow doesn't test anything itself — it relies on **your** CI to
gate the merge. At minimum, whatever check names you use need to be listed
as required status checks on `main` (step 4). The reference repos use:

```yaml
jobs:
  backend-tests: { ... }
  frontend-build: { ... }
  docker-build: { ... }
```

### 4. Require those checks on `main`, and let Actions open PRs

Both of these are one-time repo settings, not something this workflow can
configure for you:

- **Settings → Branches → Add rule** for `main`: enable "Require status
  checks to pass before merging" and add your CI job names. Also enable
  "Do not allow bypassing the above settings" if you want it to apply to
  direct pushes too, not just PRs.
- **Settings → Actions → General → Workflow permissions**: tick "Allow
  GitHub Actions to create and approve pull requests." This is **off by
  default** on every repo and the workflow cannot open its PR without it —
  see [Gotchas](#gotchas-found-by-actually-running-this) below.

### 5. Create the approval environment

```bash
gh api -X PUT repos/<you>/<repo>/environments/dependency-approval \
  -H "Accept: application/vnd.github+json" \
  -f "reviewers[][type]=User" -F "reviewers[][id]=<your numeric GitHub user id>"
```

Find your id with `gh api user -q .id`. Whoever is listed here is who gets
the approval email — GitHub sends it based on their own notification
settings, so make sure "Actions" notifications are enabled at
github.com/settings/notifications.

### 6. Test it without waiting 2 months

```bash
gh workflow run bimonthly-dependency-update.yml --repo <you>/<repo>
```

It'll pause the same way a real scheduled run would; approve it from the
Actions tab (or the email, once one has fired for real) to see the whole
thing run.

---

## Gotchas found by actually running this

Nothing below is theoretical — each of these broke a real run before it was
fixed, on the two repos this was built for.

**"GitHub Actions is not permitted to create or approve pull requests."**
A separate repo setting from everything else here — see step 4. Without it,
the `apply` job pushes its branch successfully and then fails right at
`gh pr create`.

**The PR sits forever with no checks and never merges.**
Commits pushed using the workflow's own `GITHUB_TOKEN` do **not** trigger
other workflows — a deliberate GitHub Actions loop-prevention rule, not a
bug. Your CI's normal `pull_request`/`push` triggers will simply never fire
for this branch. The workflow works around this by explicitly running
`gh workflow run ci.yml --ref "$BRANCH"` right after pushing — the resulting
check-runs attach to the commit SHA, which is what branch protection
actually matches on, regardless of which event caused the run. If you
rename your CI workflow file, update that line.

**A Docker base image bump silently changes your pinning granularity.**
If your Dockerfile pins `node:20-bookworm-slim` (major-version only) and the
check script isn't careful, it'll happily offer `node:26.8.1-bookworm-slim`
— a fully-specified patch version, which is a different kind of pin than you
started with, not just a version bump. `check_docker_base.py` matches on the
same number of dot-separated components as what's currently pinned, so a
major-only pin only ever gets offered another major-only tag.

**Testing "does it correctly refuse to merge a broken update" is worth
doing once.** Push a deliberately broken dependency on a branch, open a PR,
and confirm `gh pr merge` is refused with something like *"3 of 3 required
status checks are expected."* If it isn't refused, required status checks
aren't actually configured yet (step 4), and this whole system is currently
decorative.

---

## Who the update commits are attributed to

By default the workflow commits as whoever approved it (`git config
user.name`/`user.email` near the top of the `apply` job), not as
`github-actions[bot]` — an approved update is real, reviewed work, and this
is what makes it show up in your own commit history and contribution graph.
Change those two lines if you'd rather it stay attributed to the bot.

The email used there has to match a **verified email on your GitHub
account**, or the commit will show the right name but won't link to your
profile and won't count toward your contribution graph. Check with:

```bash
gh api user/emails
```

---

## Files

```
examples/bimonthly-dependency-update.yml            the workflow itself
scripts/check_docker_base.py                        finds newer Docker Hub tags on the same track
scripts/summarize_updates.py                         turns the three *-outdated.json files into one markdown summary
scripts/apply_pip_updates.py                          bumps backend/requirements.txt to the versions found
scripts/apply_docker_update.py                        rewrites Dockerfile FROM lines to the versions found
```

`npm` updates are applied with
[`npm-check-updates`](https://github.com/raineorshine/npm-check-updates) at
run time (`npx --yes npm-check-updates -u`), not a custom script — it already
does this well.

---

## Licence

MIT.
