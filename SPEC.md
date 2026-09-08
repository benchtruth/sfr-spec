# SFR: Silent Failure Rate

**Version 1.0 (draft)**

A specification for measuring how often an automation platform accepts work and then fails to complete it without telling anyone.

> **Status: draft.** This document is being written. Nothing here is stable until a `v1.0` tag exists.
>
> **Reference implementation: planned, not yet published.** This specification constrains how a measurement is made and reported. It does not require any particular software, including ours.

---

## 7. Conformance

*(Written first, because it is the part of this document that does any work. The sections above it exist to make the levels below meaningful.)*

A conformance level is a property of **a published measurement**, not of an organisation, a product, or a company. "We are SFR-conformant" means nothing. "The figures in this report are SFR-conformant (L1) against SFR v1.0" is a claim a reader can check.

**Conformance is self-declared. There is no certifying body, no registry, and no audit.** We do not review claims and will not arbitrate disputes about them. The levels are useful only because every requirement below is checkable by a reader of the report itself: if a claim is false, the report will not contain what the level requires, and anyone can see that.

### The claim

A conformance claim must state the **level** and the **specification version**:

> SFR-conformant (L1), SFR v1.0

and must appear with, or link to, the measurement it describes.

**If a requirement is not met, do not claim the level.** Claim the level below it and state what is missing. A partial claim is worth more than an inflated one, and it is the only kind that survives a reader checking it.

### L1 — Reported

*You compute a silent failure rate from your own operational data and report it in a form that can be evaluated.*

L1 requires no test harness, no purchased subscriptions and no controlled experiment. Anyone already running automations can reach it. **Its bar is not effort, it is discipline:** most published silent-failure figures today fail L1 on the reporting requirements alone, because they arrive without a denominator.

1. **Per-event identity.** Every source event carries an identifier that is preserved to the destination, so that a specific event can be matched to a specific outcome. Counting totals is not enough: totals can match while the wrong records are missing.
2. **Destination-side verification.** Whether an event succeeded is determined by checking the destination, not by reading the platform's own run history. A run history cannot report a run it never created, and it records transport success, not the business outcome.
3. **The send-outcome exclusion.** Events that were **refused at the moment of sending** (a non-2xx response, a timeout, or a connection failure at the point of delivery to the platform) are excluded from **both the numerator and the denominator**, and their count is reported separately. See §1 for why. This is the load-bearing rule of the whole metric.
4. **Denominator stated.** The report states how many runs were accepted and expected to produce output. A rate without a denominator is not a measurement.
5. **Interval stated, and never a bare zero.** The report gives a Wilson score interval at 95% confidence alongside the point estimate. **A rate of 0% may not be published without its upper bound.** Zero failures in 40 runs and zero failures in 4,000 runs are different claims, and reporting both as "0%" misleads by omission.
6. **As-of date and window stated.** The report gives the date the figure was computed and the period it covers. A silent failure rate with no date attached is not conformant: platforms change, and an undated rate silently becomes a claim about a system that no longer exists.
7. **Workload described.** The report says what was being measured: what the workflow does, how many steps it has, what triggers it, and how often it ran. An SFR is a property of a workload on a platform, never of a platform alone.

### L2 — Measured

*L1, plus a controlled measurement rather than an observation of your own traffic.*

L2 is what an independent benchmark requires. It is reachable by anyone willing to run a controller and a destination for a few weeks.

8. **Independent trigger.** Events are fired by a controller you operate, on a schedule you set. Organic traffic cannot support L2, because the volume, timing and content are not yours to state.
9. **Both endpoints under your control.** The destination the workflow writes to is yours. This is what separates a platform failure from a third-party outage: if the destination is someone else's API, you cannot tell which one failed, and the resulting number measures the pair, not the platform.
10. **Reference workloads.** The measurement uses the workloads defined in §3, or a documented subset. Omissions are named. Substitutions are described in enough detail that a reader can judge comparability.
11. **Method published.** Enough detail that a competent reader could re-run the measurement without asking you anything.
12. **Per-run data available.** The record of individual runs is published, or provided on request. Aggregates alone cannot be audited.

### L3 — Stress

*L2, plus the three edge behaviours that ordinary operation never exercises.*

Normal-operation measurement answers "does it drop things when nothing is wrong". L3 asks what happens at the boundaries, which is where the failures that hurt actually live. Each probe must be run and its result published, **including when the result is that nothing interesting happened.**

