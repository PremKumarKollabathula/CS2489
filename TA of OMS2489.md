# Technical Analysis: Order Taken Date Resolution in fnc_invoice_invoiced_details (CSS447B03F)

**Related Incident:** EOM429 / OMS2489 - Partial invoice information missing in EPIC
**Programs:** `EOM429\CS447B03F.sql` (`fnc_invoice_invoiced_details`), called from `EOM429\CS447A.sqlrpgle`
**Revisions covered:** PK-D (added 3-tier fallback), PK-E (removed tier 3), PK-H (added then removed EZVIEWJRN caller step),
PK-F/PK-I (re-instated tier 3, bounded to a 3 month `FROMTIME`/`TOTIME` window, after confirming the hosting job is a
genuine one-job-per-interactive-session process with no job pooling/reuse)

## 1. Background

`fnc_invoice_invoiced_details` filters invoices returned to EPIC using:

```sql
where INIVDT >= v_tkndat_num
```

`v_tkndat_num` is the order's "taken date" - the date the order was placed under its
*current* warehouse/ship-to combination. It exists to exclude stale invoice history
that belongs to a prior warehouse/ship-to, for orders that went through a
`WHSESWITCH`/`SHIPSWITCH` event.

The taken date was originally resolved from `ORDAUDH` only. The EOM429 incident
occurred because `ORDAUDH` does not retain a row for orders that have been fully
invoiced and cleared - for those orders the lookup found nothing, and partial
invoice information was dropped entirely from the EPIC response.

## 2. Fallback chain evaluated

Three tiers were considered, in resolution order:

1. **ORDAUDH** - existing lookup, current warehouse/ship-to audit history.
2. **EXTORD** - live order file; still holds `TKNDAT` for orders that are open or
   only cleared from `CODATAN` (partial invoice case).
3. **QTEMP.EXTORDJ** - a journal-extracted snapshot of `EXTORD`, built via the
   `EZVIEWJRN` command, intended to recover `TKNDAT` for orders fully invoiced and
   cleared from `COMAST`/`CODATAN`/`EXTORD` (no longer present in EXTORD itself).

## 3. Decision history: Tier 3 (QTEMP.EXTORDJ / EZVIEWJRN) removed, then re-instated

Tier 3 was implemented (revision `PK-D`/`PK-H`) and then **removed** (revision `PK-E`
in the function, `PK-H` rollback in the caller) for the following reasons:

- **Cost on every call, not just when needed.** `CSS447B03F` is a read-only SQL
  scalar function and cannot itself run `EZVIEWJRN` (a CL command via
  `QSYS2.QCMDEXC`), since SQL functions cannot perform side-effecting calls. The
  extraction had to be moved to the caller, `CS447A.sqlrpgle`, which then ran
  `DROP TABLE QTEMP.EXTORDJ` -> `EZVIEWJRN` -> query -> `DROP TABLE QTEMP.EXTORDJ`
  on **every single API invocation**, regardless of whether any order in that
  request actually needed the tier-3 fallback. This adds journal-extraction
  overhead and two extra SQL/CL round-trips to the critical path of every request.
- **QTEMP is job-scoped, not request-scoped.** If the job hosting `CS447A` is ever
  reused to serve overlapping or concurrent requests (rather than strictly one
  request per job, run to completion), the drop/create/query sequence for one
  request can race against another request's drop/create/query sequence against
  the same `QTEMP.EXTORDJ` file, risking intermittent SQL errors or incorrect
  results for unrelated orders. This could not be verified as safe for the current
  job/execution model.
- **Failure is silent by design.** The `EZVIEWJRN` call was wrapped in
  `monitor`/`on-error` so a failure there (journal not active, receiver issue,
  authority problem) would not fail the API request - but that also means a real
  infrastructure problem would go unnoticed, silently falling through to the same
  default-0 behavior described below.

Given the added latency/operational cost and residual concurrency risk of tier 3,
the decision at that time was to **stop at tier 2 (EXTORD)** and accept the
default-0 fallback documented below for the remaining case, rather than adding a
per-request journal extraction to every call of this API.

### 3.1 Re-instatement (revisions PK-F / PK-I)

Re-instatement spans both programs:
- `EOM429\CS447B03F.sql` - revision **PK-F**: restores tier 3 (`QTEMP.EXTORDJ` lookup)
  in `fnc_invoice_invoiced_details` between the `EXTORD` fallback and the default-0
  fallback.
- `EOM429\CS447A.sqlrpgle` - revision **PK-I**: restores the `EZVIEWJRN` extraction in
  `GenerateOrderJson` (pre-cleanup drop of `QTEMP.EXTORDJ`, bounded `FROMTIME`/`TOTIME`
  build, `EZVIEWJRN` call via `QSYS2.QCMDEXC` wrapped in `monitor`/`on-error`, and
  post-use cleanup drop on both the error and success paths), so `QTEMP.EXTORDJ`
  exists for `CS447B03F` to query.

