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

## Lesson 2 — Parallel jobs

**File:** `.github/workflows/02.parallel.yml`

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

## Lesson 1 — Workflow syntax & triggers

**File:** `.github/workflows/01-basics.yml`

**What I learned:**

- A workflow file lives in `.github/workflows/` and is YAML.
- `on:` defines what triggers a run — here, `push`, `pull_request`, and
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
