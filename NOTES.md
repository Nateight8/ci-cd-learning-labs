# GitHub Actions — Learning Notes

Practice repo for hands-on GitHub Actions. Each entry: what I learned,
then how it maps to a real-world pipeline.

---

## Lesson 0 — Hello World

**File:** `.github/workflows/00-hello-world.yml`

**What I learned:**

- The bare minimum a workflow file needs: `on:`, `jobs:`, `runs-on:`, `steps:`.
- `workflow_dispatch` as a trigger lets you run it manually from the
  Actions tab — useful for the very first confirmation that Actions is
  wired up correctly before adding real logic.

**Real-world application:**
Before automating anything meaningful, you confirm the pipeline
mechanism itself works — the CI/CD equivalent of "hello world." Teams
do this when first enabling Actions on a repo, to rule out permissions
or config issues before building real jobs on top.

---

## Lesson 1 — Workflow syntax & triggers

**File:** `.github/workflows/01.hello-world.yml`

**What I learned:**

- A workflow file lives in `.github/workflows/` and is YAML.
- `on:` defines what triggers a run — `push`, `pull_request`, and
  `workflow_dispatch` (manual trigger button in the Actions tab).
- `jobs:` → `steps:` structure, `runs-on:` picks the runner OS.
- `uses:` runs a pre-built action (e.g. `actions/checkout`); `run:` runs
  raw shell commands.
- Built-in context variables like `github.event_name`, `github.sha`.

**Real-world application:**
Every CI/CD pipeline starts here. This is the shape that would run
lint/tests on every PR into main for a real Node/Express service, and
could later be extended to build/deploy on push to main. Knowing
`workflow_dispatch` matters for pipelines that need a manual
"deploy now" button (e.g. releasing to production on demand).

---

## Lesson 2 — Parallel jobs

**File:** `.github/workflows/02.parallel.yml` _(adjust if named differently)_

**What I learned:**

- Jobs with no `needs:` between them run independently — GitHub Actions
  starts them all at roughly the same time, each on its own runner.
- Three jobs: `lint`, `test`, `security-scan` — none depends on another.

**Real-world application:**

- Independent checks (linting, unit tests, security/dependency
  scanning) running in parallel gives faster feedback — if linting
  fails, you know immediately without waiting on the others.
- Testing across multiple environments (Node versions, OS) at once.
- Building independent services in a monorepo simultaneously instead
  of one at a time.

---

## Lesson 3 — Sequential jobs (`needs:`)

**File:** `.github/workflows/03.sequence.yml`

**What I learned:**

- `needs: <job-name>` makes a job wait for another to succeed first —
  a hard gate, not just visual ordering.
- Chain built: `build` → `test` → `deploy-dev`.
- If a job in the chain fails, everything downstream is skipped rather
  than running anyway.

**Real-world application:**

- Build → Test → Deploy is the standard pipeline shape — you shouldn't
  test code that hasn't built, or deploy code that failed tests.
- Database migration before app deploy — deploying first could break
  things if the schema isn't ready.
- Staging deploy → smoke tests → production deploy — production should
  never run unless staging passed.

**Trigger note:** used `workflow_dispatch` here for controlled manual
testing while learning. In a real pipeline, `build`/`test` would
typically trigger automatically on `push`/`pull_request`, with manual
triggers reserved for deliberate human gates — e.g. approving a
production deploy specifically, not the whole chain.

---

## Lesson 4 — Event-driven triggers

**File:** `.github/workflows/04.event-driven.yml`

**What I learned:**

- Multiple event-driven triggers can live in the same `on:` block —
  `push`, `pull_request`, `issues`, `release` all demonstrated together.
- `push`/`pull_request` support a `branches:` filter to scope which
  branches matter; `issues`/`release` don't use `branches:` — they
  filter on `types:` instead (e.g. `opened`, `published`).
- `github.event_name`, `github.actor`, `github.ref`, `github.sha` —
  context vars that tell you exactly what fired the run and which
  commit it was, useful when one workflow reacts to several event types.
- The four broad trigger categories: **event-driven** (push, pull_request,
  issues, release...), **scheduled** (`schedule` + cron), **manual**
  (`workflow_dispatch`), and **workflow-triggered** (`workflow_run` — one
  workflow reacting to a different workflow finishing).

**Real-world application:**

- `push` to main → run tests immediately after merge, or trigger a
  staging deploy.
- `pull_request` (opened/synchronize/reopened) → the standard CI gate,
  blocks merging broken code.
