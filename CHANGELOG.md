# Changelog

All notable changes to Thannong-Agribiotech. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and
the project uses [Semantic Versioning](https://semver.org/) — `0.x` while the model is being designed and trained, `1.0.0`
for the first released model. Branches: `develop` (daily work) → `main` (checked) → `release` (released versions, tagged
`vX.Y.Z`); see [`GITHUB_COMMIT_RULE.md`](GITHUB_COMMIT_RULE.md).

## [Unreleased]

### Changed
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

[Unreleased]: https://github.com/letrthang/than-nong-agribiotech-system-one-decision-model/compare/v0.1.0...develop
[0.1.0]: https://github.com/letrthang/than-nong-agribiotech-system-one-decision-model/releases/tag/v0.1.0
