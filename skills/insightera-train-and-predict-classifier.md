---
name: Train and run a custom text classifier on InsightEra
description: >-
  Train a customer-owned text-classification model on InsightEra's NLP Platform, inspect it,
  predict against it, retrain it and delete it. Use when you need a bespoke Thai text classifier
  rather than InsightEra's stock analysis endpoints.
api: openapi/insightera-nlp-platform-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/insightera-nlp-platform-openapi.yml
operations:
  - POST /nlp/classification/train
  - POST /nlp/classification/train-with-file
  - GET /nlp/classification/token
  - POST /nlp/classification/model
  - POST /nlp/classification/predict
  - POST /nlp/classification/retrain
  - POST /nlp/classification/change-model-name
  - POST /nlp/classification/delete
---

# Train and run a custom text classifier on InsightEra

These are the only **stateful** operations on the InsightEra NLP Platform API. Everything else is
a pure transform; these create, mutate and destroy models that persist server-side under your
token. Read the retry-safety section before you automate any of it.

## Before you start

- Base URL: `https://nlp.insightera.co.th/api`, credential is `?token=<your-token>` on every call.
- **The token is the tenancy boundary.** `GET /nlp/classification/token` returns `model_id_list` —
  the models belonging to the calling token. There is no separate account or workspace id.
- **There is no sandbox.** Training creates a real model on the production service. There is no
  scratch namespace, so evaluation and production share one model space.

## Steps

1. **Train** — `POST /nlp/classification/train` with `{"data": ..., "is_sync": <bool>,
   "model_name": "<name>"}`. Returns `result`.
   - `is_sync: true` blocks until training finishes. Combined with the declared HTTP 408, a large
     synchronous training set can time out with no documented budget to size against.
   - `is_sync: false` returns immediately. **There is no webhook or callback** — poll
     `POST /nlp/classification/model` to find out when it is ready.
   - To train from a file instead, use `POST /nlp/classification/train-with-file` as
     `multipart/form-data` with a `file` part and an optional `model_name` part. This is one of
     only two multipart operations on the whole API (the other is `/nlp/ocr`).

2. **Find your model id** — `GET /nlp/classification/token`. The only `GET` on the API. Returns
   `model_id_list` for the calling token. Use this rather than assuming the train response gave
   you a durable handle.

3. **Inspect** — `POST /nlp/classification/model` with `{"model_id": "<id>"}`. Returns `message`.
   Note this is a read that is modelled as a POST.

4. **Predict** — `POST /nlp/classification/predict` with `{"model_id": "<id>", "samples": [...]}`.
   Batch: `samples` is an array. Returns `message`.

5. **Retrain** — `POST /nlp/classification/retrain` with `{"modelId": "<id>", "data": ...,
   "is_sync": <bool>}`. **Watch the casing**: retrain takes `modelId` while predict, model detail
   and rename take `model_id`. The contract is inconsistent here and it is an easy silent failure.

6. **Rename** — `POST /nlp/classification/change-model-name` with `{"model_id": "<id>",
   "model_name": "<new name>"}`.

7. **Delete** — `POST /nlp/classification/delete`. Returns `result` plus `error_msg`.
   Check `error_msg`, not just the HTTP status — this operation reports failure in the body.

## Retry safety — read this

**There is no idempotency key on this API.** No `Idempotency-Key` parameter or header exists
anywhere in the contract. That has real consequences here:

- A retried `train` or `train-with-file` creates a **second model**, not the same one. A network
  timeout on training that actually succeeded leaves you with a duplicate you must find via
  `GET /nlp/classification/token` and clean up with `delete`.
- A retried `retrain` re-runs training against the model again.
- `delete` and `change-model-name` are naturally idempotent by outcome, but give you no way to
  confirm the first attempt landed except by re-listing.

Before retrying any create or mutate call, **list with `GET /nlp/classification/token` and
reconcile** rather than firing again blind.

## Handling failure

Errors are a flat `{"message": "..."}` string with no machine-readable code.

| Status | Meaning | Action |
|---|---|---|
| 400 | `Invalid session token` — the live service's real response to a bad credential, despite the contract documenting 401. | Fix the token; do not retry. |
| 401 | Declared on every operation, not observed live. | Auth failure; do not retry. |
| 403 | `This token has no quota allowed on this service`. Undeclared in the contract. | Standing quota decision — stop and contact InsightEra. |
| 408 | Request Timeout. | Most likely on synchronous training. Switch to `is_sync: false` or shrink the dataset. Reconcile before retrying. |
| 500 | Internal Server Error. | Back off, then reconcile with `GET /nlp/classification/token` before any retry. |

No `Retry-After` and no `RateLimit-*` headers are returned on any response, so back-off timing is
entirely your choice.

## Support

`dev@insightera.co.th` (from `info.contact` in the published contract). There is no status page
and no request-id header, so include the exact timestamp, path and model id when reporting an
issue — the service gives you no correlation handle to quote.
