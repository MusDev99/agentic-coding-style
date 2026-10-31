# Agentic Coding Style

Static GitHub Pages site about how I build software with AI agents: RAG, orchestration, pipelines, guardrails, query optimization and evals.

Publish: **Settings → Pages → Deploy from a branch → `main` / `(root)`**.

## Standards

- 🌿 [BRANCHING.md](BRANCHING.md): branching strategy, Conventional Commits, branch protection, stale-branch cleanup and the rollout roadmap for the other repos.
- 🤖 `.github/workflows/branch-policy.yml`: PR check for branch names and Conventional Commit titles, copy it to each repo.
- 🧹 `.github/workflows/branch-cleanup.yml`: deletes a PR's branch after it is merged (skips forks, `main` and stacked bases), copy it to each repo.
- 🔍 [`.agents/skills/code-review/SKILL.md`](.agents/skills/code-review/SKILL.md): shared code-review checklist (intent, readability, tests, security, performance, style, docs) with findings by severity. Same file in every repo.