Both open concerns from Section 3 above were resolved before re-instating:

- **Concurrency risk closed.** Confirmed that `CS447A` runs as a genuine
  interactive job - one workstation/device session per job, never reused or
  pooled across overlapping requests. This is the same 1:1 job-to-session model
  IBM i interactive jobs always use, so the `QTEMP.EXTORDJ` drop/create/query
  sequence for one request can never race against another request's sequence.
- **Per-call cost bounded, not eliminated.** `EZVIEWJRN` is still run on every
  call to `GenerateOrderJson` (there was no reliable way to detect in advance,
  without cost, whether the current order set would need tier 3), but the
  extraction is now bounded with `FROMTIME`/`TOTIME` to the last 3 months up to
  the current timestamp, instead of a full/unbounded journal replay. This caps
  the worst-case extraction volume per call to 3 months of `EXTORD` journal
  activity rather than the file's entire journal history.
- Failure of the `EZVIEWJRN` call remains wrapped in `monitor`/`on-error` (see
  Section 3 above) - if it fails, tier 3 simply yields nothing and the chain
  falls through to tier 4 (default 0), same as before re-instatement.

## 4. Current fallback chain (post PK-F / PK-I)

```
1. ORDAUDH                             -> v_tkndat_num = AHOFFD/AHORTD
2. EXTORD (if ORDAUDH not found)       -> v_tkndat_num = TKNDAT
3. QTEMP.EXTORDJ (if EXTORD not found) -> v_tkndat_num = TKNDAT
                                           (EZVIEWJRN extract of EXTORD journal,
                                           last 3 months, run by CS447A before
                                           calling this function)
4. default 0 (if EXTORDJ not found)    -> v_tkndat_num = 0
```

## 5. Impact analysis of the default-0 fallback

When `ORDAUDH`, `EXTORD`, and `QTEMP.EXTORDJ` all miss - which now only happens for
orders that have been **fully invoiced and cleared** from `COMAST`/`CODATAN`/`EXTORD`
**and** whose taken date also falls outside the 3 month `EZVIEWJRN` extraction window
(or the `EZVIEWJRN` extraction itself failed) - `v_tkndat_num` defaults to `0`.

Because the invoice filter is `INIVDT >= v_tkndat_num`, and `INIVDT` (invoice date)
is always a positive value for any real invoice, a taken date of `0` makes this
condition **true for every invoice ever issued against the order**, with no lower
bound.

**Consequence:** For an order that went through one or more warehouse/ship-to
switches (`WHSESWITCH`/`SHIPSWITCH`) during its life, and whose taken date is now
unrecoverable (fully invoiced/cleared case), invoices issued **before the very
first switch** - i.e. under a prior, now-stale warehouse or ship-to combination -
are **no longer excluded** by the date guard. Those old invoices, which the taken
date filter was originally designed to hide, **may now appear in the EPIC
response** alongside the current, correct invoices for that order.

**Scope of exposure:** This only affects orders where:
- the order has been fully invoiced and cleared (no `ORDAUDH` row, no `EXTORD` row), **and**
- the recoverable taken date also falls outside the 3 month `EZVIEWJRN` window used to
  build `QTEMP.EXTORDJ` (or the `EZVIEWJRN` extraction failed), **and**
- the order underwent at least one warehouse or ship-to switch during its life.

Orders that were never switched are unaffected, since without a switch there is no
"prior, stale" invoice history to leak - all of the order's invoices legitimately
belong to the same warehouse/ship-to.

**Trade-off accepted:** This is a deliberate choice to favor **surfacing
potentially stale invoice history** over **silently dropping partial invoice
information**, which was the original EOM429/OMS2489 symptom. Returning some extra,
old invoice rows for a small subset of switched-and-cleared orders is considered
less harmful than omitting invoice data outright for those same orders.

## 6. Possible future mitigation (not implemented)

If the residual stale-invoice exposure described in Section 5 (orders whose taken
date falls outside the 3 month `EZVIEWJRN` window) proves unacceptable in practice,
options to revisit include:
- Widening the `EZVIEWJRN` `FROMTIME` window beyond 3 months, trading additional
  per-call extraction cost for a smaller residual exposure window.
- Gating the `EZVIEWJRN` call so it only runs when the current request's order set
  actually contains at least one order missing from both `ORDAUDH` and `EXTORD`
  (avoiding the per-call cost for requests that never need tier 3).
- Persisting the order's taken date on a permanent, durable field (e.g. a new
  column populated at the time of switch, rather than reconstructed after the
  fact) so it survives full invoicing/clearing without requiring journal
  recovery at read time.