- `issues: opened/edited/closed` → auto-label new issues, post a
  welcome comment, notify a Slack channel, or trigger cleanup on close.
- `release: published` → build/push a versioned Docker image, attach
  compiled binaries to the release automatically.
- `workflow_run` → separating build and deploy into distinct workflow
  files (different secrets/permissions/ownership), where deploy only
  fires once build has actually succeeded. Different from `needs:`,
  which orders jobs _within_ one workflow file — `workflow_run` connects
  two _entirely separate_ workflow files.
- `schedule` (cron) → nightly builds, scheduled dependency/security
  scans, cleanup jobs — not tied to any code change at all.

---

## Lesson 5 — Variables: `env`, repo `vars`, `secrets`, and precedence

**File:** `.github/workflows/05.variables.yml`

**What I learned:**

- **`env:`** — defined directly in the workflow YAML, referenced as
  plain `$VAR_NAME` in `run:` (shell-style) or `${{ env.VAR_NAME }}`
  elsewhere. Can be set at workflow, job, or step level.
- **`vars.*`** — repository/org-level variables set in GitHub UI
  (Settings → Secrets and variables → Actions → Variables), referenced
  as `${{ vars.VAR_NAME }}`. Plaintext, same as `env` — no extra
  security, just a different storage location.
- **`secrets.*`** — same idea as `vars` but encrypted at rest and
  masked in logs. The only one of the three with a real security
  reason behind the choice.
- **Precedence** — step-level `env` overrides job-level `env`, which
  overrides workflow-level `env`, when the same variable name is
  defined at multiple levels. Verified by defining `APP_ENV` at all
  three levels and confirming each scope prints its own value, with
  the step-level override not leaking into other steps in the same job.
- **`workflow_dispatch` inputs** — values a human fills in via a form
  when manually triggering a run (`inputs.<name>`), distinct from
  `workflow_call` inputs, which are parameters passed when one workflow
  calls another as a reusable workflow.

**Real-world application:**

- `env:` — local, workflow-specific values, changed via code review
  (e.g. `NODE_ENV: test` for a CI job).
- `vars:` — shared config reused across multiple workflow files, or
  editable by someone without needing to open a PR (e.g. `APP_NAME`
  used by build, test, and deploy workflows alike).
- `secrets:` — anything sensitive, no exceptions: DB connection
  strings, API keys, deploy credentials.
- **For this project specifically** (Next.js + Better-Auth, Express,
  Postgres): staging vs production DB URLs are secrets, since they
  contain credentials. The cleaner real-world mechanism for
  "same secret name, different value per stage" is **GitHub
  Environments** (Settings → Environments) — each environment (staging,
  production) holds its own scoped `DATABASE_URL` secret, and a job
  picks which one applies via `environment: ${{ inputs.environment }}`.
  This also unlocks environment protection rules (e.g. requiring manual
  approval before a production deploy) — more idiomatic than manually
  branching on separate secret names with an if/else.
- `workflow_dispatch` inputs → a deploy workflow where a human picks
  "staging" or "production" from a dropdown at trigger time, instead of
  maintaining near-duplicate workflow files.

---

## Lesson 6 — Outputs

**File:** `.github/workflows/06.outputs.yml`

**What I learned:**

- **Step outputs** — a step writes a value via `echo "name=value" >>
$GITHUB_OUTPUT`, and needs an `id:` so later steps can reference it
  as `${{ steps.<id>.outputs.<name> }}`. Works _within the same job_,
  no `needs:` required — steps in a job already run in order and share
  context automatically.
- **Job outputs** — a job re-exposes a step's output at the job level
  via its own `outputs:` block. A _different_ job can only read this
  if it explicitly declares `needs: <job-name>`, then reads it as
  `${{ needs.<job-name>.outputs.<name> }}`.
