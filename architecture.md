# Thannong-Agribiotech — architecture

*Design, 2026-10-11. Nothing here is built yet. Numbers marked **(est.)** are planning estimates to check in the first
experiments, not measurements.*

## 0. Summary

Thannong-Agribiotech is a **System One decision model**: an encoder that reads text, plus a small **decision head** that
answers typed questions with calibrated probabilities. It follows the proven Laya recipe (Option B of the AIMTAS plan),
with two differences that make it ours:

1. **Domain-adaptive pretraining (DAPT)** of the encoder on agricultural-biotech and medical text, so it knows the domain
   before it learns any task.
2. **Our own decision head**, trained by us, released under our licence, in two sizes (edge and cloud).

It serves the same `POST /v1/systemone` API as Laya, so AIMTAS callers can switch between the two by configuration.

```
                 ┌──────────── training (this repo) ─────────────┐
 public encoder  │ S1 DAPT        S2 head:       S3 domain    S4 calibrate │   GGUF / ONNX     /v1/systemone
 mmBERT-small ──►│ agri-biotech ► general typed ► decisions ► + S5 AIMTAS  │──► HF model repo ─► llama.cpp or own server
 mmBERT-base     │ + medical MLM  decisions(RL)   (RL)          task tune  │                    (HF Space / cluster)
                 └─────────────────────────────────────────────────────────┘
```

## 1. Goals and non-goals

**Goals**
- Typed decisions (`choice`, `score`, `noul`) for agricultural biotechnology and medicine, in **English, Vietnamese and
  Japanese** first, for the **ten focus countries** of §6.0 first.
