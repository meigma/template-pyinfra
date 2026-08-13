---
id: 001
title: Bootstrap pyinfra template repo
started: 2026-08-12
---

## 2026-08-12 18:00 — Kickoff
Goal for the session: user asked to start a session immediately after repo creation; no task stated yet. Expected direction: build out the meigma/template-pyinfra template repository.
Current state of the world: public repo meigma/template-pyinfra created and cloned to ~/code/meigma/template-pyinfra. Session protocol framework installed from ~/code/ai (AGENTS.md/CLAUDE.md, .session.md, skills under .agents/skills, .gitignore entries) and pushed as the initial commit on main (9fb459a). Journal branch journal/jmgilman created via session-setup, scaffold committed and pushed; journal worktree at .wt/journal-jmgilman. Repo otherwise empty — no template content yet.
Plan: await the user's first task for this session.

## 2026-08-12 18:10 — Design doc drafted
Task: synthesize a design for template-pyinfra from template-go (tooling maturity), pyinfra-incus (plugin architecture), and pyinfra 3.x docs.
Done: surveyed both repos via subagents; pulled pyinfra docs on custom facts/operations/plugin packaging; wrote DESIGN.md in this session folder.
Key learnings: facts/operations plugins need no entry points (only pyinfra.connectors group is discovered); pyinfra-incus is already a Python port of template-go's release layer (uv + hatchling + moon + release-please draft + trusted publishing), so most config transfers verbatim; pyinfra-incus test approach (mock-free unit tests on rendered commands + @local integration harness with prepare/apply helpers) beats pyinfra's internal JSON-fixture harness for plugins; template-go gap worth closing: .github/scripts tests exist but nothing runs them.
Decisions in DESIGN.md: plain "Use this template" repo (no cookiecutter); sample domain wraps `git config` so CI needs no daemon; miniature pyinfra-incus architecture (facts.py/operations.py over pure _domain modules + _cli.py QuoteString layer); template-go extras included (mkdocs+Pages, repository-settings-as-code, dependabot, DELETE_ME.md, attest.yml SLSA L3 pattern).
Open questions logged: Python floor, security-scan substitute (osv-scanner+zizmor recommended), PyPI attestation approach, AGENTS.md Python ruleset.
Next: user review of DESIGN.md.
