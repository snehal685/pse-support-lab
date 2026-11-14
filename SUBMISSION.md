# Submission — Invoicely Lite Product Support Engineer Round 2

- **Name:** Snehal Vharambale
- **Date submitted:** 14 November 2026
- **Repository link:** https://github.com/snehal685/pse-support-lab
- **Total time spent (real time):** Not accurately tracked; exact total hours are unavailable.

> **Verification status — important:** The written investigation below was prepared from repository code, assignment documentation, sample workbooks and Python reproductions supplied in `evidence/analysis-output.txt`. The local stack has since been started successfully in Docker and the Portal at `http://localhost:3000` was visually confirmed to load. The initial API import-list query showed only five historical October imports (no November customer batches at that point). The first `make load` attempt failed during the tools image build because `go mod download` hit a TLS handshake timeout to `proxy.golang.org`; a direct Ubuntu `curl -I https://proxy.golang.org` returned HTTP/2 200. A retry was started, but its final result has not yet been captured in this document.
>
> **Do not treat predicted runtime outcomes as observed facts.** Grafana/Loki/Prometheus/ASP/DB checks, invoice-by-invoice delivery states, the proposed code patches, and post-load reproduction are still pending verification. Wherever this document describes expected logs, counts, OOM status, or ASP responses, it is an investigation hypothesis or a query to run—not captured evidence. No customer email was actually sent; all replies below are drafts only. Do not submit until the pending evidence is either collected or clearly disclosed as unavailable.

> Replies are drafts on the **exercise clock** (Mon 16 Nov 2026, Asia/Dubai, UTC+4).

- **Items I fully investigated, and why:** 2 (platform-wide P1, hits every customer and a go-live), 1 (P1, month-end, duplicate risk, same root-cause family), 4 (32 of 40 invoices silently not delivered), 3 (compliance/auditor, clear product defect). Item 5 is a short, complete write-up because it is working-as-documented. I chose by priority and blast radius, not by ease.

**Priority / working order:** Inc 2 → Inc 1 → Inc 4 → Inc 3 → Inc 5. Acknowledgements go out in timestamp order (Inc 1 first at 09:20), but I work Inc 2 first because it is a shared fault: a restarting API can corrupt every other investigation (lost queue, 502s).

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:20 (received 09:05)
- **Reply:**

> Subject: Re: URGENT - monthly invoice upload stuck on "Processing"
>
> Hello Rashid,
>
> Thank you for the detail — I understand this is month-end and you need to know exactly what has gone out. I'm Product Support at Invoicely Lite and I'm looking at your three uploads now.
>
> **Please do not upload the file again for now.** Your "already exists" messages are the portal protecting you from issuing the same invoice number twice, and re-uploading adds to the confusion. I will tell you invoice by invoice what has and hasn't gone to the network, and the safe way to send the rest.
>
> You'll hear from me by **10:00** with that list, whether or not I have the full answer yet.
>
> Regards, Snehal Vharambale, Product Support, Invoicely Lite

### Priority and justification

- **Priority:** P1
- **Why:** Matrix question 1 — Oasis cannot get its invoices to the ASP at all (the file never completes in the UI), plus month-end close and a customer who has stopped acting for fear of duplicates (tax-document integrity). It is also the first visible symptom of a platform defect (see Inc 2).
- **SLA deadlines:** first response by 09:35 (sent 09:20) / next update 10:00, then every 30–60 min / resolution target 13:05.

### Investigation log

