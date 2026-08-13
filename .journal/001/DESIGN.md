# Design: template-pyinfra

A GitHub "Use this template" repository that spawns new pyinfra plugin packages.
It merges the tooling/automation maturity of `template-go` with the package
architecture of `pyinfra-incus`, shipping a small set of sample custom
primitives (facts + operations) meant to be renamed and molded into a real
package.

Sources synthesized:

- `~/code/meigma/template-go` — tooling maturity bar (mise, moon, release-please,
  hardened CI, template-instantiation UX).
- `~/code/meigma/pyinfra-incus` — proven pyinfra plugin architecture, uv/moon
  Python config, Python release pipeline (already a mature port of the
  template-go release layer).
- pyinfra 3.x docs (`docs/api/facts.md`, `docs/api/operations.md`,
  `docs/api/connectors.md`, upstream `AGENTS.md`) — current API contracts.

## Key decisions

1. **Plain "Use this template" repo, no cookiecutter/copier.** Same as
   template-go: literal placeholder strings discoverable via
   `rg "template-pyinfra|template_pyinfra"`, a `DELETE_ME.md` first-setup
   checklist, `is_template = true` in `.github/repository-settings.toml`.
2. **No entry points needed.** pyinfra only discovers `pyinfra.connectors` via
   entry points; facts and operations ship as ordinary importable modules
   (verified in pyinfra source and confirmed by pyinfra-incus). The template
   includes a commented-out `[project.entry-points.'pyinfra.connectors']`
   stanza in `pyproject.toml` for projects that later add a connector.
3. **Adopt the pyinfra-incus package architecture wholesale, in miniature.**
   Thin public `facts.py` / `operations.py` over pure private domain modules
   plus a `_cli.py` quoting/security layer. This is the part users mold.
4. **Adopt the pyinfra-incus Python toolchain config nearly verbatim** (uv +
   hatchling + ruff + mypy + pytest + moon tasks), since it is already the
   Python translation of template-go's stack.
5. **Adopt template-go extras pyinfra-incus lacks:** MkDocs Material docs site +
   Pages workflow, `repository-settings.toml` + `configure_github_repo.py`
   settings-as-code, dependabot, DELETE_ME.md, and the session/agent protocol
   (already installed here).
6. **Sample domain: a real, ubiquitous CLI (`git config`).** Sample primitives
   must be runnable in CI without a daemon (unlike Incus) while still being
   realistic. `git` is present on every runner and gives natural idempotency
   (read config → diff → set/unset). See "Sample primitives" below.
   Alternative considered and rejected: a fictional `examplectl` (nothing would
   ever execute, so integration tests and the README demo would be fake).

## Repository layout

```
mise.toml / mise.lock              # python, uv, moon pinned; locked = true
.python-version                    # single pin, e.g. 3.12.x (see Open questions)
pyproject.toml                     # hatchling, src layout, PEP 735 dev group
uv.lock                            # committed; enforced via `uv lock --check`
moon.yml                           # root project tasks (below)
.moon/workspace.yml                # projects: {root: '.', docs: 'docs'}
.moon/toolchains.yml               # empty by design; system toolchain via mise
src/template_pyinfra/
  __init__.py                      # re-exports all public facts/operations, __all__
  py.typed
  facts.py                         # public fact classes only
  operations.py                    # public @operation functions only
  _cli.py                          # command builder: QuoteString discipline, option-lookalike rejection
  _gitconfig.py                    # sample domain module: parse / diff / build commands (pure)
tests/
  conftest.py                      # --integration flag, skip logic
  test_cli.py                      # command-builder tests + reserved-argument meta-test
  test_facts.py
  test_operations.py
  integration/
    _helpers.py                    # new_state / prepare / apply / fact_value (from pyinfra-incus)
    test_gitconfig.py              # @local, marked `integration`
docs/                              # MkDocs Material site (template-go pattern)
  mkdocs.yml  pyproject.toml  uv.lock  .python-version  moon.yml  docs/index.md
.github/
  dependabot.yml                   # github-actions@/, uv@/, uv@/docs
  repository-settings.toml         # squash-only, signed commits, rulesets, is_template
  scripts/configure_github_repo.py (+ test)   # ported from template-go
  scripts/validate-release.py (+ test)        # from pyinfra-incus
  workflows/{ci,docs-pages,release-please,release-dry-run,release,security-scan}.yml
README.md  CONTRIBUTING.md  SECURITY.md  DELETE_ME.md  CHANGELOG.md
AGENTS.md / CLAUDE.md / .session.md / .agents/skills/   # already installed
```

