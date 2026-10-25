# Contributing to Solstice

Thanks for helping. This guide covers how to get set up, how changes are
made, and what happens after you open a pull request.

Questions are welcome on the [Solstice Discord](https://discord.gg/gZTJfUujX).
Everyone taking part is expected to follow the [Code of Conduct](CODE_OF_CONDUCT.md).
Security problems go through [SECURITY.md](SECURITY.md), never a public issue.

## Finding something to work on

Issues labelled `good first issue` are small and self-contained, and say which
files are involved. Comment on an issue before starting so two people do not
pick up the same one. If an issue has had no activity for a week after being
claimed, it is open again.

For anything larger than a bug fix, open an issue first and describe what you
want to change, so the approach can be agreed before you write the code.

## The one rule that matters

Solstice exists to suspend stateful Flink jobs without losing their state. A
job must never be suspended unless its state has been captured first, and must
never be resumed from anything other than its own savepoint.

Any change to the suspend or resume path needs tests showing that rule still
holds, including the failure cases. Most of the existing tests in
`operator/src/test` are there to protect it.

## Development setup

The repository has four parts:

| Directory | What it is | Language |
| --- | --- | --- |
| `operator/` | The Kubernetes operator | Java 17, Maven |
| `telemetry/` | Carbon intensity service | Python 3.11, FastAPI |
| `dashboard/` | Status dashboard | React with Vite, FastAPI backend |
| `charts/`, `k8s/` | Helm chart and raw manifests | YAML |

### Operator

You need Java 17 and Maven 3.9 or later.

```bash
mvn -f operator/pom.xml verify
mvn -f operator/pom.xml test -Dtest=PlanExecutorTest
mvn -f operator/pom.xml test -Dtest=PlanExecutorTest#resumeIsBlockedWithoutASavepoint
```

### Telemetry service

```bash
cd telemetry
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cd src && uvicorn main:app --port 8080
```

Without `ELECTRICITY_MAPS_TOKEN` it runs in simulated mode.
`/telemetry/status?simulate=dirty` and `?simulate=clean` force a grid state.

### Dashboard

```bash
cd dashboard
npm install
npm run build
```

### Running everything on a local cluster

You need Docker, Minikube, kubectl, Helm, Java 17 and Maven. Give Docker at
least 6GB of memory.

```bash
make up
```

This creates a Minikube profile called `solstice`, builds the images, deploys
the Flink Kubernetes Operator, SeaweedFS, the Solstice operator and a sample
Flink job, and finishes with a smoke test that suspends the job and resumes
it from its savepoint. The first run takes around 20 minutes.

| Command | What it does |
| --- | --- |
| `make demo` | Suspend the job, then resume it from its savepoint |
| `make demo-dirty` | Suspend the job and wait for its savepoint |
| `make demo-clean` | Resume the job and wait for it to restore |
| `make status` | Show the controller, the Flink job and the pods |
| `make test` | Run the operator test suite |
| `make teardown` | Delete the cluster |

`PROFILE`, `CPUS` and `MEMORY` change the Minikube profile and its size, for
example `make up PROFILE=solstice-dev MEMORY=8g`.

## Making a change

1. Fork the repository and create a branch from `main`, for example
   `telemetry-timeout` or `savepoint-panel`.
2. Keep the pull request focused on one issue. Two unrelated fixes are two
   pull requests.
3. Add or update tests for what you changed, and run them locally.
4. Update the README or docs if you changed behaviour or configuration.

## Commit history

Pull requests are merged with their commits intact, so every commit lands on
`main` as you wrote it. The commit history is reviewed along with the code.

- Each commit makes one coherent change. Keep refactoring, moving code and
  behaviour changes in separate commits so each can be reviewed on its own.
- The build and tests pass at every commit, not only at the last one, so that
  `git bisect` keeps working.
- A small pull request is often a single commit. A larger one is several
  commits, ordered so that each builds on the one before it.
- Do not add commits that correct earlier commits in the same pull request,
  such as "fix typo" or "address review comments". Amend the commit they
  belong to (`git commit --fixup <sha>`, then
  `git rebase -i --autosquash upstream/main`) and force-push the branch.
- Do not merge `main` into your branch. Rebase onto it instead with
  `git fetch upstream && git rebase upstream/main`.

### Commit messages

Commits follow [Conventional Commits](https://www.conventionalcommits.org):

```
type(scope): what the change does, in plain words

Why the change is needed, and anything a reviewer would not see from the
diff alone. Wrap the body at 72 characters.

Fixes #42.
```

- The summary starts with a lowercase letter, uses the imperative ("add",
  not "added" or "adds") and has no period at the end. Keep the whole line
  within 72 characters.
- Every commit references its issue on the last line of the message. Use
  `Fixes #42.` on the commit that completes the issue and `Part of #42.` on
  the commits that lead up to it.
- The explanation can be left out of a trivial commit. The issue reference
  cannot.
- A change that breaks existing installs, such as a change to the CRD,
  configuration, metric names or status strings, adds `!` after the type or
  scope, for example `feat(operator)!: ...`, and explains the break in the
  body.

Types: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `ci`.
Scopes follow the part of the code: `operator`, `scheduling`, `cost`,
`forecast`, `inventory`, `evaluation`, `metrics`, `telemetry`, `dashboard`,
`chart`, `k8s`, `scripts`, `monitoring`, `github`. The scope can be left out
when a change has no single area.

Examples:

```
feat(dashboard): show the last savepoint time for each job

Operators had to read the FlinkDeployment status to find out when a job
was last checkpointed before a suspension. The jobs table now shows it
next to the job state.

Fixes #57.
```

```
refactor(cost): move the break-even calculation into CostModel

Part of #61.
```

## Pull requests

- Open the pull request against `main`, for a single issue.
- Write the title as a Conventional Commit, in the same form as a commit
  summary. For a single-commit pull request, use the commit summary.
- In the description, say in a few sentences what the pull request does and
  anything reviewers should look at closely, then add `Fixes #N`, how you
  tested the change, and screenshots for dashboard or Grafana changes.
- When you address review feedback, change the commits themselves and
  force-push, then reply on the pull request saying what changed.
- CI must pass before review, and one maintainer approval is needed to merge.
- Pull requests are rebased onto `main` when merged, keeping each commit.

## What to expect from review

A maintainer will give a first response within 3 working days. If you have
heard nothing after that, a polite nudge on the pull request or on Discord is
welcome.

Review comments are about the code, not about you. If a change is not going to
be accepted, we will say so and explain why rather than leave it open.

## Licence

Solstice is licensed under the [Apache License 2.0](LICENSE). By contributing,
you agree that your contributions are licensed under the same terms.