| Time | What I checked / did | What I found |
|---|---|---|
| 09:25 | Read `samples/oasis-builders-nov-2026.xlsx` | 600 rows, 8 invoices × 75 lines, **only 5 distinct buyers** (`evidence/analysis-output.txt`). "A bigger month" = 600 rows vs ~12 for a normal file. |
| 09:35 | `importer.Import` (`importer.go:35-127`) | **Synchronous**: the HTTP request does parse + validate every invoice, then returns. Upload page shows "Processing" until then. |
| 09:45 | `validate.validateRow` (`validate.go:165-174`) | Calls `dir.IsRegistered(buyer)` **once per row, with no cache** → 600 sequential SMP lookups. |
| 09:50 | `asp-mock/main.go:40,335` | Each lookup sleeps `lookupLatency = 200 ms` → 600 × 0.2 s ≈ **120 s** for one file (before DB round trips; validation also does a DB duplicate check per invoice). |
| 10:00 | `web/app/api/[...path]/route.ts:6,25` | The Next proxy aborts at `UPSTREAM_TIMEOUT_MS = 60 000` → customer sees a 504/"something went wrong"/spinner, while… |
| 10:05 | `importer.go:38` | …the API uses `context.Background()` with a 10-min budget, so it **keeps working after the browser gave up**. The upload actually completes server-side. |
| 10:10 | `store.InsertInvoices` / `ExistingStatus` (`store.go:282-308`) | All invoices of an import are inserted as `Parsed` **before** validation; `ExistingStatus` matches any status except `Rejected`, from *other* imports. So a second upload's copies collide with the first's (AE-BR-09), and while the first is still validating it can collide with the *second's* `Parsed` rows too. |
| 10:15 | Dead end | Considered ASP rate limit (2/s, burst 8): Oasis has 8 invoices, so it does not trip. Considered DB lock/slow query: no evidence, the sequential 200 ms lookups already explain >60 s. |

### Root cause

- **What happened:** The consolidated file takes ~2 min to process synchronously, longer than the portal's 60 s proxy timeout. The customer saw a stuck/failed upload and re-uploaded twice. Each attempt created its own import in the background; the later copies of each invoice number were rejected as AE-BR-09 against the earlier copies. Nothing was "issued twice" — the portal blocked duplicates — but the customer's picture is now fragmented across 3 imports.
- **Where:** `validate.go:166` (uncached per-row lookup), `importer.go:35-127` (synchronous import), `route.ts:6` (60 s timeout), `store.go:298` (duplicate check counts in-flight `Parsed` rows from sibling imports).
- **Why:** per-row external call × 600 rows × 200 ms, run inside the request; plus no feedback to the user, which invites retries.
- **Evidence:** code references above; `evidence/analysis-output.txt` (600 rows / 5 buyers / 120 s). *To confirm on the running stack:* `make replay CUSTOMER=oasis` and time it; Loki `{service="api"} | json | msg="import completed"` → `duration_ms` ≈ 120 000+; `{service="web"}` → `status:504 durationMs≈60000`; Grafana "participant lookup duration" panel; SQL below.
- **Confidence / open questions:** High on cause of slowness (arithmetic matches). Which invoices ended up where depends on exact timing → must be read from the DB.

**Query to answer Rashid's question 1** (run on prod DB):
```sql
SELECT i.invoice_number, i.status, i.rejected_by, i.asp_message_id, i.import_id, i.created_at
FROM invoices i JOIN sellers s ON s.id=i.seller_id
WHERE s.name ILIKE 'Oasis Builders%' AND i.created_at::date='2026-11-16'
ORDER BY i.invoice_number, i.created_at;
```
Cross-check against the ASP: `curl "http://localhost:8090/v1/documents?sender=<Oasis Peppol id>"` — the ASP list is the source of truth for what reached the network. Expected: for each of the 8 numbers, at most **one** non-rejected row; the rest `Rejected / portal / AE-BR-09`.

### Customer resolution reply OR escalation note

- **Which one and why:** **Update to customer now + escalation (L3)** for the product fix. Support can read the data and tell Rashid what is where, but cannot make the portal handle large files.

> **Update (10:00 draft — fill the table from the query above before sending):**
> Hello Rashid, here is where your November invoices stand. [table: invoice number → Accepted / Rejected-duplicate]. *Accepted* means the network has it and it is on its way to your customer. A *Rejected – already exists* line is a harmless duplicate of an invoice that is already accepted: nothing was issued twice. For the invoices that are not yet accepted, **please do not re-upload the whole file** — that is what creates the duplicates. Tell me and I will [retry them from our side / send you a small file containing only those numbers]. The slow screen is a fault on our side (large files take longer than the portal waits); I have passed it to engineering and will update you by **11:00**.

