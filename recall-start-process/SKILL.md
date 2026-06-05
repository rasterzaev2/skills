---
name: recall-start-process
description: >
  Validate an extracted recall JSON, compute an idempotency key, and start the recall
  process in the Finom backend (creates a case for human review — it does NOT move money
  or change limits). Use when an agent has a recall JSON and must open the case in the backend.
---

# Start Recall Process

Take a recall JSON (produced by the `recall-extract` skill), validate it, and call the Finom
backend to open the recall case. The endpoint only **creates a case in a PREPARED state** —
every real action (limits, refunds) is approved by a human later inside the process. You do not
move money or change limits here.

## When to use
You have a recall JSON and the task is to start/open the recall process in the backend.

## When NOT to use
- You were asked to extract data from a file — that is the `recall-extract` skill.
- The JSON fails the validation gate below — then do NOT call the backend; return `needs_manual_review`.

## Endpoint (hackathon: no auth)
The intake endpoint is open for the hackathon — no token, no authorization header.

```
INTAKE_URL = https://REPLACE-ME.finom.dev/api/recalls/intake
```

Replace `REPLACE-ME...` with the real URL in this file (one place, Step 3). That URL is the only
config — there is no environment variable and no auth header.

## Step 1 — Validation gate (hard stop if it fails)
Refuse to call the backend unless ALL are true:
- `finom_beneficiary_iban` is present AND `finom_iban_verified == true`
- at least one `transactions[]` entry with a numeric `amount > 0`
- `total_amount` present and equals the sum of executed transfers (recompute; if mismatch, fix it before sending, and add `total_recomputed` to flags you pass on)
- `extraction_confidence != "low"`

If any fail → output `{"result":"needs_manual_review","reason":"<which checks failed>", "extraction": <the JSON>}` and STOP. Do not POST.

## Step 2 — Idempotency key
Compute a stable key so a re-sent email never opens a duplicate case:

```bash
KEY=$(printf '%s|%s|%s|%s|%s' \
  "$BENEFICIARY_IBAN" "$TOTAL_AMOUNT_2DP" "$EARLIEST_VALUE_DATE" "$COUNTERPARTY_BANK_NAME" "$EMAIL_SUBJECT" \
  | sha256sum | cut -d' ' -f1)
```

(`TOTAL_AMOUNT_2DP` formatted like `3650.00`; `EARLIEST_VALUE_DATE` = min `value_date` across transactions.)

## Step 3 — POST to the backend
No auth header (hackathon). POST to the hardcoded URL:

```bash
curl -sS -X POST "https://REPLACE-ME.finom.dev/api/recalls/intake" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $KEY" \
  -d "$(jq -n --arg k "$KEY" --arg ch "email" --arg fr "$FILE_REF" --arg subj "$EMAIL_SUBJECT" \
        --argjson ext "$EXTRACTION_JSON" \
        '{idempotency_key:$k, source:{channel:$ch, file_ref:$fr, email_subject:$subj}, extraction:$ext}')"
```

## Step 4 — Interpret the response & report
Expected: `{ "recall_case_id": "...", "status": "PREPARED|NEEDS_REVIEW|DUPLICATE", "existing": bool }`.

- `PREPARED` / `NEEDS_REVIEW` → report `recall_case_id` + `status` as your result.
- `DUPLICATE` or `existing: true` → report the existing `recall_case_id` and note it was a duplicate (do not retry).
- Non-2xx → retry the POST exactly once; if it still fails, report **blocked** with the HTTP status/body, owner: backend, action: investigate intake endpoint. Never silently drop a recall.

Output a short JSON result: `{"result":"started","recall_case_id":"...","status":"...","idempotency_key":"...","duplicate":false}`.