- Calibrated probabilities that callers can gate on (AIMTAS's System 1 confidence gate, escalation to System 2).
- Two sizes: **Edge** (runs on the AIMTAS gateway's CPU, ≤ ~150 M parameters) and **Cloud** (~300 M).
- Drop-in for Laya on `POST /v1/systemone` (Jev-compatible, [TypeSafe docs](https://docs.typesafe.ai/introduction)).
- Beat Option A (fine-tuned Laya) on the AIMTAS farm hold-out before it replaces it.

**Non-goals**
- No text generation, chat, diagnosis or prescription. System 2 (an LLM) writes text; people make medical decisions.
- No actuation. Callers keep every safety rule in deterministic code.
- No pretraining of an encoder from scratch (Option C).

## 2. The reference: how Laya works

Facts from the `convaiinnovations/laya` and `laya-multilingual` model cards (read 2026-10-11):

| Part | Laya-multilingual |
|---|---|
| Encoder | `jhu-clsp/mmBERT-base` — ModernBERT architecture, 22 layers, hidden 768, 256k vocabulary, 307 M parameters, pretrained on 3T+ tokens in 1,800+ languages (MIT) |
| Decision head | 2 transformer layers, an **option-marker scorer** (each option sits at its own `[MASK]` token; scores are softmaxed over that question's options, so options are defined per request) and an **act/escalate** head. ~15 M parameters; 322 M total |
| Budget | 1,024 tokens per question, of which 256 for the question and its options (the encoder reads up to 8,192) |
| Training | the head trained from scratch with **RLCD**: the policy reports a distribution, exploration adds zero-mean Gaussian noise to the logits, the reward is a strictly proper scoring rule (log + spherical; ranked probability score for ordinal questions), updates are REINFORCE with a group-mean baseline. 15,987 updates, 4 epochs, ≈ 5 h |
| Known limits | ships uncalibrated (fit temperatures per question type × option count); `noul` can under-report "true"; `score` has a position bias; near chance zero-shot on specialised workflows — "a fast base to specialise" |

We reuse the **ideas** (option markers, typed questions, proper-scoring-rule training, calibration). Their weights are not
part of Option B.

## 3. Model

### 3.1 Encoder

| Variant | Start from | Params | Layers / hidden | Why |
|---|---|---|---|---|
| **Thannong-Agribiotech-Edge** | `jhu-clsp/mmBERT-small` | 140 M (42 M without embeddings) | 22 / 384 | fast on the gateway CPU; multilingual |
| **Thannong-Agribiotech** (cloud) | `jhu-clsp/mmBERT-base` | 307 M (110 M without embeddings) | 22 / 768 | best quality per size among open multilingual encoders; Laya's own base |
| (option) English-only | `answerdotai/ModernBERT-large` | ~395 M | 28 / 1024 | only if a deployment is English-only |

- **Tokenizer: keep mmBERT's 256k vocabulary unchanged.** Adding tokens would need new, untrained embeddings; DAPT teaches
  the domain's words through the existing sub-word pieces instead. Measure the token count of domain text before and after
  (a term split into many pieces is a hint, not a blocker).
- Most parameters are the vocabulary table (~200 M of mmBERT-base's 307 M). If the Edge variant is still too slow on the
  gateway, the first lever is **vocabulary pruning** to the languages we serve, then quantisation (Q8_0).

### 3.2 Decision head (ours)

Same contract as Laya's so the callers and the API do not change:

- **Input:** `[CLS] question + options (each option prefixed by its own [MASK]) [SEP] state [SEP]`, ≤ 1,024 tokens, ≤ 256 for
  the question block.
- **Layers:** 2 transformer layers over the encoder output (hidden = encoder hidden), then:
  - **option scorer** — a linear score at each option's `[MASK]` position → softmax over that question's options (`choice`);
    the same with ordered levels plus an ordinal loss (`score`); two options true/false (`noul`);
  - **act/escalate head** — one probability that the caller should hand the case to a person or System 2.
- **One forward pass per question** (as in Laya); a request with *n* questions = *n* passes, batched.
- **Calibration parameters** stored with the model: one temperature per (question type, option count).

Whether the head must match Laya's *exact* tensor names and sizes is the servability question of §5.

## 4. Training pipeline

| Stage | What | Data | Method | Output |
|---|---|---|---|---|
| **S0** | Build and clean the corpora and decision sets | §6 | dedup, language ID, licence filter, PII filter | versioned datasets + data card |
| **S1** | **Domain-adaptive pretraining** of the encoder | agri-biotech + medical text, plus general replay | continued masked-language modelling (MLM ~15 %), seq 1,024, learning-rate warm-up then decay | `encoder-dapt-vN` |
| **S2** | **General decision skills** — learn typed questions at all | public classification / NLI / QA / routing sets rewritten as `choice` / `score` / `noul` | soft cross-entropy warm-up, then RLCD with proper scoring rules | `head-general-vN` |
| **S3** | **Domain decisions** | agri-biotech + medical typed questions (§6.2) | RLCD; option shuffling; balanced per domain and language | `agribiotech-vN` |
| **S4** | **Calibration** | held-out slice per question type × option count | temperature scaling (minimise NLL / ECE) | temperatures in the model files |
| **S5** | **Task adaptation** (per deployment) | e.g. AIMTAS `farm-v1.json` data from Option A | short fine-tune, then S4 again | `agribiotech-aimtas-vN` |

Notes:
- **Replay:** keep 20–30 % general text in S1 and general decisions in S3 so the model does not forget what mmBERT knew.
- **Positive control** before every RL stage (as Laya's guide advises): a copy of the data with the answer planted in the
  state must reach ≈ 1.00; if not, the setup is wrong.
- **Seeds:** at least 3 per stage that ships; report the spread.

## 5. Serving — the head-layout decision

| Path | How | Pros | Cons |
|---|---|---|---|
| **5a. Laya-compatible layout** (recommended first) | Save the checkpoint in Laya's layout (`model.safetensors`, `encoder/config.json`, `rl_agent_config.json`, `tokenizer/`) with the same head modules and names; convert with llama.cpp's `convert_hf_to_gguf.py` (it detects a Laya decision model and writes `decision.type = laya`) | runs on the same llama.cpp `llama-server` and HF Space as Laya; GGUF Q8_0 on CPU | our head is limited to what llama.cpp's Laya support implements; must re-check on every llama.cpp upgrade |
| **5b. Our own server** | ONNX export + a small server (e.g. Python FastAPI + ONNX Runtime, or Java + ONNX Runtime) implementing `/v1/systemone` | any head design, any question type | we own a server: auth, limits, health, packaging |

**Milestone M1 is a spike that decides this early:** train a tiny head on `mmBERT-small` in the Laya layout, convert it, and
call `/v1/systemone` on llama.cpp. If it works, 5a; if not, 5b.

Model releases (planned, Hugging Face, owner account `letrthang`):

| Repo | Contents |
|---|---|
| `letrthang/thannong-agribiotech` | safetensors checkpoint, tokenizer, temperatures, model card, data card |
| `letrthang/thannong-agribiotech-GGUF` | `thannong-agribiotech-Q8_0.gguf` (path 5a) |
| `letrthang/thannong-agribiotech-edge` / `-edge-GGUF` | the Edge variant |

Each release is a tagged revision (`v0.1`, …) so a server can pin or roll back.

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
allow (aim ≥ 20 % each), the rest English.

| Source (candidates) | Domain | Licence — **check before use** |
|---|---|---|
| PubMed abstracts (NLM baseline) | medical, veterinary, biotech | NLM terms; abstracts may carry publisher copyright |
| PubMed Central **Open Access** subset — commercial-use portion | medical, biotech | per-article licence (CC BY, CC0 …); use only commercial-use-compatible articles |
| FAO publications, AGROVOC thesaurus | agriculture, aquaculture, livestock | often CC BY-NC-SA IGO — **non-commercial terms may rule them out** |
| Open-access agriculture / aquaculture / veterinary journals | agri-biotech | per journal (CC BY preferred) |
| Wikipedia (en / vi / ja) — science, agriculture, medicine categories | all | CC BY-SA |
| Government extension and veterinary guidance of the ten focus countries (§6.0) | agriculture, aquaculture, veterinary | per government; often reusable — verify |
| **AIMTAS own data**: SOPs, Training Profiles, simulator scenarios, anonymised logs (owner-approved) | farm operations | ours |

**Excluded:** patient records or any personal health data (e.g. MIMIC-type datasets), anything without a licence that
permits model training, personal data, and content that gives operational detail for enhancing pathogens (biosecurity
filter).

### 6.2 Decision data (S2, S3)

- **S2 general:** public classification, NLI, intent, sentiment, topic and QA sets rewritten into typed questions (the
  Laya family was benchmarked on tasks of this kind). Track each set's licence in the data card.
- **S3 domain**, built per domain with a fixed question contract, for example:

| Domain | Example questions |
|---|---|
| Aquaculture | urgency of a water-quality reading; which equipment fixes it; disease suspicion from signs |
| Livestock / poultry | house climate problem (`AIR_QUALITY` / `TEMPERATURE` / …); feed or water issue; biosecurity risk level |
| Crops / soil | nutrient deficiency class from symptoms; irrigation need |
| Animal health (veterinary) | which body system; refer-to-vet now (`noul`); notifiable-disease suspicion |
| Medicine (human, later) | triage level of a symptom description (advice only); document type; relevant-passage (`keep_passage`) |
| Microbiology / lab | sample type; contamination suspicion; protocol step type |

  Labels come from (1) rules written with domain experts, (2) existing labelled public sets, (3) a System 2 LLM as a
  **teacher** for the grey zone, with ≥ 10 % human review per batch. Hold-out sets are written or checked by people and never
  generated by the same templates.

## 7. Compute (est.)

| Stage | Hardware | Time | Notes |
|---|---|---|---|
| M1 spike | Mac (MPS) or free Kaggle T4 | hours | tiny data |
| S1 DAPT, Edge (140 M), ~5 B tokens | 1× A100 / H100 80 GB | ~0.5–1 day | rented |
| S1 DAPT, Cloud (307 M), ~10–20 B tokens | 1–4× A100 / H100 | ~1–4 days | rented; the largest cost (~US$100–1,000) |
| S2 + S3 head (RLCD) | 1× 24–48 GB GPU (L4 / A10G / A100) | ~5–20 h per run, 3 seeds | Laya's head: ≈ 5 h |
| S4 calibration | CPU | minutes | |
| S5 AIMTAS task | free Kaggle 2× T4 | hours | same as Option A |
| Serving | CPU (HF Space, gateway) | — | Q8_0 GGUF ≈ 1 byte per parameter |

GPU rental prices change; check current rates before each run.

## 8. Evaluation

| Suite | Measures | Ship gate (proposal — owner sets the final numbers) |
|---|---|---|
| General typed decisions (same sets Laya reports) | accuracy, ECE | not worse than Laya-multilingual by more than 2 points |
| Domain decision hold-outs (§6.2), per domain | accuracy, macro-F1, ECE | beats Laya-multilingual + Option A on each domain |
| **AIMTAS farm hold-out** (Option A's `test-holdout.jsonl`, `farm-v1.json`) | per question, per farm type (fish / prawn / chicken / pig / mixed), per language | **beats Option A**; urgency recall ≥ 0.95 |
| Languages | en / vi / ja | vi and ja ≤ 5 points below en |
| Countries (§6.0) | every suite broken down per focus country and season | no country more than 5 points below the average |
| Calibration | ECE after S4 | ≤ 0.10 |
| Robustness | answer stability with options reordered; position bias on `score` | no first/last-position bias |
| Speed | p50 / p95 per question: Edge on the gateway CPU, Cloud on a 2-vCPU Space | Edge p95 ≤ Laya's on the same CPU |

## 9. Safety, ethics and compliance

- **Advice only.** The model never acts; callers keep deterministic safety rules (AIMTAS: interlocks, envelopes, limits,
  modes, approvals). Every answer carries its probability and the act/escalate score so callers can hand uncertain cases to
  System 2 or a person.
- **Medicine:** decision support for qualified users, **not a medical device**, no diagnosis or prescription; every model
  card says so. Human-health use starts only after veterinary results are solid and with an owner decision.
- **Data:** a **data card** lists every source, licence, date, filter and share. No personal or patient data. Anonymised
  AIMTAS logs only with the owner's approval. Public weights can leak training text, so nothing private goes in.
- **Biosecurity:** the model classifies and does not generate text; training text is still filtered for operational
  pathogen-enhancement content.
- **Fairness:** report results per language, domain and farm type; do not ship a model that is much worse for one group.

## 10. Repository layout (planned)

```
README.md                 what and why
architecture.md           this file
data/                     source lists, licence register, data card (no large files in git)
  sources.yaml
  DATA_CARD.md
questions/                versioned question contracts per domain (e.g. aquaculture-v1.json)
pretrain/                 S1 DAPT configs and scripts
train/                    S2–S3 head training (RLCD), S4 calibration, S5 task adaptation
eval/                     hold-out sets (small, committed) and eval scripts
convert/                  GGUF (path 5a) or ONNX (path 5b) export
serve/                    only if path 5b: our /v1/systemone server
reports/                  one report per training run
```

Weights and large datasets go to Hugging Face, never to git.

## 11. Milestones

| # | Milestone | Exit criteria |
|---|---|---|
| M0 | Scope and data register | licence decided: Apache-2.0 (2026-10-11); owner picks the first medical scope; `sources.yaml` with licences checked |
| M1 | **Serving spike** | a tiny head on `mmBERT-small` answers on `/v1/systemone` (path 5a) — or the decision for 5b |
| M2 | Corpora v1 | cleaned, deduplicated, licence-filtered DAPT text; token counts per domain and language |
| M3 | DAPT Edge | lower MLM loss on domain hold-out text than the base, general loss not worse by more than a small margin |
| M4 | Head v1 (S2 + S3) on Edge | domain hold-outs beat Laya-multilingual zero-shot |
| M5 | DAPT + head on Cloud | as M3–M4 for the 307 M variant |
| M6 | AIMTAS task (S5) | beats Option A on the AIMTAS hold-out → candidate for AIMTAS System 1 |
| M7 | Release | HF repos, model + data cards, GGUF, served on a Space; AIMTAS switches by config after the owner's OK |

## 12. Relationship to Option A and to AIMTAS

- **Option A** (fine-tuned Laya, repo `aimtas-system-one-laya-slm`) is AIMTAS's System 1 until Option B wins its hold-out.
- Both share: the `POST /v1/systemone` API, the question contracts (`farm-v1.json` …), the state renderer, and the AIMTAS
  hold-out set — so the comparison is fair and the switch is configuration only (the gateway and cloud System 1 endpoint
  settings).
- Option B's Edge variant can later run **on the gateway itself** (CPU), keeping System 1 working while a farm is offline.

## 13. Risks

| Risk | Mitigation |
|---|---|
| Our head ends up worse than Laya's | S2 general decision training with Laya-style data; compare against Laya at every milestone; keep Option A until B wins |
| llama.cpp cannot serve our head | M1 spike first; path 5b as fallback |
| Licences of agriculture/medical text block commercial use | licence register from M0; prefer CC BY / CC0 / public-domain sources; exclude NC sources if the model is commercial |
| Domain shift forgets general skills | 20–30 % replay in S1 and S3; general eval in every report |
| Too little Vietnamese / Japanese domain text | translate selected English passages with review; collect VN / JP extension material; report per language |
| Data dominated by one or two countries (e.g. USA in English text) | minimum share per focus country; per-country reports; place in every `state` |
| Wrong thresholds across places (climate, season, species) | bands per country / species / season from local guidance or Training Profiles; never one global table |
| Medical misuse | advice-only design, model card limits, human-health scope gated by the owner |
| Cost overrun in DAPT | start with Edge (small), measure the gain per billion tokens before scaling |