## Toolchain

- **mise** (`mise.toml` + committed `mise.lock`): `python`, `aqua:astral-sh/uv`,
  `aqua:moonrepo/moon`; `[settings] lockfile = true, locked = true` (fail-closed
  installs); `[env]` empty initially. ruff/mypy/pytest come from the uv dev
  group, not mise (pyinfra-incus precedent).
- **uv**: `uv run --locked` everywhere; PEP 735 `[dependency-groups] dev = [mypy,
  pytest, ruff]`; runtime dep `pyinfra>=3.x,<4`.
- **moon** root tasks (all through `uv run --locked`):
  `format` (ruff format --check + ruff check --select I), `lint` (ruff check),
  `lock` (uv lock --check), `typecheck` (mypy), `test` (pytest, unit only),
  `build` (uv build), `scripts-test` (pytest for `.github/scripts` — closes a
  known template-go gap where script tests exist but nothing runs them),
  `check` (deps: all of the above + `docs:build`, `runInCI: true`),
  `test-integration` (pytest --integration, separate CI job only).
- **Lint config** (pyproject): ruff line-length 100, lint select
  `["B","E4","E7","E9","F","I","RUF","SIM","UP"]`; mypy near-strict over `src/`
  only; pytest `testpaths = ["tests"]` with `integration` marker.

## CI/CD workflows

All workflows: top-level `permissions: {}` with per-job grants, all actions
SHA-pinned with version comments, `persist-credentials: false`, concurrency
group with cancel-in-progress.

- **ci.yml** — PR + push to main: checkout → `jdx/mise-action` (cache: true) →
  uv cache keyed on `uv.lock` + `docs/uv.lock` → `moon ci --summary minimal`.
- **integration.yml** — PR + push: runs `moon run root:test-integration` on
  ubuntu-latest (git is preinstalled; no daemon setup needed — much simpler
  than pyinfra-incus's Incus matrix, and that simplicity is the point of the
  sample domain).
- **docs-pages.yml** — template-go pattern: build `docs:build` on PR; deploy to
  Pages on push to main (`configure-pages` / `upload-pages-artifact` /
  `deploy-pages`, `id-token: write` on deploy only).
- **release-please.yml** — App-token minted via `create-github-app-token`
  (`vars.MEIGMA_RELEASE_APP_ID` + `secrets.MEIGMA_RELEASE_APP_PRIVATE_KEY`),
  `googleapis/release-please-action`; config: `release-type: python`,
  `draft: true`, `include-v-in-tag`, `extra-files` jsonpath bump of the package
  version inside `uv.lock` (pyinfra-incus precedent).
- **release-dry-run.yml** — PR jobs gated on `release-please--*` head branches
  (broad trigger so required contexts always report): `uv build --no-sources`,
  `validate-release.py`, wheel smoke test via
  `uv run --no-project --with "$wheel"`, `uv publish --dry-run`.
- **release.yml** — on `v*` tags: `resolve-release` validates tag and polls for
  the draft release → rerun `moon run root:check` → rebuild → validate metadata
  against tag → smoke test → `uv publish --trusted-publishing always` to PyPI
  (environment `pypi`, `id-token: write`) → GitHub artifact attestations for
  the wheel/sdist checksums via an isolated reusable `attest.yml` (template-go
  SLSA L3 pattern) → inspection summary in `$GITHUB_STEP_SUMMARY`; publishing
  the draft release stays a human decision.
- **security-scan.yml** — weekly cron. template-go scans a container image;
  there is no image here, so substitute: `osv-scanner` (or `pip-audit`) against
  `uv.lock` + `zizmor` over workflow files, SARIF-uploaded to code scanning.
  (Decision point — see Open questions.)

## Sample primitives (the moldable core)

Domain: repo-scoped git configuration, wrapping `git config`. Everything below
is deliberately small so a new project can `rg`-replace and gut it in minutes.

- **`_cli.py`** — `git_command(*args, scope...) -> StringCommand` following
  pyinfra-incus rules: bare `str` only for trusted literals, every user value
  wrapped in `QuoteString`, values starting with `-` rejected
  (option-lookalike guard), structured payloads via `_stdin` if ever needed.
