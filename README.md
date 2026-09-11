# AI Application CI/CD Guardrails

A GitHub Actions pipeline that runs four independent security checks on every pull
request touching an AI-powered application. Each check is a separate job so that a
failure in one does not hide a failure in another, and so the pipeline reads clearly
as defense in depth rather than a single pass/fail gate.

## Why four layers, not one

A single "AI security scan" step is not defensible in an interview, because it hides
what is actually being checked. This pipeline separates concerns:

| Layer | Tool | Question it answers |
|---|---|---|
| Secret detection | Gitleaks | Did a developer commit a live API key? |
| Static analysis (SAST) | Semgrep | Does the application code itself have an injectable or unsafe pattern? |
| Dependency risk | pip-audit | Does a library the app depends on have a known CVE? |
| Prompt-injection / jailbreak testing | Promptfoo | Can the deployed system prompt be overridden or made to leak instructions? |

The first three are standard DevSecOps controls applied to any codebase. The fourth is
specific to AI applications: it is the only layer that tests the *behaviour* of the
model under adversarial input, rather than the code around it.

## Design principles applied

- **Fail closed.** Every job blocks the merge on failure. Nothing is advisory-only.
- **Least privilege secrets.** The workflow only requests `OPENAI_API_KEY` in the one
  job that needs it (prompt evaluation), not globally at workflow level.
- **Defense in depth.** A pipeline that only checked prompts would miss a leaked key.
  One that only checked secrets would miss a jailbreakable system prompt. Both classes
  of failure are real and have shipped in production AI apps.
- **Independent jobs, not a single script.** Each job produces its own pass/fail status
  in the PR check list, so a reviewer can see exactly which control failed, not just
  that "security" failed.

## What this is not

This is not a claim of "zero trust" architecture on its own — zero trust refers to
identity and network access control (mTLS, short-lived credentials, per-request auth),
which is a separate and larger topic. This pipeline is a CI/CD guardrail suite: it
catches specific, well-known failure classes in AI applications before merge. Be precise
about that distinction if asked, rather than over-claiming.

## How to actually use this

1. Copy `sample_app/` or point `promptfoo.config.yaml` at your own system prompt.
2. Add `OPENAI_API_KEY` (or your provider's key) as a repository secret.
3. Push to a branch and open a PR. Watch all four jobs run in the Actions tab.
4. Deliberately break one test (e.g. weaken a system prompt) to see the pipeline
   correctly fail. This is the step most people skip, and it's the one that proves
   the pipeline actually catches something rather than always passing green.