> ### Escalation: Large (600-row) uploads exceed the 60 s proxy timeout and trigger duplicate re-uploads
> **Priority:** P1 — month-end close, tax invoices not issued, customer blocked.
> **Customer / seller:** Oasis Builders Supply LLC (Sharjah). **Reported:** Rashid Al Mansoori, email 09:05.
> **Affected:** 3 imports from today ~09:00; 8 invoices OBS-2611-001…008; some rejected AE-BR-09 against earlier copies.
> **Symptoms:** Upload sits on "Processing", then error page; duplicates "already exist".
> **Evidence:** 600 rows, 5 distinct buyers; `validate.go:166` one `IsRegistered` per row, asp-mock sleeps 200 ms (`main.go:335`) ⇒ ~120 s; `route.ts:6` 60 s; `importer.go:38` background ctx continues after client abort.
> **Repro:** `make replay CUSTOMER=oasis`.
> **Ruled out:** ASP rate limit (8 invoices), DB.
> **Suspected cause:** as above, high confidence.
> **Workaround:** none for customer; support can retry/clean up per invoice.
> **Ask:** (1) cache participant lookups per file/with TTL (patch in repo: `directory.go`), (2) make import asynchronous (return 202, poll status) or raise timeout, (3) make AE-BR-09 ignore in-flight imports. Review by 12:00.
> **Next customer update:** 11:00.

### Workaround offered
Do not re-upload. Support reads the DB/ASP list, tells him what is accepted, and sends only missing invoices (a small file with only those numbers — numbers of *rejected* invoices can be reused per the lifecycle doc; numbers of accepted ones cannot). Smaller files (≈1 invoice-per-buyer-day) also finish < 60 s.

