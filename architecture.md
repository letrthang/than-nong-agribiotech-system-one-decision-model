# Thannong-Agribiotech — architecture

*Design, 2026-10-11. Nothing here is built yet. Numbers marked **(est.)** are planning estimates to check in the first
experiments, not measurements.*

## 0. Summary

Thannong-Agribiotech is a **System One decision model, compatible with TypeSafe's Jev**: a text encoder plus small
**decision heads** that answer typed questions with calibrated probabilities, behind the same System One API as Jev. It is built **from the bare `jhu-clsp/mmBERT` encoder**, with no part taken from
any existing decision model (owner, 2026-10-11). Everything above the encoder is ours: the domain pretraining, the heads,
the training method and the server.

1. **Domain-adaptive pretraining (DAPT)** of the encoder on agricultural-biotech and medical text, so it knows the domain
   before it learns any task.
2. **Our own decision heads**, trained with proper-scoring-rule losses and calibrated.
3. **Our own server** implementing TypeSafe's `POST /v1/systemone` API, so AIMTAS callers switch to it by configuration.
4. Two sizes: **Edge** (gateway CPU) and **Cloud**.

```
                 ┌──────────────────── training (this repo) ────────────────────┐
 bare encoder    │ S1 DAPT          S2 heads:          S3 domain      S4 calibrate  │  safetensors + ONNX   our server
 mmBERT-small ──►│ agri-biotech  ►  general typed   ►  decisions   ►  S5 per-task  │──► HF model repos ──► POST /v1/systemone
 mmBERT-base     │ + medical MLM    decisions          (+ teacher)    adaptation    │                      (Space / cluster / gateway)
                 └───────────────────────────────────────────────────────────────┘
```

## 1. Goals and non-goals

**Goals**
- Typed decisions (`choice`, `score`, `noul`) for agricultural biotechnology and medicine, in **English, Vietnamese and
  Japanese** first, for the **ten focus countries** of §6.0 first.
- Calibrated probabilities that callers can gate on, plus an **escalate** probability (hand the case to System 2 or a
  person).