- **`facts.py`** — two facts demonstrating the two canonical shapes:
  - `GitVersion(FactBase[str])` — argument-less, class-attribute-style command,
    `requires_command`, trivial `process`.
  - `GitConfig(FactBase[dict])` — parameterized (`scope`/`path`), stores args as
    instance attrs between `command()` and `process()` (pyinfra calls both on
    one instance), parses `git config --list --null` output.
  - Both use a `_fact_process` decorator converting any parse error to
    `FactProcessError` so one bad host degrades instead of aborting the run,
    a `default()` staticmethod, and typed `command()/process()` signatures
    (pyinfra 3.x convention).
- **`operations.py`** — one operation demonstrating the full idempotency loop:
  - `config_entry(key, value=None, *, present=True, scope=...)` — reads
    `host.get_fact(GitConfig, ...)`, calls `host.noop(...)` when converged,
    otherwise yields `StringCommand`s built by `_gitconfig.py`; domain errors
    converted to `OperationValueError` via a `_guarded` helper.
  - Parameter naming avoids pyinfra reserved global arguments (no bare `name`);
    a meta-test introspects every exported operation's signature against
    `pyinfra.api.arguments.all_argument_meta` to enforce this permanently.
- **`_gitconfig.py`** — pure functions: parse output, compute changes
  (`config_changes(current, desired)`), build commands. No I/O, no pyinfra
  state; this is what makes unit tests mock-free.
- **`__init__.py`** re-exports everything with `__all__`, mirroring pyinfra's
  own `pyinfra.facts.server` / `pyinfra.operations.files` import ergonomics.

## Testing strategy

Two tiers, per pyinfra-incus (not pyinfra's internal JSON-fixture harness —
that is for pyinfra core development, not plugins):

- **Unit** (default, mock-free): assert `str(command)` /
  `command.get_raw_value()` for builders and facts; feed literal output lines
  to `process()`; test the pure domain functions directly. Include the
  multi-line-output rejoin case (`process` receives `list[str]`).
- **Integration** (`pytest.mark.integration`, `--integration` flag in
  conftest): drive the real pyinfra API against `@local` using the 4-helper
  harness (`new_state` / `prepare` / `apply` / `fact_value`); assert
  `meta.will_change` / `meta.did_change()`; verify real state via independent
  `subprocess` calls; prove idempotency by applying twice. Operates on a
  throwaway repo in `tmp_path` so it never touches the runner's real config.

## Template instantiation UX (DELETE_ME.md)

1. What the template provides / how the layers fit (moon entrypoints, release
   rationale).
2. First-setup checklist: rename dist + import package (`template-pyinfra` /
   `template_pyinfra`), `rg` for placeholders, `uv lock`, decide whether to
   keep the docs site, configure PyPI trusted publishing + the release App,
   run `configure_github_repo.py plan|apply`, `moon run root:check`, rewrite
   README/CONTRIBUTING/SECURITY, add a LICENSE, delete DELETE_ME.md.
3. Gut-the-sample guide: which files hold the sample domain and the order to
   replace them (facts → operations → domain module → tests → README example).

## Open questions

1. **Python floor** — pyinfra-incus uses `>=3.11`; template-go pins latest
   (3.14.x). Recommendation: pin `.python-version` to current stable and set
   `requires-python = ">=3.11"` so generated packages stay broadly usable.
2. **Security scan substitute** — osv-scanner vs pip-audit for `uv.lock`;
   include zizmor for workflow linting? Recommendation: osv-scanner + zizmor.
3. **Attestations for PyPI artifacts** — keep template-go's isolated
   `attest.yml` (GitHub attestations on checksums) alongside PyPI trusted
   publishing, or rely on trusted publishing alone as pyinfra-incus does.
   Recommendation: include `attest.yml`; it is the SLSA L3 differentiator.
4. **AGENTS.md ruleset** — template-go carries a Go best-practices ruleset in
   its ai-protocol block; write a Python/pyinfra equivalent (typing, QuoteString
   discipline, reserved-arg rule, fact/operation conventions)?

## Implementation phases

1. Toolchain skeleton: mise, uv/pyproject, moon, empty package, CI green.
2. Sample primitives + unit tests (`_cli.py`, facts, operation, meta-test).
3. Integration tier + workflow.
4. Release layer: release-please, dry-run, release, validate-release.py.
5. Docs site + Pages workflow.
6. Repo governance: repository-settings.toml + script, dependabot,
   security-scan, README/CONTRIBUTING/SECURITY/DELETE_ME.