### Prevention ideas
Async import with progress; cache SMP lookups (done in patch); idempotency key on uploads (file hash + seller) so a re-upload is recognised; scope AE-BR-09 to committed invoices; alert on `invoicely_import_duration_seconds` p95 > 30 s; proxy timeout > import budget or no proxy timeout on upload.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:55 (alert 09:42, Priya's forward 09:50)
- **Sent to:** Priya Nair (#cs-escalations) and support-oncall
- **Reply:**

> Hi Priya — I own this. Yes: the alert and the 502s you're hearing about are almost certainly the same incident (API container restarting). Invoices sitting in "Validated" since ~09:30 are consistent with that too (queue lost on restart). I'm confirming the cause now. Customers can use **Retry stuck invoices** on the batch page for anything stuck in Validated; please ask them not to re-upload large files for now. I'll tell you before your afternoon call whether Wednesday is safe — update at **10:30**, then every 30 min. — Snehal Vharambale

### Priority and justification

- **Priority:** P1.
- **Why:** Production service restarting, intermittent 502 for all customers on upload, in-flight submissions lost; a go-live in 2 days.
- **SLA deadlines:** first response 10:12 (sent 09:55) / update every 30–60 min / resolution 13:42.

### Investigation log

| Time | What I checked / did | What I found |
|---|---|---|
| 09:55 | Alert rule `rules.yml` `invoicely-api-restarted` | Fires on `changes(process_start_time_seconds{job="api"}[15m]) > 0` → the *process* restarted (not just unhealthy). |
| 10:00 | `docker-compose.yml:60-61` | api has `mem_limit: 128m` and `memswap_limit: 128m` (no swap headroom), `restart: unless-stopped` → an OOM kill ⇒ silent restart loop. |
| 10:05 | Code: what grows in memory? | `importer.SourceCache` (`sources.go`) `Put` is called for **every upload** (`importer.go:95`) and **never evicts**. Each entry holds the raw .xlsx bytes, the full excelize `File`, all parsed rows and every mapped invoice. A 600-row file is far heavier in memory than on disk. `make load` replays Oasis + 240 peak uploads at 8 concurrent. |
| 10:15 | `submit.go` queue | Submission queue is an in-memory channel (cap 100 000). On restart it is empty; invoices already `Validated` are never re-enqueued → "stuck in Validated since 09:30". |
| 10:20 | 502s | `route.ts:56` returns 502 when the api is unreachable (connection refused during restart). Matches "mostly when they upload". |
| 10:25 | Dead ends | Panic/nil pointer: no obvious path (`g.Rows[0]` is safe, groups never empty). DB down: api would exit after 30 pings only at start-up. asp-mock memory is separate (64 m) — check but not the alerting service. |

### Root cause

- **What happened:** The api container is being killed and restarted (3× in 15 min). While it is down the portal returns 502; every restart drops the in-memory submission queue, leaving validated invoices unsent.
- **Where:** `services/api/internal/importer/sources.go` (unbounded cache), `importer.go:95`, `docker-compose.yml:60` (128 MB limit); queue loss in `submit.go` `Submitter.queue`.
- **Why:** The cache keeps every uploaded workbook for the life of the process, so heap rises with traffic until the cgroup limit kills the process; nothing re-submits after restart.
- **Evidence (to confirm on the running stack):**
  - `docker inspect -f '{{.State.OOMKilled}} {{.RestartCount}}' invoicely-lite-api-1` → expect `true N`.
  - Grafana "Container memory (working set)" – api line saw-tooths to ~128 MiB then drops; `go_memstats_heap_inuse_bytes{job="api"}` climbs between restarts.
  - `docker compose logs api | tail` – log stops abruptly with no panic; last line precedes a new `api starting`.
  - SQL: `SELECT status,count(*) FROM invoices WHERE updated_at > now()-interval '2 hours' GROUP BY 1;` → a pile of `Validated`.
- **Confidence:** Medium-high on OOM from unbounded cache (mechanism clear, numbers unverified); **if `OOMKilled=false`, pivot to the log tail before the restart** (panic / `os.Exit`).

### Customer resolution reply OR escalation note

- **Which one and why:** **Escalate (L3) — P1, page on-call now** (policy: page as soon as impact is confirmed), plus update to Priya.

> ### Escalation: API container restart loop (suspected OOM) — 502s and stuck Validated invoices
> **Priority:** P1 — all customers; Marina Bay go-live Wed 18 Nov.
> **Reported:** Grafana alert 09:42, Priya Nair 09:50.
> **Affected:** all uploads since ~09:30; unknown number of invoices in Validated.
> **Symptoms:** intermittent 502 on upload; invoices stuck in Validated.
> **Evidence:** alert expr `changes(process_start_time_seconds{job="api"}[15m])=3`; `mem_limit/memswap_limit 128m` (`docker-compose.yml:60`); unbounded `SourceCache`.
> **Repro:** `make load`, watch api memory.
> **Suspected cause:** OOM kill from unbounded `SourceCache`. Confidence medium-high.
> **Workaround:** customers click Retry stuck invoices; support triggers `POST /v1/imports/{id}/retry` for affected imports.
> **Ask:** (1) confirm OOMKilled; (2) deploy bounded cache (patch in repo) and raise `mem_limit` to ≥384 m; (3) re-enqueue Validated/Failed invoices at start-up; (4) soak test with `make load` before Wednesday. First answer in 30 min.
> **Next customer update:** 10:30.

> **Update to Priya (10:30 draft):** Wednesday readiness: *conditional*. I can say "yes" once (a) the fix is deployed, (b) `make load` runs clean twice with api memory flat and no restarts, and (c) Marina Bay's file size is checked. I'll confirm by 15:00, before your call; today I would tell their GM we are actively fixing a fault that does not lose data and that we'll confirm readiness this afternoon.

### Workaround offered
Retry stuck invoices (batch page / `POST /v1/imports/{id}/retry`); avoid parallel large uploads; ops can restart api and temporarily raise memory.

### Prevention ideas
Bound/evict source cache (patch); persist the queue (or re-enqueue Validated at boot); memory alert at 80% of limit before the restart alert; load test in CI; document the memory budget.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):** 10:45 (received 10:15)
- **Reply:**

> Dear Fatima,
>
> Thank you for the clear write-up and for attaching the file. You asked why some invoices were rejected with BR-CO-15 and others were not, and what you need to do. I'm Snehal Vharambale in Product Support and I'm looking at your file now.
>
> BR-CO-15 is a check that the invoice total with VAT equals the total without VAT plus the VAT. I'm going to compare the figures our portal generated against your file, line by line, rather than guess. Please don't change or re-send anything yet.
>
> I know your auditor needs a written explanation; I'll send a first one by **14:00** today.
>
> Regards, Snehal Vharambale

### Priority and justification

- **Priority:** P2.
- **Why:** Subset of invoices rejected (3 of 6 in the file); compliance/audit pressure raises urgency but invoice issuing for the rest is working.
- **SLA deadlines:** first response 11:15 (sent 10:45) / update 14:00 / resolution 18:15 (8 h of business hours).

### Investigation log

| Time | What I checked | Found |
|---|---|---|
| 10:50 | Parsed `samples/al-noor-fuel-nov-2026.xlsx` | 6 invoices; line prices have 3 decimals (e.g. 2.917), qty large. Template guide says up to 3 decimals is allowed. |
| 11:00 | ASP rule `validate()` in `asp-mock/main.go:~150-175` | BR-CO-15: `cents(TaxInclusive) == cents(TaxExclusive)+cents(Tax)`, strict equality in cents. |
| 11:10 | `pint.computeTotals` (`pint.go:157-186`) | `TaxExclusive = round2(Σ line net)` and `Tax = round2(Σ round2(line VAT))`, but **`TaxInclusive = round2(Σ qty × price × (1+rate))`** — computed from the *unrounded* quantity×price, not from the rounded line net. |
| 11:20 | Reproduced in Python (`evidence/analysis-output.txt`) | Invoices 0042, 0044, 0046 → total with VAT is 0.01 / 0.02 / 0.01 lower than net + VAT. 0041, 0043, 0045 match. This is the "some yes, some no" pattern. |
| 11:25 | Why not 0045? | Single-line invoice whose qty×price×1.05 rounds to the same cent. Whether the rounding gap shows depends on line count and the 3rd decimal of price, not on the customer's data being wrong. |

### Root cause

- **What happened:** Our portal calculated the invoice total with VAT inconsistently with the totals it sent beside it; the ASP correctly refused them.
- **Where:** `services/api/internal/pint/pint.go:159-185` (`gross += l.Quantity * l.UnitPrice * (1 + l.VATRate/100)`).
- **Why:** Rounding done at different stages: line amounts and VAT are rounded to 2 dp per line (as in the file), the gross is not. Multi-line invoices with 3-dp prices accumulate 0.01–0.02 drift.
- **Evidence:** table above; per-invoice figures: 0042 net 14307.08 + VAT 715.35 = **15022.43**, portal sent 15022.42; 0044 11254.84 + 562.75 = **11817.59** vs 11817.57; 0046 5343.80 + 267.20 = **5611.00** vs 5610.99. *On the stack:* SQL `SELECT invoice_number,total_excl_vat,total_vat,total_incl_vat,errors FROM invoices WHERE invoice_number LIKE 'ANF-2611-%';` and Loki `{service="asp-mock"} | json | msg=~".*rule.*"`.
- **Confidence:** High; the customer's file is correct.

### Customer resolution reply OR escalation note

- **Which one and why:** **Escalate to L3** (code defect, I cannot change totals from support) **and** update the customer.

> **Update to customer (14:00 draft):**
> Dear Fatima, thank you for your patience. The short answer: **the problem is on our side, not in your file.** Your quantities, prices and line amounts are correct. When the portal builds each invoice it calculates the total with VAT in a slightly different way from the other totals, which on some invoices produces a one-or-two fils difference. The network checks that these totals agree exactly, so it refused invoices ANF-2611-0042, -0044 and -0046; the other three had no difference, which is why they passed. 1) *Why these:* the difference appears on invoices with several lines and three-decimal fuel prices. 2) *What you need to do:* nothing in your file. 3) *When:* I've passed it to our engineers with an exact correction; I will confirm the date once agreed — next update by **16:00**. Once the fix is live I will ask you to re-send **only those three invoices** (their numbers are free to reuse, since they were rejected); sending the whole file again would fail for the three that were accepted. If your auditor would like it, I can provide a one-page note with the figures above.

