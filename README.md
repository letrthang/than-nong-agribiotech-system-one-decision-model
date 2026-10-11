# Thannong-Agribiotech — System One SLM for agricultural biotechnology and medicine

**Thannong-Agribiotech** (Thần Nông Agribiotech) is a planned **System One decision model**, a small language model (SLM)
**compatible with [TypeSafe's Jev](https://docs.typesafe.ai/introduction)**: it speaks the same System One API and answers the same typed questions, so a caller
built for Jev can use it unchanged. It is a fast *decision* model, not a chatbot. You give it a **state** (a sensor reading, an operator message, a lab note, a document passage) and
**typed questions**; it returns typed answers with probabilities. It is built **from day 1** for **agricultural
biotechnology** (aquaculture, livestock and poultry, crops and soil, animal health, feed, microbiology, biosecurity) and
**medicine** (veterinary and human health, with a One Health view that links the two).

**The target: our flagship System One model.** Trained and optimised from day 1 for agricultural biotechnology and
medicine, it works the way TypeSafe's [Jev](https://docs.typesafe.ai/introduction) works (for example
[`~typesafe/jev-latest` on OpenRouter](https://openrouter.ai/~typesafe/jev-latest)): a decision model behind a stable API
that any System One caller can use. In AIMTAS it is the planned **replacement for the current System 1 model,
[`aimtas-system-one-laya-slm`](https://github.com/letrthang/aimtas-system-one-laya-slm)**.

> **Status (2026-10-11): design.** Nothing is trained yet. This repo holds the plan and, later, the training code and the
> server. See [`architecture.md`](architecture.md) for the full design.

## What makes it ours

It is built from a **bare pretrained encoder**, [`jhu-clsp/mmBERT`](https://huggingface.co/jhu-clsp/mmBERT-base) (MIT
licence; reads 1,800+ languages), with **nothing taken from any existing decision model** (owner, 2026-10-11):

- **Domain knowledge from the first day.** The encoder is further pretrained on agricultural-biotech and medical text
  before any task training, so it reads terms like *Vibrio parahaemolyticus*, *coccidiosis*, *dissolved oxygen sag* or
  *biofloc* as familiar words.
- **Our own decision heads, our own training, our own server**, designed, trained and released by us.
- **Two models:**
  - **Thannong-Agribiotech-Edge**: the small one, used by the **AIMTAS gateway on the farm** (runs on its CPU, also offline);
  - **Thannong-Agribiotech-Skynet**: the bigger one, used by the **AIMTAS cloud server**.

It is the long-term System One of the **AIMTAS** smart-farm platform, where a fast System 1 classifies, filters and gates,
next to a large language model as System 2:

- **Replaces `aimtas-system-one-laya-slm`** as AIMTAS's System 1 (Edge on the farm gateway, Skynet on the cloud server)
  once it scores better on the same AIMTAS hold-out tests; the owner decides the switch.
- **Hosted like Jev:** a public endpoint with keys and limits (a Hugging Face Space first), versioned releases, a model card
  and published evaluation numbers. Listing it with model providers, as Jev is on OpenRouter, is a later step the owner
  decides.

## What it does (and does not do)

- **Does:** typed decisions — `choice` (pick one of the options), `score` (an ordinal level), `noul` (yes/no with a
  probability); calibrated probabilities; an **escalate** probability that tells the caller when to ask System 2 or a person.
- **Does not:** write text, diagnose, prescribe, or act. It is **advice for deterministic callers**: AIMTAS keeps interlocks,
  limits and modes in code; in medicine, a qualified person decides. It is **not a medical device**.

## Focus: languages and countries

- **Languages first:** English, Vietnamese, Japanese.
- **Countries first:** **Vietnam, USA, Canada, Australia, New Zealand, Japan, Korea, Norway, Finland, Sweden, Singapore.** Farming
  depends on place: climate and weather, soil, water, species and breeds, diseases, and each country's rules. So the
  training data, the domain knowledge and the tests are collected and reported **per country**. Every training example
  carries its country and climate zone, and a model is judged on each country, not only on the average.
  See [`architecture.md`](architecture.md) §6.0.

## API

**[Jev](https://docs.typesafe.ai/introduction)-compatible:** our server speaks TypeSafe's **System One API**, `POST /v1/systemone`
([TypeSafe System One docs](https://docs.typesafe.ai/introduction)): the same request (`state` + typed `questions` of type
`choice` / `score` / `noul`), the same typed answers with probabilities, so it is a drop-in alternative to Jev. AIMTAS's gateway and cloud already call that API, so
they can switch to this model by configuration:

```sh
curl -s https://<server>/v1/systemone -H "Authorization: Bearer <API_KEY>" -H "Content-Type: application/json" -d '{
  "state": "Broiler house 2, Vietnam (tropical), day 18: ammonia 32 ppm (limit 20), rising since 03:00; fans 1-2 off.",
  "questions": {
    "problem": {"type": "choice", "instructions": "What is the main problem?",
                "criteria": {"AIR_QUALITY": "ammonia or CO2 too high", "TEMPERATURE": "too hot or cold",
                             "DISEASE_SIGN": "signs of disease", "NONE": "nothing wrong"}},
    "urgent":  {"type": "noul", "instructions": "Does this need action now?"}}}'
```

## Where things will live

| What | Where |
|---|---|
| Plan, training code, eval sets, server code | this repo (public) |
| Edge model (farm gateway) | Hugging Face `letrthang/thannong-agribiotech-edge` (planned) |
| Skynet model (cloud server, bigger) | Hugging Face `letrthang/thannong-agribiotech-skynet` (planned) |
| CPU inference export (ONNX, int8) | the same Hugging Face repos, `onnx/` folder (planned) |
| Live server | our own `/v1/systemone` server image: a Hugging Face Space for the public, the AIMTAS cluster and gateway for the farms |

## Open decision (owner)

1. **Which medical scope** comes first: veterinary (closest to AIMTAS) or human health too.

## Licence

**Apache-2.0** (owner, 2026-10-11) — this repository ([`LICENSE`](LICENSE)) and the released model weights. The base
encoder mmBERT is MIT. Training data keeps its own licences; see the data card.

## Read next

- [`architecture.md`](architecture.md) — the model, heads, data, training stages, server, evaluation, safety and milestones.
