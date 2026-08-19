---
name: Analyze Thai text with the InsightEra NLP Platform
description: >-
  Run a Thai text corpus through InsightEra's NLP Platform API to produce cleaned text, tokens,
  part-of-speech tags, named entities and sentiment. Use when you need Thai-language analysis and
  hold an InsightEra service token.
api: openapi/insightera-nlp-platform-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/insightera-nlp-platform-openapi.yml
operations:
  - POST /nlp/cleaning
  - POST /nlp/tokenize
  - POST /nlp/pos
  - POST /nlp/ner
  - POST /nlp/sentiment-new
---

# Analyze Thai text with the InsightEra NLP Platform

Every operation below was verified present in InsightEra's published Swagger 2.0 contract. The
contract publishes **no operationIds**, so steps are identified by method and path — the stable
ids in `overlays/insightera-nlp-platform-overlay.yaml` are ours, not the provider's, and will not
be recognised by InsightEra's own tooling.

## Before you start

- Base URL: `https://nlp.insightera.co.th/api`
- Every request needs `?token=<your-token>` in the **query string**. There is no header auth and no
  OAuth. Tokens come from InsightEra directly; there is no self-service signup.
- Because the credential rides in the URL, never log full request URLs and never paste one into a
  shared console or an issue tracker.
- There is no sandbox. Every call is production and consumes quota.

## Steps

1. **Clean the raw text** — `POST /nlp/cleaning` with `{"text": "<raw>"}`. Returns `result`.
   Do this first when the input came from social posts or scraped HTML.

2. **Tokenize** — `POST /nlp/tokenize` with `{"engine": "<engine>", "text": "<cleaned>"}`.
   Returns `result`. `engine` is a selectable string; the contract does not enumerate the valid
   values, so read the default in the spec and treat anything else as unverified.

3. **Tag parts of speech** — `POST /nlp/pos` with `{"texts": ["...", "..."]}`. This operation is
   **batch**: it takes an array, not a single string. Returns `message` plus `status`.

4. **Recognize named entities** — `POST /nlp/ner` with `{"texts": [...]}`. Also batch. Returns
   `message` plus `status`.

5. **Score sentiment** — `POST /nlp/sentiment-new` with `{"engine": "<engine>", "texts": [...]}`.
   Note the path suffix `-new`: there is no `/nlp/sentiment`, and no record of what the original
   was or when it changed.

## Batching

`/nlp/pos`, `/nlp/ner`, `/nlp/sentiment-new` and `/nlp/extract-email` accept a `texts` array;
`/nlp/clustering`, `/nlp/common-phrase` and `/nlp/classification/predict` accept `samples`.
**No maximum batch size is published.** All 23 operations declare HTTP 408 Request Timeout, so
size batches conservatively and back off on 408 rather than assuming a limit.

## Handling failure

Errors are a flat `{"message": "..."}` string with no machine-readable code, so you must
string-match. The status codes that matter, and what they really mean:

| Status | What it actually is | What to do |
|---|---|---|
| 400 | `Invalid session token` — missing or bad credential. **The live service returns 400 here, not the documented 401.** | Fix the token. Do not retry. |
| 401 | Declared on every operation but not observed live. | Treat as an auth failure. Do not retry. |
| 403 | `This token has no quota allowed on this service` — per-token, per-service quota. **Undeclared in the contract.** | Stop. Retrying will not help; request quota from InsightEra. |
| 408 | Request Timeout. | Reduce batch size or payload, then retry with backoff. |
| 500 | Internal Server Error. | Retry with backoff; escalate to dev@insightera.co.th. |

**Do not build a retry loop around 403.** It reads like a transient throttle but is a standing
quota decision, and no `Retry-After` or `RateLimit-*` header is returned to tell you otherwise.

## Retry safety

There is no idempotency key. The operations in this skill are stateless pure transforms, so a
retried call is harmless — it just costs quota again. That is **not** true of the classification
operations; see `insightera-train-and-predict-classifier.md`.
