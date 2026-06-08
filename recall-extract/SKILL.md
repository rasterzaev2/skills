---
name: recall-extract
description: "Extract structured data from an incoming SEPA fraud-recall email/file (FR/ES/NL/EN/IT/DE, often multi-message threads with embedded transfer tables) into a single canonical JSON object. Use when an agent is handed a path to a recall file and must produce the recall JSON. Pure extraction — no network calls, no actions."
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
Apply these rules, then fill the EXACT template in Step 3.

### Output contract (read this first — it is mandatory)
- Output **exactly the keys in the Step 3 template — no more, no fewer.** Do not invent new
  top-level or nested keys. Do not drop keys. Do not rename keys.
- If something does not fit an existing key, record it in `flags` (a string) — never as a new key.
- Anything absent → `null` (scalars) or `[]` (arrays/objects-of-lists). Never omit the key.
- Keep the key **order** as in the template.
- Output raw JSON only — no prose, no markdown fences, no comments.

### Core rules
1. **Beneficiary IBAN is ALWAYS the Finom-held account that received the funds.** Identify by ANY of: BIC starts with `FNOM`; the IBAN contains `FNOM`; or the email explicitly names it as the account "at your entity" (`de su entidad`, `su filial`, `votre établissement`, `your bank/entity`, `begunstigde ... bij u`). Put it in `finom_beneficiary_iban`, set `finom_iban_verified: true`. If you cannot confirm an IBAN belongs to Finom, set `finom_iban_verified: false` and add `beneficiary_not_confirmed_finom` to `flags`.
2. **Sender IBAN is the debited / originating (victim) account** — "donneur d'ordre", "compte débiteur", "cuenta emisor", "sender account", "afzender". Put in `sender_iban`. If only a partial/internal ref is given (e.g. `BBVA 0182-4464`), leave `sender_iban: null`, put the raw value in `sender_account_ref`, add `sender_iban` to `missing_fields` and `sender_iban_partial_only` to `flags`.
3. **Never confuse direction.** The bank writing to us is the counterparty/victim side; the funds landed at Finom. Sender = victim account; Beneficiary = Finom account.
4. `counterparty_bank` = the bank that SENT us this recall (contact/reply-to). `originator_bank` = the bank of the sender/victim account (`name`, `bic`); often the same as the counterparty — if so, repeat it; if unknown, set both subfields to null.
5. Normalize IBANs to uppercase, no spaces. If an IBAN looks malformed, keep it and add `invalid_iban_format` to `flags`.
6. **Amounts → JSON numbers.** Parse European formats: `1.350,00`→1350.0, `349,3`→349.3, `634,15`→634.15. (Integer-valued amounts like `800` are fine; the backend normalizes display to 2 decimals.)
7. **Dates → ISO `YYYY-MM-DD`.** Parse `dd/mm/yyyy`, `dd-mm-yyyy`, `dd.mm.yyyy`. `value_date` = transaction/execution date (NOT the email date). `received_at` = email date (date only) if present, else null.
8. **One object per transfer in `transactions[]`.** Many recalls contain several transfers (tables, "X€ et Y€", multiple rows). Capture each. `total_amount` = sum of `executed` transfers. Add `multiple_transactions` to `flags` when there is more than one. Rows clearly `declined`/`pending`/"tentative" are excluded from `total_amount` (but still listed with the right `status`).
9. **`transactions[].status` is an enum: `executed` | `pending` | `declined` | `null`.** Map source wording: `EXECUTE`/`Exécuté`/`Executed`/`Ejecutada`/`Uitgevoerd`/`Eseguito` → `executed`; `tentative`/`en cours`/`pending`/`processing` → `pending`; `rejected`/`rejeté`/`declined`/`failed` → `declined`; unknown → `null`. Always lowercase the enum value.
10. `remittance_info` (per transaction) = the payment label / "Motif" / "Référence bénéficiaire" / concept text on that transfer, else null.
11. `recall_type`: FRAD = fraud / authorised push payment (default for fraud wording); DUPL = duplicate; TECH = technical; RFRO = request-for-recall / consent-based. Unclear → `UNKNOWN`.
12. `fraud_typology` = short category (e.g. "fausse annonce auto", "phishing", "APP fraud", "cavalerie"). `recall_reason` = a short free-text reason in the email's own terms, else null.
13. Response deadline ("15 días", "15 days to respond") → `deadline_business_days` (int) else null.
14. Handling instructions → `special_instructions` (e.g. "do not change subject", "keep same thread", "do not use terminal", "forward to Finom Italy branch", "return by rejecting the transfer").
15. `police_report.mentioned`/`.attached`: booleans. List attachment filenames in `attachments` (`[]` if none).
16. Default `currency` to `"EUR"` when a € amount is present and no other currency is stated.
17. **Never invent a value.** Anything absent → null, and add the field name to `missing_fields`.
18. `extraction_confidence`: `high` | `medium` | `low` — how clean/complete the source was.

### Step 3 — Fill EXACTLY this template
Return this object, same keys, same order, filled from the file. `[]` where there is nothing.
```json
{
  "source_file": null,
  "email_subject": null,
  "channel": "email",
  "language": null,
  "counterparty_bank": { "name": null, "contact_name": null, "contact_email": null, "reply_to": null },
  "originator_bank": { "name": null, "bic": null },
  "client_name": null,
  "recall_type": "FRAD",
  "recall_reason": null,
  "fraud_typology": null,
  "sender_name": null,
  "sender_iban": null,
  "sender_account_ref": null,
  "finom_beneficiary_iban": null,
  "finom_bic": null,
  "finom_iban_verified": false,
  "beneficiary_name": null,
  "transactions": [
    { "amount": 0, "currency": "EUR", "value_date": null, "reference": null, "transaction_id": null, "remittance_info": null, "status": null }
  ],
  "total_amount": 0,
  "currency": "EUR",
  "received_at": null,
  "deadline_business_days": null,
  "police_report": { "mentioned": false, "attached": false },
  "attachments": [],
  "special_instructions": null,
  "extraction_confidence": "high",
  "missing_fields": [],
  "flags": []
}
```

## Step 4 — Output
Print the filled JSON object and nothing else — no prose, no markdown fences. Set `source_file` to
the input path. This output is your entire result; the Lead/Mapper consume it verbatim.