> ### Escalation: BR-CO-15 on multi-line invoices — TaxInclusive computed from unrounded gross
> **Priority:** P2 (audit pressure). **Customer:** Al Noor Fuel & Lubricants LLC. **Reported:** Fatima Khoury, 10:15.
> **Affected:** ANF-2611-0042, -0044, -0046 (and any multi-line invoice with 3-dp prices, any customer).
> **Evidence:** payload excerpt `totals: {taxExclusiveAmount:14307.08,taxAmount:715.35,taxInclusiveAmount:15022.42}`; ASP 422 `BR-CO-15 … Got 15022.42, expected 15022.43`; code `pint.go:171,184`; Python reproduction attached.
> **Repro:** `make replay CUSTOMER=al-noor`.
> **Cause:** high confidence. **Fix (patch in repo):** `gross := round2(net) + round2(vat)`. Add a unit test over this file.
> **Ask:** review/release today; re-run replay and confirm 6/6 Accepted. **Next update to customer:** 16:00.

### Workaround offered
None the customer can do (their data is right). Interim: hold the three invoices; after fix re-upload only those.

### Prevention ideas
Compute totals in integer fils; unit tests for rounding with 3-dp prices; validate BR-CO-10/14/15 in the portal *before* submit so it reads "rejected by portal" with a clear explanation; add a regression test using this sample.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):** 11:50 (received 11:30)
- **Reply:**