13. **Destination outage.** Make your own destination unavailable for a stated window and record, per platform: whether it retries at all, how many times, with what backoff, whether pending retries are visible anywhere in the platform's interface, whether anything notifies you, and whether recovery produces duplicates.
14. **Success-wrapped failure.** Have your destination answer with a genuine 2xx status carrying a failure in the body, and record what the platform writes into its run history, what it bills, and what it takes to detect it.
15. **Sustained load.** Hold a stated multiple of the workload's normal rate for a stated duration, and report delivery, latency drift and refusals across the window. State the absolute rate, not only the multiple: a large multiple of a small number is still a small number.

### On claiming a level you do not hold

The failure mode this section is designed against is a report that reads as rigorous while omitting the one thing that would make its number checkable. Every requirement above is a thing a reader can look for and fail to find.

If you meet L1 and want L2, run the measurement. Do not claim L2 because your data is large.

---

## 1. Definitions and boundaries

### Silent failure

**A silent failure is a unit of work that a platform accepted, told you nothing was wrong about, and did not complete.**

Three things must all be true:

- **Accepted.** The platform took the event and produced a success response to whoever sent it.
- **Not completed.** The expected effect never appeared at the destination.
- **Not surfaced.** Nothing in the platform's normal reporting told the operator. No error, no failed-run status, no notification.

The third condition is what makes it silent, and it is what makes the class worth its own metric. A failure that announces itself is an operational event: you see it, you retry it, you move on. A failure that does not announce itself is a data-integrity event that surfaces weeks later as a customer asking where their record went.

### Silent Failure Rate (SFR)

```
        missed + partial
SFR = ──────────────────────────────────────
      runs accepted and expected to produce output
```

- **missed**: accepted, nothing arrived at the destination.
- **partial**: accepted, some but not all of the expected effects arrived (a two-branch workflow that delivered one branch).
- The denominator counts runs the platform **accepted** and that were **expected to produce output**. A run correctly stopped by a filter is not a failure and is not in the denominator; it is reported separately (see below).

Report with a Wilson score interval at 95% confidence. See §5.

### Rejected at send, and why it is excluded

**A rejection at send is not a silent failure and must not be counted as one.**

When a platform refuses an event at the moment of delivery, with a non-2xx status, a timeout, or a dropped connection, **the sender knows immediately.** It can log it, alert on it, and retry it. No run was created. Nothing is hidden.

This is the opposite property from the one the metric exists to measure. Mixing the two produces a number that means nothing: it would combine "the platform told you and you can retry" with "the platform did not tell you and the data is gone", and a reader could not tell which had happened.

**Rejections are therefore excluded from both the numerator and the denominator**, and reported as a separate count. They are a real and useful signal, an availability signal, and they belong next to the SFR rather than inside it.

> **Worked example.** In a five-hour sustained-load run of 2,880 events against a single platform, 4 events were refused at send and 0 accepted events failed to arrive. The conformant report is: **SFR 0 of 2,876**, with 4 rejections at send reported alongside. Reporting "4 failures in 2,880" would be wrong in both directions: it would inflate the failure rate with events the sender saw immediately, and it would hide the fact that nothing accepted was lost.

### Success-wrapped failure

An event where the destination returned a successful HTTP status carrying a failure in its body, for example `200 OK` with `{"ok": false}`.

Platforms evaluate step success on the transport status by design, so these are recorded as successful runs. This is not a platform defect: a platform cannot know what failure looks like inside an arbitrary API's response schema without being told.

**Whether these count toward SFR depends on the workload definition, and the report must say which convention it used.** If the reference workload defines success as "the destination recorded the write", a success-wrapped failure is a silent failure. If it defines success as "the destination answered 2xx", it is not. Both are defensible; leaving it unstated is not.

### What SFR is not

- **It is not uptime.** Uptime asks whether the platform answered. SFR asks whether your runs completed. A platform can report 100% uptime while a workload loses every event, and we have measured exactly that at a free-tier quota boundary.
- **It is not the platform's error rate.** An error rate counts failures the platform knows about. SFR counts the ones it does not.
- **It is not a property of a platform.** It is a property of a **workload on a platform, over a window**. Quoting an SFR without its workload, denominator and dates is quoting an artefact, not a measurement.

---

*Sections 2 to 6 are in progress.*
