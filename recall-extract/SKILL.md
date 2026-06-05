---
name: recall-extract
description: >
  Extract structured data from an incoming SEPA fraud-recall email/file (FR/ES/NL/EN/IT/DE,
  often multi-message threads with embedded transfer tables) into a single canonical JSON
  object. Use when an agent is handed a path to a recall file and must produce the recall JSON.
  Pure extraction — no network calls, no actions.
---

# Recall Extract

Read ONE incoming fraud-recall file and emit a single canonical JSON object. You do not
act on anything, call any backend, or move data anywhere — you read and output JSON only.

## When to use
The agent is given a path to a recall file (PDF / .eml / .html / .txt / image) and must
return the structured recall data.

## When NOT to use
- The task asks you to start a process, call an API, set limits, or move money — that is the
  Mapper's job, not this skill.
- The file is clearly not a fraud recall — still output JSON, but with
  `extraction_confidence: "low"` and a `flags` entry explaining why.

## Step 1 — Get the text of the file
Input is an absolute file path. Read its text first:

- **PDF (text-based, like forwarded bank emails):**
  `pdftotext -layout "<path>" -` (preferred — keeps table columns aligned).
  If `pdftotext` is unavailable: `python3 -c "import pdfplumber,sys; print('\n'.join((p.extract_text() or '') for p in pdfplumber.open(sys.argv[1]).pages))" "<path>"`.
- **Scanned / image-only PDF or image:** OCR with `tesseract "<path>" - -l eng+fra+spa+nld` then proceed. Set `extraction_confidence` no higher than `"medium"`.
- **.eml / .html / .txt:** read directly.

Never guess the content — work only from the extracted text.

## Step 2 — Extract into the schema
Apply these rules, then output the JSON in Step 3.

### Core rules
1. **Beneficiary IBAN is ALWAYS the Finom-held account that received the funds.** Identify by ANY of: BIC starts with `FNOM`; the IBAN contains `FNOM`; or the email explicitly names it as the account "at your entity" (`de su entidad`, `su filial`, `votre établissement`, `your bank/entity`, `begunstigde ... bij u`). Put it in `finom_beneficiary_iban`, set `finom_iban_verified: true`. If you cannot confirm an IBAN belongs to Finom, set `finom_iban_verified: false` and add `beneficiary_not_confirmed_finom` to `flags`.
2. **Sender IBAN is the debited / originating (victim) account** — "donneur d'ordre", "compte débiteur", "cuenta emisor", "sender account", "afzender". Put in `sender_iban`. If only a partial/internal ref is given (e.g. `BBVA 0182-4464`), leave `sender_iban: null`, put the raw value in `sender_account_ref`, add `sender_iban` to `missing_fields` and `sender_iban_partial_only` to `flags`.
3. **Never confuse direction.** The bank writing to us is the counterparty/victim side; the funds landed at Finom. Sender = victim account; Beneficiary = Finom account.
4. Normalize IBANs to uppercase, no spaces. If an IBAN looks malformed, keep it and add `invalid_iban_format` to `flags`.
5. **Amounts → numbers, dot decimal, two decimals.** Parse European formats: `1.350,00`→1350.00, `349,3`→349.30, `634,15`→634.15.
6. **Dates → ISO `YYYY-MM-DD`.** Parse `dd/mm/yyyy`, `dd-mm-yyyy`, `dd.mm.yyyy`. `value_date` = transaction/execution date (NOT the email date). `received_at` = email date (date only) if present, else null.
7. **One object per transfer in `transactions[]`.** Many recalls contain several transfers (tables, "X€ et Y€", multiple rows). Capture each. `total_amount` = sum of executed/claimed transfers. Ignore rows clearly marked declined / "tentative" / not executed for the sum, but you MAY note them in `flags`.
8. `recall_type`: FRAD = fraud / authorised push payment (default for fraud wording); DUPL = duplicate; TECH = technical; RFRO = request-for-recall / consent-based. Unclear → `UNKNOWN`.
9. Response deadline ("15 días", "15 days to respond") → `deadline_business_days` (int) else null.
10. Handling instructions → `special_instructions` (e.g. "do not change subject", "keep same thread", "do not use terminal", "forward to Finom Italy branch", "return by rejecting the transfer").
11. `police_report.mentioned`/`.attached`: booleans. List attachment filenames in `attachments`.
12. Default `currency` to `"EUR"` when a € amount is present and no other currency is stated.
13. **Never invent a value.** Anything absent → null, and add the field name to `missing_fields`.
14. `extraction_confidence`: `high` | `medium` | `low` — how clean/complete the source was.

### Schema (return exactly these keys)
```json
{
  "source_file": "string|null",
  "email_subject": "string|null",
  "channel": "email",
  "language": "string",
  "counterparty_bank": { "name": "string|null", "contact_name": "string|null", "contact_email": "string|null", "reply_to": "string|null" },
  "client_name": "string|null",
  "recall_type": "FRAD|DUPL|TECH|RFRO|UNKNOWN",
  "fraud_typology": "string|null",
  "sender_name": "string|null",
  "sender_iban": "string|null",
  "sender_account_ref": "string|null",
  "finom_beneficiary_iban": "string|null",
  "finom_bic": "string|null",
  "finom_iban_verified": true,
  "beneficiary_name": "string|null",
  "transactions": [
    { "amount": 0.0, "currency": "EUR", "value_date": "string|null", "reference": "string|null", "transaction_id": "string|null", "status": "string|null" }
  ],
  "total_amount": 0.0,
  "currency": "EUR",
  "received_at": "string|null",
  "deadline_business_days": null,
  "police_report": { "mentioned": false, "attached": false },
  "attachments": ["string"],
  "special_instructions": "string|null",
  "extraction_confidence": "high|medium|low",
  "missing_fields": ["string"],
  "flags": ["string"]
}
```

## Step 3 — Output
Print the JSON object and nothing else — no prose, no markdown fences. Set `source_file` to the
input path. This output is your entire result; the Lead/Mapper consume it verbatim.