> Hi Daniel,
>
> Thanks — useful detail, and thank you for holding the resend. **Please keep holding it**: resending now would likely create duplicates on our side. Short answer to your first question: I believe invoices are not lost but are **stuck before reaching the network** — that matches "Submitted" with an empty ASP reference. I'm confirming against the ASP's own record of what it received and will come back by **13:00** with the list of which of the 40 arrived and a safe plan for the rest. Sahara's payment-run deadline is noted. — Snehal Vharambale

### Priority and justification

- **Priority:** P2 (escalating to P1 on confirmation if another customer's >8-invoice batch is affected).
- **Why:** 32 of 40 invoices not delivered, payment run this week; but it is a subset of invoices, nothing is deleted, and we have a safe path.
- **SLA deadlines:** first response 12:30 (sent 11:50) / update 13:00 then every 2 h / resolution 19:30 (next business day if after 18:00).

### Investigation log

| Time | Checked | Found |
|---|---|---|
| 11:55 | `httpapi.createImport` (`httpapi.go:~130-165`) | Returns **201** as soon as the file is parsed/validated; submission is background → Daniel's 201 is expected, not a delivery receipt. |
| 12:00 | Sample: 60 rows / 40 invoices | Two buyers (hospital + one more). |
| 12:05 | ASP `limiter` `asp-mock/main.go:~32,76-90, 215-225` | Fair-use: burst 8, then 2 docs/s per **sender**; exceeding gives **HTTP 429 + Retry-After**; "the document has not been received". |
| 12:10 | `submit.go` original `switch resp.StatusCode` (`:133-155`) | Handles 200, 422 and 500/502/503/504 only. **429 matches no case** → invoice stays `Submitted` (set before sending, line ~104), no event, no retry. |
| 12:15 | `httpapi.retryInvoice` (`:~250-262`) | Retry only allowed for Validated/Failed → **409 on Submitted**, so neither the portal button nor the API can fix these. |
| 12:20 | Simulation of 40 sends with 4 workers vs bucket (`evidence/analysis-output.txt`) | **8 accepted, 32 throttled** → exactly the "a few arrived, rest didn't". |
| 12:25 | Dead end | Considered buyer SMP lookups failing: failure only logs a warning and wouldn't leave "Submitted". Considered loss in connector: portal got the file (201), so no. |

### Root cause

- **What happened:** Four workers push invoices to the ASP faster than the ASP's per-sender fair-use limit. The ASP answers 429 ("not received, send again later"). Our submitter doesn't recognise 429, so the invoice stays "Submitted" forever, with no ASP reference. ~8 of 40 got through.
- **Where:** `services/api/internal/submit/submit.go:133-155` (missing 429 / default case), `httpapi.go` retry guard (409 on Submitted).
- **Why:** The ASP integration contract (architecture doc) documents 429 + Retry-After; the client ignores it. Any seller with more than ~8 invoices in a burst is exposed.
- **Evidence (to confirm):** `curl "http://localhost:8090/v1/documents?sender=<Sahara peppol id>"` → ~8 docs. Loki `{service="asp-mock"} | json | msg="rate limit exceeded"` (≈32 lines, `retry_after_s`). `{service="api"} | json | status_code=429`. Grafana ASP responses panel: 429 series. SQL: `SELECT status,count(*),count(*) FILTER (WHERE asp_message_id='') FROM invoices WHERE import_id=<id> GROUP BY 1;` → `Submitted` ≈ 32 with empty reference.
- **Confidence:** High.

### Customer resolution reply OR escalation note

- **Which one and why:** **Escalate to L3** — needs a code fix and a data repair (Submitted→Failed for those without a message id), neither possible from support.

> ### Escalation: 429 from ASP leaves invoices in Submitted forever (data loss risk in perception)
> **Priority:** P2 → P1 if reproduced for other senders. **Customer:** Sahara Medical Supplies FZE via Bluepeak ERP. **Reported:** 11:30.
> **Affected:** import for `sahara-medical-batch-1116.xlsx`, 40 invoices SMS-1116-*, ~32 Submitted with no ASP ref.
> **Evidence:** as above (ASP list, 429 logs, SQL, `submit.go` switch).
> **Repro:** `make replay CUSTOMER=sahara`.
> **Cause:** high confidence. **Patch in repo:** `sendWithBackoff` honours Retry-After (≤8 retries) and a `default` case marks anything unexpected `Failed`.
> **Ask:** (1) deploy fix; (2) data repair: `UPDATE invoices SET status='Failed' WHERE import_id=… AND status='Submitted' AND asp_message_id='' AND updated_at < now()-interval '10 min'` (+ event row), then `POST /v1/imports/{id}/retry` — **only after the ASP list confirms which ones it already holds** to avoid duplicates at the hospital; (3) check other senders for the same pattern today. By 14:00.
> **Next customer update:** 13:00.

> **Update to Daniel (13:00 draft):** Hi Daniel, confirmed: nothing was lost. Of the 40 invoices, [N] were accepted by the network and [M] were throttled by the network's fair-use limit, which our portal did not handle — that is our fault, not your connector's. The 201 you received only confirms we received the file. **Your questions:** 1) the rest are still held in our system, not lost; 2) **please do not resend** — we'll resend them from our side after the fix and I'll send you the list of invoice numbers and their network references so Sahara can reconcile with the hospital. Next update by **15:00**.