- Two sizes: **Edge** (runs on the AIMTAS gateway's CPU, ≤ ~150 M parameters) and **Cloud** (~300 M).
- **Jev-compatible:** speak TypeSafe's System One API (`POST /v1/systemone`, [docs](https://docs.typesafe.ai/introduction))
  with the same request and answer shapes and question types (`choice`, `score`, `noul`) as Jev, so any Jev caller (the
  AIMTAS gateway and cloud among them) can switch to it by configuration.
- Beat AIMTAS's current System 1 model on the AIMTAS farm hold-out before it replaces it there.

**Non-goals**
- No text generation, chat, diagnosis or prescription. System 2 (an LLM) writes text; people make medical decisions.
- No actuation. Callers keep every safety rule in deterministic code.
- No pretraining of an encoder from scratch.
- No weights, code or head design taken from another decision model; only the bare mmBERT encoder (MIT) and public data.

## 2. The base encoder: mmBERT

`jhu-clsp/mmBERT` (Johns Hopkins University, 2025; paper arXiv:2509.06888; MIT licence) is a multilingual **ModernBERT**
encoder: bidirectional, masked-language-model pretrained on 3T+ tokens in 1,800+ languages (public training data),
8,192-token context, 256k-token vocabulary. It reads Vietnamese, Japanese, Korean, Norwegian, Finnish and Swedish as well
as English. It outputs one vector per token and has no task head; that is what we add.

| Variant | Start from | Params | Layers / hidden | Use |
|---|---|---|---|---|
| **Thannong-Agribiotech-Edge** | `jhu-clsp/mmBERT-small` | 140 M (42 M without embeddings) | 22 / 384 | gateway CPU; multilingual |
| **Thannong-Agribiotech** (cloud) | `jhu-clsp/mmBERT-base` | 307 M (110 M without embeddings) | 22 / 768 | best quality per size among open multilingual encoders |

- **Tokenizer: keep mmBERT's 256k vocabulary unchanged.** New tokens would need untrained embeddings; DAPT teaches the
  domain's words through the existing sub-word pieces. Measure token counts of domain text before and after.
- Most parameters are the vocabulary table (~200 M of mmBERT-base's 307 M). If the Edge model is still too slow on the
  gateway, the first lever is **vocabulary pruning** to the languages we serve, then int8 quantisation.

## 3. Decision heads (our design)

**Input format** (one sequence, ≤ 1,024 tokens; the state is truncated first, never the question):

```
[CLS] <type> <question instructions> [SEP] <option text, when the head scores one option> [SEP] <state> [SEP]
```

`<type>` is a short word (`choice` / `score` / `noul`) so one encoder serves all heads. The state carries its context
(country, climate zone, season, farm or patient type, units; §6.0).

| Question type | Head | Output |
|---|---|---|
| `choice` (2–20 options) | **cross-encoder option scorer**: one pass per (question, option, state); a linear layer on the pooled `[CLS]` vector gives one score per option; softmax over the question's options | the chosen option + a probability per option |
| `score` (ordered levels) | the same option scorer plus an **ordinal** constraint (cumulative-link output over the ordered levels) | a level + probabilities; no position bias by design (levels are scored as text, order enters only the loss) |
| `noul` | **binary head** on `[CLS]` of (question, state) | P(true) |
| all | **escalate head** on `[CLS]` of (question, state) | P(escalate): the case should go to System 2 or a person (uncertain, out of domain, or high stakes) |

- **Why a cross-encoder for options:** it handles any number of options and any option text, and it needs no special
  tokens. Its cost is one pass per option; options of one question run as one batch.
- **Faster variant (v2, only if latency needs it):** a single pass with all options in one sequence and a pooled vector
  per option span. Kept only if it matches the cross-encoder's accuracy.
- **Calibration parameters** stored with the model: one temperature per (question type, option count bucket), §4 S4.

## 4. Training pipeline

| Stage | What | Data | Method | Output |
|---|---|---|---|---|
| **S0** | Build and clean the corpora and decision sets | §6 | dedup, language ID, licence filter, PII filter, biosecurity filter | versioned datasets + data card |
| **S1** | **Domain-adaptive pretraining** of the encoder | agri-biotech + medical text, plus general replay | continued masked-language modelling (mask ~15 %), seq 1,024, warm-up then decay | `encoder-dapt-vN` |
| **S2** | **General decision skills**: learn typed questions at all | public classification / NLI / intent / topic / QA sets rewritten as typed questions | supervised, **proper-scoring-rule losses**: log loss (cross-entropy) for `choice` / `noul`, ranked probability score for `score`, Brier as a check; option-order shuffling; soft labels where several answers are acceptable | `heads-general-vN` |
| **S3** | **Domain decisions** | agri-biotech + medical typed questions (§6.2) | as S2; **teacher labels** from a System 2 LLM for the grey zone (soft targets), ≥ 10 % human review per batch; balanced per domain, country and language | `agribiotech-vN` |
| **S4** | **Calibration** | held-out slice per question type × option count | temperature scaling (minimise NLL; report ECE) | temperatures in the model files |
| **S5** | **Task adaptation** (per deployment) | e.g. AIMTAS's farm question contract and data | short fine-tune, then S4 again | `agribiotech-aimtas-vN` |

Notes:
- **Replay:** keep 20–30 % general text in S1 and general decisions in S3 so the model keeps what mmBERT knew.
- **Positive control** before S2 and S3: a copy of the data with the answer planted in the state must reach ≈ 1.00; if not,
  the setup is wrong, not the task.
- **Seeds:** at least 3 per stage that ships; report the spread.
- **Later, optional:** a reinforcement-learning stage with a proper-scoring-rule reward, only if supervised training plateaus.
  It is not needed for the first releases.

## 5. Serving: our own `/v1/systemone` server

| Part | Design |
|---|---|
| Model files | safetensors (training) + **ONNX** export of encoder + heads, **int8** quantised for CPU (ONNX Runtime) |
| Server v1 | Python **FastAPI + ONNX Runtime** in a small Docker image: `POST /v1/systemone` (TypeSafe request/response shape), `GET /v1/models`, `GET /health`; API key from an environment secret; per-question batching; a request limit per key |
| Where it runs | a public **Hugging Face Space** (demo + API), the **AIMTAS cloud** cluster, and later **on the gateway itself** (Edge model, CPU), so System 1 keeps working while a farm is offline |
| In-process option (later) | the AIMTAS gateway / cloud (Java) can load the same ONNX file with ONNX Runtime for Java, with no HTTP hop |
| Compatibility tests | a **Jev-compatibility** contract suite: TypeSafe's documented request/response examples plus AIMTAS's real requests (question sets, typed answers, error codes) must pass against our server |

Model releases (planned, Hugging Face account `letrthang`), each a tagged revision (`v0.1`, …) so a server can pin or roll
back:

| Repo | Contents |
|---|---|
| `letrthang/thannong-agribiotech` | safetensors checkpoint, tokenizer, temperatures, `onnx/` (int8), model card, data card |
| `letrthang/thannong-agribiotech-edge` | the Edge variant, same layout |

## 6. Data

### 6.0 Focus countries and regional conditions (owner, 2026-10-11)

The model is built first for **Vietnam, USA, Canada, Australia, New Zealand, Japan, Korea, Norway, Finland and Sweden**.
Their conditions differ a lot, and a decision that is right in one can be wrong in another (a water temperature that is
normal for a Vietnamese shrimp pond is lethal for Norwegian salmon), so **place is part of the data**:

| Country | Climate (broadly) | Farming examples to cover (to confirm with domain experts) |
|---|---|---|
| Vietnam | tropical monsoon (north: humid subtropical) | shrimp and prawn, pangasius, tilapia, pigs, poultry, rice-fish |
| USA | temperate to subtropical, arid west | poultry, pigs, cattle, catfish and trout, row crops |
| Canada | cold temperate / continental | dairy, pigs, poultry, salmon (coastal), greenhouses |
| Australia | arid to tropical; drought and heat | sheep and cattle, poultry, prawns, barramundi |
| New Zealand | temperate maritime | dairy and sheep, green-lipped mussels, king salmon |
| Japan | humid subtropical to cool temperate; typhoons | poultry, pigs, wagyu, yellowtail and sea bream, oysters |
| Korea | humid continental, monsoon summers | pigs, poultry, flatfish (olive flounder), seaweed, greenhouses |
| Norway | cool maritime to subarctic | Atlantic salmon and trout (sea pens, RAS), dairy |
| Finland | boreal, long cold winters | dairy, pigs, poultry, rainbow trout (RAS) |
| Sweden | temperate to boreal | dairy, pigs, poultry, RAS fish farming |

What this means for each stage:

- **Every example carries its context:** country, climate zone, season (the hemisphere matters: Australia and New
  Zealand have winter in June–August), farm type, species or crop, and the units in use (°C / °F, mg/L, ppm). The state
  renderer writes these into the `state` text, so the same question can have different right answers by place.
- **Bands and thresholds come from the place:** normal ranges for water, air, soil and animal health are taken per
  country, species and season from that country's guidance (or AIMTAS Training Profiles), never one global table.
- **Regulation-aware labels:** where a decision depends on national rules (notifiable diseases, withdrawal periods,
  water discharge, animal welfare), labels follow that country's rules; the question says which country.
- **Balance and reporting:** a minimum share of S1 text and S3 decision data per country; every evaluation is reported
  **per country** (and per language and domain); a model that is much weaker for one country does not ship.
- **Languages:** decisions are asked and answered in English, Vietnamese and Japanese first. Sources in Korean, Norwegian,
  Finnish and Swedish can still be used as DAPT text (mmBERT reads all of them), and their key facts can be brought into
  the decision data in English with review; Korean as a question language can follow later.

Country sources to check first (national agriculture, fisheries, food-safety and veterinary bodies, research institutes and
universities). Licences differ; **check each before use** (US federal works are usually public domain; others vary):

| Country | Examples of official / research sources |
|---|---|
| Vietnam | the national ministry for agriculture and its research institutes; veterinary and fisheries departments |
| USA | USDA (incl. ARS, APHIS), NOAA Fisheries, FDA Center for Veterinary Medicine, land-grant university extension |
| Canada | Agriculture and Agri-Food Canada, Canadian Food Inspection Agency, Fisheries and Oceans Canada |
| Australia | CSIRO, the federal department of agriculture, state primary-industries departments |
| New Zealand | Ministry for Primary Industries, AgResearch / Crown research institutes |
| Japan | MAFF, NARO, the Fisheries Research and Education Agency |
| Korea | MAFRA, the Rural Development Administration, the National Institute of Fisheries Science |
| Norway | Norwegian Veterinary Institute, Norwegian Food Safety Authority, Institute of Marine Research |
| Finland | Natural Resources Institute Finland (Luke), the Finnish Food Authority |
| Sweden | Swedish University of Agricultural Sciences (SLU), the Swedish Board of Agriculture |

### 6.1 Text for DAPT (S1)

Target mixture **(est.)** for 5–20 billion tokens: agricultural biotechnology 35 % · medicine (veterinary first, then human)
25 % · AIMTAS documents 5–10 % · general replay 30 %. Within the domain text: as much Vietnamese and Japanese as licences
allow (aim ≥ 20 % each), the rest English; a minimum share per focus country (§6.0).

| Source (candidates) | Domain | Licence — **check before use** |
|---|---|---|
| PubMed abstracts (NLM baseline) | medical, veterinary, biotech | NLM terms; abstracts may carry publisher copyright |
| PubMed Central **Open Access** subset, commercial-use portion | medical, biotech | per-article licence (CC BY, CC0 …); use only commercial-use-compatible articles |
| FAO publications, AGROVOC thesaurus | agriculture, aquaculture, livestock | often CC BY-NC-SA IGO — **non-commercial terms may rule them out** |
| Open-access agriculture / aquaculture / veterinary journals | agri-biotech | per journal (CC BY preferred) |
| Wikipedia (en / vi / ja, plus ko / no / fi / sv) — science, agriculture, medicine categories | all | CC BY-SA |
| Government extension and veterinary guidance of the ten focus countries (§6.0) | agriculture, aquaculture, veterinary | per government; often reusable — verify |
| **AIMTAS own data**: SOPs, Training Profiles, simulator scenarios, anonymised logs (owner-approved) | farm operations | ours |

**Excluded:** patient records or any personal health data (e.g. MIMIC-type datasets), anything without a licence that
permits model training, personal data, and content that gives operational detail for enhancing pathogens (biosecurity
filter).

### 6.2 Decision data (S2, S3)

- **S2 general:** public classification, NLI, intent, sentiment, topic and QA sets rewritten into typed questions. Track
  each set's licence in the data card.
- **S3 domain**, built per domain with a fixed, versioned question contract, for example:

| Domain | Example questions |
|---|---|
| Aquaculture | urgency of a water-quality reading; which equipment fixes it; disease suspicion from signs |
| Livestock / poultry | house climate problem (`AIR_QUALITY` / `TEMPERATURE` / …); feed or water issue; biosecurity risk level |
| Crops / soil | nutrient deficiency class from symptoms; irrigation need |
| Animal health (veterinary) | which body system; refer to a vet now (`noul`); notifiable-disease suspicion (per country) |
| Medicine (human, later) | triage level of a symptom description (advice only); document type; is this passage relevant |
| Microbiology / lab | sample type; contamination suspicion; protocol step type |

  Labels come from (1) rules written with domain experts, (2) existing labelled public sets, and (3) a System 2 LLM as a
  **teacher** for the grey zone, with ≥ 10 % human review per batch. Hold-out sets are written or checked by people and
  never generated by the same templates.

## 7. Compute (est.)

| Stage | Hardware | Time | Notes |
|---|---|---|---|
| M1 spike (tiny heads, server, ONNX) | Mac (MPS) or free Kaggle T4 | hours | tiny data |
| S1 DAPT, Edge (140 M), ~5 B tokens | 1× A100 / H100 80 GB | ~0.5–1 day | rented |
| S1 DAPT, Cloud (307 M), ~10–20 B tokens | 1–4× A100 / H100 | ~1–4 days | rented; the largest cost (~US$100–1,000) |
| S2 + S3 heads | 1× 24–48 GB GPU (L4 / A10G / A100) | ~5–20 h per run, 3 seeds | |
| S4 calibration | CPU | minutes | |
| S5 AIMTAS task | free Kaggle 2× T4 | hours | |
| Serving | CPU (Space, cluster, gateway) | — | int8 ONNX ≈ 1 byte per parameter |

GPU rental prices change; check current rates before each run.

## 8. Evaluation

| Suite | Measures | Ship gate (proposal — owner sets the final numbers) |
|---|---|---|
| General typed decisions (public sets of §6.2) | accuracy, ECE | at least as good as the bare-encoder baseline trained on S2 only |
| Domain decision hold-outs (§6.2), per domain | accuracy, macro-F1, ECE | DAPT model beats the same heads on the non-DAPT encoder |
| **AIMTAS farm hold-out** (AIMTAS's shared eval set and question contract) | per question, per farm type (fish / prawn / chicken / pig / mixed), per language | **beats AIMTAS's current System 1 model**; urgency recall ≥ 0.95 |
| Languages | en / vi / ja | vi and ja ≤ 5 points below en |
| Countries (§6.0) | every suite broken down per focus country and season | no country more than 5 points below the average |
| Calibration | ECE after S4 | ≤ 0.10 |
| Robustness | answer stability with options reordered; ordinal position bias | no first/last-position bias |
| Speed | p50 / p95 per question: Edge on the gateway CPU, Cloud on a 2-vCPU Space | Edge p95 within the gateway's System 1 timeout with margin |

## 9. Safety, ethics and compliance

- **Advice only.** The model never acts; callers keep deterministic safety rules (AIMTAS: interlocks, envelopes, limits,
  modes, approvals). Every answer carries its probability and the escalate probability so callers can hand uncertain cases
  to System 2 or a person.
- **Medicine:** decision support for qualified users, **not a medical device**, no diagnosis or prescription; every model
  card says so. Human-health use starts only after veterinary results are solid and with an owner decision.
- **Data:** a **data card** lists every source, licence, date, filter and share. No personal or patient data. Anonymised
  AIMTAS logs only with the owner's approval. Public weights can leak training text, so nothing private goes in.
- **Biosecurity:** the model classifies and does not generate text; training text is still filtered for operational
  pathogen-enhancement content.
- **Fairness:** report results per language, domain, country and farm type; do not ship a model that is much worse for one
  group.

## 10. Repository layout (planned)

```
README.md                 what and why
architecture.md           this file
LICENSE                   Apache-2.0
data/                     source lists, licence register, data card (no large files in git)
  sources.yaml
  DATA_CARD.md
questions/                versioned question contracts per domain (e.g. aquaculture-v1.json)
pretrain/                 S1 DAPT configs and scripts
train/                    S2–S3 heads, S4 calibration, S5 task adaptation
eval/                     hold-out sets (small, committed) and eval scripts
export/                   ONNX export + int8 quantisation
serve/                    the /v1/systemone server (FastAPI + ONNX Runtime), Dockerfile, contract tests
reports/                  one report per training run
```

Weights and large datasets go to Hugging Face, never to git.

## 11. Milestones

| # | Milestone | Exit criteria |
|---|---|---|
| M0 | Scope and data register | licence: Apache-2.0 (decided 2026-10-11); owner picks the first medical scope; `sources.yaml` with licences checked |
| M1 | **End-to-end spike** | tiny heads on `mmBERT-small`, exported to ONNX int8, answered by our server on `POST /v1/systemone`; AIMTAS's real requests pass the contract tests |
| M2 | Corpora v1 | cleaned, deduplicated, licence-filtered DAPT text; token counts per domain, language and country |
| M3 | DAPT Edge | lower MLM loss on domain hold-out text than the base; general loss not worse by more than a small margin |
| M4 | Heads v1 (S2 + S3) on Edge | domain hold-outs: DAPT model beats the non-DAPT baseline |
| M5 | DAPT + heads on Cloud | as M3–M4 for the 307 M variant |
| M6 | AIMTAS task (S5) | beats AIMTAS's current System 1 model on the AIMTAS hold-out |
| M7 | Release | HF repos, model + data cards, ONNX, our server on a Space; AIMTAS switches by configuration after the owner's OK |

## 12. Relationship to AIMTAS

- AIMTAS's gateway and cloud call System 1 on `POST /v1/systemone`; switching to this model is a change of endpoint and
  key in their System 1 settings, after it passes M6.
- It reuses AIMTAS's **question contracts** and **farm hold-out set** (AIMTAS data, shared by every System 1 candidate) so
  the comparison is fair.
- The Edge variant can later run **on the gateway itself** (CPU), keeping System 1 working while a farm is offline.

## 13. Risks

| Risk | Mitigation |
|---|---|
| Our heads are not good enough on general typed questions | S2 on broad public data; positive controls; compare with the non-DAPT baseline at every milestone; keep the current AIMTAS System 1 until M6 passes |
| Cross-encoder cost with many options | batch options per question; keep `choice` ≤ 20 options; the single-pass v2 head if latency needs it |
| Our own server is more to maintain | small FastAPI service, contract tests, one Docker image for Space, cluster and gateway |
| Licences of agriculture/medical text block commercial use | licence register from M0; prefer CC BY / CC0 / public-domain sources; exclude NC sources |
| Domain shift forgets general skills | 20–30 % replay in S1 and S3; general eval in every report |
| Too little Vietnamese / Japanese domain text | translate selected English passages with review; collect VN / JP extension material; report per language |
| Data dominated by one or two countries (e.g. USA in English text) | minimum share per focus country; per-country reports; place in every `state` |
| Wrong thresholds across places (climate, season, species) | bands per country / species / season from local guidance or Training Profiles; never one global table |
| Medical misuse | advice-only design, model card limits, human-health scope gated by the owner |
| Cost overrun in DAPT | start with Edge (small), measure the gain per billion tokens before scaling |