- Common mistakes: forgetting `id:` on the producing step (nothing to
  reference), referencing outputs with plain `${...}` instead of
  `${{ ... }}`, or trying to run shell commands directly inside an
  `outputs:` block (it's YAML metadata, not a place to execute code).

**Real-world application:**
A `build` job computes a Docker image tag (e.g. from `github.sha` or a
version bump) and exposes it as a job output; a separate `deploy` job
(`needs: build`) reads that exact tag to know which image to pull and
run — without job outputs, `deploy` would have no way to know what
`build` actually produced.

---

## Lesson 7 — Conditions (`if:`)

**File:** _(reference only, no dedicated practice file — concept was clear without a build exercise)_

**What I learned:**

- `if:` gates whether a step or job runs at all, based on an
  expression — `github.ref`, `github.event_name`, or built-in status
  functions: `success()` (default when `if:` is omitted), `failure()`,
  `always()` (runs even after a prior failure — used for cleanup/
  notifications), `cancelled()`.

**Real-world application:**

- `if` isn't just about deciding _what_ to build — its most common
  real-world use is deciding **who gets notified, and when**: gate a
  quality-check job first, branch deploy jobs on `github.ref` (`main`
  vs `develop`), then use `if: failure()` on a notify job so the team
  gets alerted only when something actually breaks, and
  `if: success() && github.ref == 'refs/heads/main'` to notify on a
  successful production deploy specifically.
- Directly relevant to this project: a failed Postgres migration or a
  broken auth flow should trigger an immediate notification, not fail
  silently in a log nobody checks.

---

## Note — A single job re-run can show a misleading graph

Re-running just one job in an already-completed workflow (rather than
a fresh full run) can make the Actions graph look wrong — e.g.
`deploy-dev` showing green/complete while `build` is still spinning.
This isn't `needs:` breaking; it's the graph mixing the fresh re-run
of one job with stale statuses from the previous full run.

**Real-world application:** always confirm you're looking at one
continuous run (check timestamps, or trigger a fresh full run) before
concluding a dependency chain is broken.

---

## Note — YAML indentation breaks steps silently (loudly, actually)

```yaml
steps:
  - name: Hello World
run: echo "Hello World"
```

`run:` here is indented _less_ than `name:`, so it falls out of the
step entirely instead of being a property of it. Result: GitHub Actions
reports an invalid workflow file — a step with no `run`/`uses`/etc. to
act on, plus an unexpected top-level `run:` key.

**Fix:** `run:` must align directly under `name:` at the same indent
level, both under the `- `:

```yaml
steps:
  - name: Hello World
    run: echo "Hello World"
```

**Real-world application:** YAML is whitespace-sensitive. This is one
of the most common first mistakes in any workflow file — unlike the
misplaced-folder issue below, this one _is_ caught, with a specific
line/column reference in the Actions tab.

---

## Note — `run:` defaults to bash; running Python without `shell:`

By default, `run:` steps on `ubuntu-latest` execute through **bash**,
regardless of what language the command "looks like." Two ways to
actually run Python inside a step:

**Method 1 — call the `python` binary from bash** (no `shell:` needed):

```yaml
- run: python -c "print('Hello in python')"
```

This works because the whole `print(...)` sits safely inside a quoted
string handed to `python` as one argument — bash never tries to parse
what's inside the quotes.

**Method 2 — `shell: python`** (treats the whole `run:` block as Python
source, no wrapper needed):

```yaml
- run: print('Hello using shell: python')
  shell: python
```

**What happens if you skip both** — write raw `print('Hello')` under
`run:` with no `shell:` key:

```yaml
- run: print('Hello from explicit bash shell')
```

This does **not** fail with "command not found" as expected. It fails
with a **bash syntax error**:

```
syntax error near unexpected token `'Hello from explicit bash shell''
```

Reason: parentheses `()` are special syntax in bash (they open a
subshell). Bash can't even parse the line — it never gets as far as
looking up whether `print` is a valid command. This is different from
Method 1, where the parentheses are protected inside a quoted string.

**Real-world application:** `ubuntu-latest` comes with Python
pre-installed, so calling it directly from bash (Method 1) is common
and simple for one-off scripting inside a step. `shell: python`
(Method 2) matters more for multi-line Python logic where wrapping
everything in `python -c "..."` gets unwieldy. Knowing bash is the
silent default — and that it fails on parse, not just on missing
commands — explains a whole class of confusing early errors.

---

## Note — Misplaced workflow folder (silent failure)

GitHub only scans the exact path `.github/workflows/` at the repo root.
If it's misnamed or misplaced (e.g. `.github/workflow/`, singular, or
nested in a subfolder), there is **no error message anywhere** — the
workflow simply never appears in the Actions tab. No red X, no log.

**Real-world application:**
This is a common early mistake precisely because it fails silently.
Worth checking folder placement first whenever a workflow "isn't
running" and there's no error to go on.

---

## Coming up

Next: reusable workflows (`workflow_call`), then scaffolding the real
project — Next.js (Better-Auth) + Express API + Postgres, containerized
with Docker (networking + volumes), deployed to EC2.
