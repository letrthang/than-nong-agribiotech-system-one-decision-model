# Changelog

All notable changes to Thannong-Agribiotech. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and
the project uses [Semantic Versioning](https://semver.org/) — `0.x` while the model is being designed and trained, `1.0.0`
for the first released model. Branches (simplified Gitflow): feature branches → `develop` (integration) → `release/vX.Y.Z`
→ `main` (production); merging a release branch into `main` tags `vX.Y.Z` and publishes the GitHub Release automatically;
see [`GITHUB_COMMIT_RULE.md`](GITHUB_COMMIT_RULE.md).

## [Unreleased]

## [0.1.1] — 2026-11-09

### Changed
- **Simplified Gitflow.** Releases are prepared on `release/vX.Y.Z` (from `develop`; final-check fixes go directly on it) and
  merged into `main` by PR. The new `.github/workflows/release.yml` then tags `vX.Y.Z`, publishes the GitHub Release (the
  CHANGELOG section + generated notes) and merges `main` back into `develop`; `.github/workflows/gitflow-guard.yml` marks a
  PR into `main` red unless it comes from `release/vX.Y.Z` with its CHANGELOG section and an unused tag. No deployment yet.
  `GITHUB_COMMIT_RULE.md` describes the flow with a diagram (`images/gitflow.webp`, source `images/gitflow.html`).
- **Named as a decision model.** The repository is now `than-nong-agribiotech-system-one-decision-model` (was
  `than-nong-agribiotech-system-one-slm`; GitHub redirects the old URL), the README calls the model a System One decision
  model rather than a small language model, and links point to the renamed Laya repo
  `aimtas-system-one-decision-model-laya`.

## [0.1.0] — 2026-10-17

The first release: the plan and architecture of a Jev-compatible System One decision model for agricultural biotechnology
and medicine. No model is trained yet.

### Documentation
- **README** — what the model is (a fast *decision* model, not a chatbot), what makes it ours (built from the bare mmBERT
  encoder; our domain pretraining, decision heads, training method and server), what it does and does not do, the
  `POST /v1/systemone` API (compatible with TypeSafe's Jev) and where the models will live (Edge for the farm gateway,
  Skynet for the cloud server, on Hugging Face).
- **Focus** — five languages (English, Vietnamese, Japanese, Korean, Chinese) and twelve countries first (Vietnam, USA,
  Canada, Australia, New Zealand, Japan, Korea, Norway, Finland, Sweden, Singapore, China), with data and tests per country.
- **`architecture.md`** — goals, the base encoder, the decision heads, the training pipeline (DAPT → heads → domain decisions
  → calibration → per-task adaptation), our server, data, compute, evaluation, safety and compliance, repository layout,
  milestones, the relationship to AIMTAS and the risks.
- **`GITHUB_COMMIT_RULE.md`** — the git flow guide and what must never be committed in this public repo.
- **Licence** — Apache-2.0 for this repository and the released model weights.

[Unreleased]: https://github.com/letrthang/than-nong-agribiotech-system-one-decision-model/compare/v0.1.1...develop
[0.1.1]: https://github.com/letrthang/than-nong-agribiotech-system-one-decision-model/compare/v0.1.0...v0.1.1
[0.1.0]: https://github.com/letrthang/than-nong-agribiotech-system-one-decision-model/releases/tag/v0.1.0
