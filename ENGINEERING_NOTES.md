# Dossier: engineering notes

What we learned building and running Dossier: what worked, what didn't, how the second opinion works and what it
actually achieves. Dated, factual, numbers from the running system or the published eval. Newest first.

## 1. The first real bank statement (2026-11-02)

The synthetic forgery set was caught 100 % with 0 false alarms. The first **genuine** UK bank statement sent in by
email was flagged *balances don't add up*. Every row did add up; six problems were ours:

| What happened | Cause | General fix |
|---|---|---|
| Side-panel figures ("Average credit balance £606.00", "Start balance") taken as row balances | the text layer joins every word on the same height into one line, across the whole page | code learns each money column (paid in / out / balance) per page: the right edge that lines up with the most row lines, counting every occurrence of every value; a figure outside its column is not used |
| The same amount paid in and paid out (10,450.00 twice) boxed at the wrong one | the first match on the page won | values are re-boxed at the occurrence in their own column, on their row's line |
| "10450.00" vs the printed "10,450.00" scored as a near miss | fuzzy text match | an amount without grouping marks is an exact match, but only inside **one** printed word in a well-formed amount (a security review showed "12.34" "5.00" side by side would otherwise pass as 12,345.00) |
| Row dates printed without a year ("14 Sep") marked wrong | the date check needed a year | day + month must match; the year is accepted when the date falls inside the statement period |
| A four-line address not found | the verifier joined at most two lines | up to four consecutive lines |
| Two £538.00 payments on one day read as one row (or one row given the next row's balance) | the model merges / skips rows; it varied between four readings of the same file | see below |

**Skipped rows without a loophole.** Code finds every figure printed in a money column, inside the table, that no
extracted row is boxed on. The running balance is then re-walked in printed order: each row's money where its amount is
printed, each balance checked where *it* is printed, skipped amounts and unclaimed balances put back where they are.
Only if that walk adds up **everywhere** does the result become `STATEMENT_ROWS_MISSED` (a person checks the rows).
Otherwise `BALANCE_BROKEN` stays a failed check. Automated security reviews found three versions of this rule
exploitable along the way (a stray figure excusing any break; a copied balance hiding the real printed one; a merged
row) and each became a regression test.

**Result.** All four stored model readings of that statement now come out clean or "rows skipped, balances add up";
the forgery eval is unchanged (100/100 caught, 0 false alarms in the free truth mode). Regression tests use made-up
numbers; no real document is in the repository.

**Lesson.** Synthetic packs prove the checks catch what we forge; they don't prove the reading survives real layouts.
One real document found five problems the generator never produced.

## 2. The second opinion (TypeSafe Jev)

### What it is and what it sees
A separate decision model asked, after the checks, two questions about the case:

- `document_fraud_risk` (choice): **low** (consistent, clean files, every check passed) · **moderate** (something a
  person should glance at) · **high** (a likely forgery: a person must check the originals);
- `accept_without_review` (yes / no).

It sees the **evidence only**: the kinds of documents, how each was read, the reason codes the checks raised, counts
of the consistency matrix and the outcome so far. Never a name, address, date, number or file. The case keeps exactly
what was sent, and the app shows it ("What it saw").

### The rules
- Asked about **every** evaluated case (not while documents are still being read), once per state of the evidence
  (a hash of what it would see); same evidence, no second call.
- Answers are validated (probabilities 0–1 summing to ≈ 1, the claimed choice really the most likely); an invalid
  answer counts as no answer. 2.5 s timeout + one retry. **Fail-open:** no key, an error or a bad answer leaves the
  checks' outcome as it is.
- It can only make the outcome **stricter**:
  - every check passed + `high` with probability ≥ 0.6 → **needs a person** (`SECOND_OPINION_RISK`, warn);
  - the case already needs a person or failed a check → its view is recorded, a high risk is added as an info note;
  - it **never** clears a finding or accepts a case.
- Agreement is measured against the checks: on a passing case it agrees when the risk is low or it would accept; on a
  case a person sees anyway it agrees when the risk isn't low or it wouldn't accept.

### What it achieved (published eval, real model reading, 44 synthetic applicant packs, 2026-11-02)

| Packs | Caught by the checks | Second opinion: low / moderate / high | Agrees |
|---|---|---|---|
| Salary raised in the PDF (4) | 4 | 0 / 0 / 4 | 4 |
| Salary raised, balances recalculated (4) | 4 | 0 / 0 / 4 | 4 |
| Amount retyped over a white box, other font (4) | 4 | 0 / 0 / 4 | 4 |
| Retyped in the same font (4) | 4 | 0 / 0 / 4 | 4 |
| Statement remade in Word (4) | 4 | 0 / 3 / 1 | 4 |
| Name changed on a bill (4) | 4 | 0 / 0 / 4 | 4 |
| Another applicant's statement (4) | 4 | 0 / 3 / 1 | 4 |
| Template-farm statement, broken balances (4) | 4 | 0 / 4 / 0 | 4 |
| Passport MRZ altered (4) | 4 | 0 / 0 / 4 | 4 |
| Bill seven months old (4) | 4 | 0 / 4 / 0 | 4 |
| **Genuine (4)** | 4 clean | **2 / 2 / 0** | 2 |

- Agreed with the checks on **42 / 44**; raised the risk on **40 / 40** forged packs; sent **0 / 4** genuine packs to a
  person. On 2 genuine packs it noted a *moderate* doubt: recorded, outcome unchanged.
- With the generator's exact readings (free "truth" mode, 44 packs): 44 / 44 agree, every genuine pack *low*.
- Live (9 cases on 2026-11-02): 9 / 9 agree; median answer ≈ 200 ms.

### What it is for, and what it is not
It reads the checks' own codes, so **it cannot find a forgery the checks missed** (and in the eval it found none).
Its value is an independent, cheap sanity check on the final call, with a documented risk band per case, and a
safety net if a future check passes something odd-looking. It is not a detector, and the pages say so.

### What didn't work at first
- The first rule asked only on cases where every check passed ("the only outcome it can change"). In practice it
  was asked on 1 case out of 11, the team saw an empty panel everywhere else, and there was nothing to measure.
  Docket (every document) and Front Desk (every lead) were already asking always; Dossier now does too.
- A file already read (the same bytes, sent again by email or through an upload link) is not read again, so the
  worker never triggered the second opinion for that case. Those requests now ask in a background task.
- The published eval summary had `model: null`: the reading model was never recorded. Fixed, with a test.

## 3. Detection eval: how and why

- A generator builds an applicant's pack (bank statement, utility bill, passport) as real PDFs, then forges one
  document the way fraudsters do. Ten forgery types; each "caught" only if the case is not *consistent* **and** a code
  meant to catch that forgery fired.
- Modes: `forensics` (the file checks alone, free), `truth` (every check with the generator's exact reading, free),
  `model` (the production model reads every document, a few cents; this is what is published).
- Genuine packs are measured as hard as forgeries: a checker that flags genuine applicants gets switched off.

## 4. App and data lessons

- **Erased applicants kept counting** as cases ("Reading", pending) in the app and on the public numbers. An erased
  case is now a record that it existed, not a case.
- **Samples next to real work.** One click loads eight synthetic applicants; without a filter they bury the real
  ones. The review queue keeps real cases, samples and "waiting for the applicant" apart.
- **Product words in shared settings.** Dossier reused Docket's settings panels and their explanations said
  "invoices". Shared components now take each product's own texts.
- **Live, not mock-up.** Every number on the Overview comes from one endpoint over the database; the pipeline board
  moves on server-sent events from the worker.

## 5. Principles that held

- The model reads, code decides: every value must be found on the page before it counts; the outcome is decided by
  versioned rules, never by the model.
- Documents, not people: the outcome says what is wrong with a document; a person decides about the applicant.
- Fail closed on fraud, fail open on the optional: a check that can't run sends the case to a person; an unavailable
  second opinion changes nothing.
- Every security review finding becomes a test.