### Workaround offered
None for the connector (do not resend). Support/engineering re-queue only the invoices the ASP list says are missing; for any future batch, feeding ≤8 invoices per few seconds avoids throttling.

### Prevention ideas
Backoff on 429 (patch); a `default` branch so unknown codes fail visibly; pace submissions with a per-sender limiter matching the ASP; alert on invoices in `Submitted` > 5 min; "stuck Submitted" retry in the portal; show "Submitted" age in UI; the 201 body should say "accepted for processing" not imply delivery.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):** 12:40 (received 12:10)
- **Reply:**

> Dear Hessa,
>
> I'm sorry for the frustration — I understand your customers are waiting. You're right that 13 November 2026 is a perfectly good date, and I can see why you're worried. Nothing is broken with your invoices; the portal needs dates written in one specific way so it never has to guess. I'll send you the exact steps (two minutes in Excel) within the hour, by **13:30**. — Snehal Vharambale

### Priority and justification

- **Priority:** P3 (P4-like "working as documented", but a small business is blocked until they edit the file — so P3).
- **Why:** One customer, one cause, a simple workaround.
- **SLA deadlines:** first response 16:10 (sent 12:40) / update daily / resolution 3 business days.

### Investigation log

| Time | Checked | Found |
|---|---|---|
| 12:45 | Opened `samples/ghaf-cafe-nov-2026.xlsx` | Dates in col D/E are **text** cells (`data_type=s`, format General): `'13/11/2026'`, `'13/12/2026'` — all 8 rows (`evidence/analysis-output.txt`). |
| 12:50 | `sheet.ParseDate` (`sheet.go:~165-181`) | Accepts `YYYY-MM-DD` text or Excel date serials only. `13/11/2026` matches neither → AE-BR-04, exactly the customer's message. |
| 12:55 | `docs/template-field-guide.md#dates` | Documented behaviour, with reason (dd/mm vs mm/dd ambiguity on a tax invoice). |

