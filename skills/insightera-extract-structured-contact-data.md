---
name: Extract structured Thai contact data from text and images
description: >-
  Turn unstructured Thai text or a scanned image into structured contact data using InsightEra's
  NLP Platform — OCR an image, then extract a structured Thai address and any email addresses.
  Use for order forms, delivery slips, chat transcripts and social posts.
api: openapi/insightera-nlp-platform-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/insightera-nlp-platform-openapi.yml
operations:
  - POST /nlp/ocr
  - POST /nlp/cleaning
  - POST /nlp/address-extractor
  - POST /nlp/extract-email
  - POST /nlp/datetime-parser-new
---

# Extract structured Thai contact data from text and images

This is the flow InsightEra's Thai address extractor is built for: an unstructured blob — a chat
message, a social comment, a photographed delivery slip — becomes labelled fields.

## Before you start

- Base URL: `https://nlp.insightera.co.th/api`, credential is `?token=<your-token>` in the query
  string on every call.
- **This flow handles personal data.** Addresses, phone numbers and email addresses are being sent
  to a third-party processor in Thailand. InsightEra publishes a Thai privacy policy but no DPA,
  no PDPA compliance statement and no data-retention statement for API payloads, so establish your
  own lawful basis and retention position before sending real customer data.
- There is no sandbox; every call is production.

## Steps

1. **If the input is an image, OCR it first** — `POST /nlp/ocr` as `multipart/form-data` with an
   `image` file part. Returns `result` plus `status`. This is one of only two multipart operations
   on the API. No maximum file size or accepted format list is published.

2. **Clean the text** — `POST /nlp/cleaning` with `{"text": "<raw or OCR output>"}`. OCR output in
   particular benefits from this before extraction.

3. **Extract the address** — `POST /nlp/address-extractor` with `{"text": "<cleaned>"}`. Per the
   contract's own description, this labels:
   1. Name
   2. Phone number
   3. Tambol (sub-district)
   4. Amphoe (district)
   5. Province
   6. Postcode
   7. Probability of the input actually being an address, `0.0` to `1.0`

   **Use field 7 as a gate.** The operation returns a structure for any input; the probability is
   the only signal distinguishing a real address from a hallucinated parse of unrelated text.
   Pick a threshold and discard below it rather than trusting the labelled fields blindly.

4. **Extract email addresses** — `POST /nlp/extract-email` with `{"texts": [...]}`. Batch: takes an
   array, returns `results` plus `status`. Note this takes `texts` (plural array) while
   `address-extractor` takes `text` (single string) — the two are not interchangeable.

5. **Parse any dates mentioned** — `POST /nlp/datetime-parser-new` with `{"text": "<cleaned>"}`,
   for delivery dates or appointment times in the same blob. Returns `status` plus `timestamp`.

## Handling failure

Errors are a flat `{"message": "..."}` string with no machine-readable code.

| Status | Meaning | Action |
|---|---|---|
| 400 | `Invalid session token` — the live response to a bad credential, not the documented 401. | Fix the token; do not retry. |
| 403 | `This token has no quota allowed on this service` — per-token, **per-service** quota, undeclared in the contract. | Stop. Quota is granted per service, so OCR can be denied while tokenize works. |
| 408 | Request Timeout — most likely on a large image or a long `texts` batch. | Shrink and retry with backoff. |
| 500 | Internal Server Error. | Retry with backoff; escalate to dev@insightera.co.th. |

## Retry safety

All operations in this flow are stateless transforms with no server-side effect, so retries are
safe and cost only quota. There is no idempotency key on this API, but none is needed here.

## Privacy note

Because the credential travels in the query string and the payloads carry personal data, do not
log full request URLs, and do not retain InsightEra responses longer than your own policy allows.
No request-id is returned, so there is no correlation handle to hold instead of the payload.
