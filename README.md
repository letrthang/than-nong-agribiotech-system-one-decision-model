# Thannong-Agribiotech — System One SLM for agricultural biotechnology and medicine

**Thannong-Agribiotech** (Thần Nông Agribiotech) is a planned **System One small language model (SLM)**: a fast *decision*
model, not a chatbot. You give it a **state** (a sensor reading, an operator message, a lab note, a document passage) and
**typed questions**; it returns typed answers with probabilities in one forward pass. It is built **from day 1** for
**agricultural biotechnology** (aquaculture, livestock and poultry, crops and soil, animal health, feed, microbiology,
biosecurity) and **medicine** (veterinary and human health, with a One Health view that links the two).

> **Status (2026-10-11): design.** Nothing is trained yet. This repo holds the plan and, later, the training code.
> The model AIMTAS uses today is **Laya** (Option A, below). See [`architecture.md`](architecture.md) for the full design.

## Why a new model (Option B)

The AIMTAS smart-farm platform runs a two-system AI: a fast **System 1** that classifies, filters and gates, and a large
language model as **System 2** that reasons. Three ways to get System 1 were compared (2026-10-11):

| Option | What | Where |
|---|---|---|
| **A — fine-tune Laya** | Take `convaiinnovations/laya-multilingual` (mmBERT encoder + decision head, 322 M) and fine-tune it on AIMTAS farm data | **first**, in [`aimtas-system-one-laya-slm`](https://github.com/letrthang/aimtas-system-one-laya-slm) |
| **B — our own Laya-like model** | Start from a pretrained encoder, adapt it to agri-biotech + medical text, and train **our own** decision head | **this repo** |
| C — from scratch | Pretrain an encoder too | out of scope (cost, and open encoders already exist) |

Option B gives what A cannot:

- **Domain knowledge from the first day.** The encoder is further pretrained on agricultural-biotech and medical text
  *before* any task training, so it reads terms like *Vibrio parahaemolyticus*, *coccidiosis*, *dissolved oxygen sag* or
  *biofloc* as familiar words, not rare tokens.
- **Our own head and question types**, sized for our callers, and a model we fully control (data, licence, releases).
- **A choice of size** — a small variant for the edge gateway's CPU and a larger one for the cloud.

Option A ships first. Option B replaces it only where it **beats Option A on the same hold-out tests**.

## What it does (and does not do)

- **Does:** typed decisions — `choice` (pick one of the options), `score` (an ordinal level), `noul` (yes/no with a
  probability); calibrated probabilities; English, Vietnamese and Japanese first, other languages from the multilingual base.
- **Does not:** write text, diagnose, prescribe, or act. It is **advice for deterministic callers**: AIMTAS keeps interlocks,
  limits and modes in code; in medicine, a qualified person decides. It is **not a medical device**.

## API

It will speak the same **Jev-compatible System One API** as Laya — `POST /v1/systemone`
([TypeSafe System One docs](https://docs.typesafe.ai/introduction)) — so AIMTAS's gateway and cloud can switch models
without code changes:

```sh
curl -s https://<server>/v1/systemone -H "Authorization: Bearer <API_KEY>" -H "Content-Type: application/json" -d '{
  "state": "Broiler house 2, day 18: ammonia 32 ppm (limit 20), rising since 03:00; fans 1-2 off.",
  "questions": {
    "problem": {"type": "choice", "instructions": "What is the main problem?",
                "criteria": {"AIR_QUALITY": "ammonia or CO2 too high", "TEMPERATURE": "too hot or cold",
                             "DISEASE_SIGN": "signs of disease", "NONE": "nothing wrong"}},
    "urgent":  {"type": "noul", "instructions": "Does this need action now?"}}}'
```

## Where things will live

| What | Where |
|---|---|
| Plan, training code, eval sets | this repo (public) |
| Trained weights (safetensors) | Hugging Face `letrthang/thannong-agribiotech` (planned) |
| GGUF for llama.cpp | Hugging Face `letrthang/thannong-agribiotech-GGUF` (planned) |
| Live server | a Hugging Face Space or the AIMTAS cluster, like Laya's |

## Open decisions (owner)

1. **Licence** of this repo and of the model (Apache-2.0 recommended — it matches Laya and allows commercial use; the
   repo has no `LICENSE` yet).
2. **Which medical scope** comes first: veterinary (closest to AIMTAS) or human health too.
3. **Head layout:** Laya-compatible (servable by today's llama.cpp) or our own design (needs our own server) —
   [`architecture.md`](architecture.md) §5.

## Read next

- [`architecture.md`](architecture.md) — the model, data, training stages, serving, evaluation, safety and milestones.
- [Laya training plan (Option A)](https://github.com/letrthang/aimtas-system-one-laya-slm) — the question contract and
  farm eval set that Option B reuses.