### Root cause
Working as designed. Cells hold text, not Excel dates, in `dd/mm/yyyy`; the portal deliberately refuses to guess the order. (If she'd entered real dates, `RawCellValue` gives the serial and it passes.)
- **Confidence:** High.

### Customer resolution reply OR escalation note
- **Which one and why:** Resolution reply — no escalation (product working as documented). The improvement (clearer error text) goes in as a *suggestion*.

> Dear Hessa, thank you for your patience — your invoices are fine, and your dates are right. The portal can't accept "13/11/2026" typed as text because the same text could mean 13 November or (in other countries) a different day, and on a tax invoice we must never guess. Here is how to fix it: 1) In Excel, select the two date columns (issue date and due date) → right-click → **Format Cells → Custom** and type `yyyy-mm-dd`. 2) Because your dates were typed as text, Excel won't change them yet: in an empty column enter `=TEXT(DATE(RIGHT(D2,4),MID(D2,4,2),LEFT(D2,2)),"yyyy-mm-dd")`, copy it down, then paste the results as values over the date columns (do the same for the due date). The result looks like `2026-11-13`. Then upload again — as these four invoices were all rejected before they were sent, the same invoice numbers can be used. I've attached a copy of your file with the dates corrected [if you prefer]. Please tell me if anything still looks wrong; I'll check in tomorrow morning.

### Workaround offered
The two-step Excel fix above, or let me send you a corrected file.

### Prevention ideas
Show a hint in the error text ("use 2026-11-13; see how to convert") and a link; accept unambiguous `dd/mm/yyyy` when day > 12 *or* ask the user to pick the format; ship the template with date cells pre-formatted.

---

## Overall

### Prioritisation
Acknowledgement order follows arrival (Inc 1 09:20 → Inc 2 09:55 → Inc 3 10:45 → Inc 4 11:50 → Inc 5 12:40). Working order: **2, 1, 4, 3, 5**. Inc 2 first because a restarting API makes all other evidence unreliable and is customer-wide with a go-live; Inc 1 next (P1, month-end) and I now think it shares a platform root with Inc 2 (large uploads). Inc 4 before Inc 3: 32 invoices undelivered and a payment run, versus 3 invoices with a known, fixed-in-code defect and no customer-side action. I considered raising Inc 4 to P1 once I realised the defect hits any sender with >8 invoices.

### What I would do differently
Run the stack first and capture screenshots/log lines (this write-up could not), and write unit tests for the rounding and the 429 path. Ask the customer for the first upload time in Inc 1 so I can build an exact timeline. Check the ASP's own list before telling anyone "nothing was lost".

### What I learned / what I would need to learn
PEPPOL 5-corner model, PINT AE and EN 16931 arithmetic rules (BR-CO-10/14/15), why 429 ≠ 5xx semantically (not received, retryable), Next.js route-handler timeouts, Go `context` lifetimes, cgroup memory limits vs Go heap. I learned them from the repo docs and code. Next: Grafana/Loki query practice, Go profiling (`pprof`) for memory, and the real PINT AE spec.

### Proposed code changes in Claude's working copy (not verified in the live working repository)
- `pint.go`: `TaxInclusive = round2(net) + round2(vat)` (Inc 3).
- `submit.go`: 429 handling with Retry-After backoff + `default` → Failed (Inc 4).
- `directory.go`: 10-min cache for participant lookups (Inc 1).
- `sources.go`: bounded source cache (10 entries) (Inc 2).
These are described in the pre-existing Claude-generated write-up. Their presence in the user's current local repository and their correctness have not yet been independently verified. Do not claim they are deployed or tested. Verify the diffs, compile, run unit tests, and replay the relevant customers before relying on them.

### Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| 2026-11-14 | Exact start/end times were not recorded; total active hours are unavailable. | Local environment setup, Portal check, initial API import-list check, investigation and documentation review |

### AI usage disclosure
This submission was drafted with **Claude (Anthropic, via Claude Code)**: it read the repo, parsed the sample workbooks, ran the Python reproductions in `evidence/analysis-output.txt` and wrote the patches and this document. It could **not** run Docker/Grafana/Postgres or compile the Go code. Findings that depend on the running system are marked *to confirm*; **you must verify them against the live stack, edit the replies in your own voice, and fill the name/time fields before submitting** — the assignment states the work must be your own.
