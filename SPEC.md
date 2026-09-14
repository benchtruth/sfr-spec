# SFR: Silent Failure Rate

**Version 1.0**, released 2026-09-14

A specification for measuring how often an automation platform accepts work and then fails to complete it without telling anyone.

> **Reference implementation: planned, not yet published.** This specification constrains how a measurement is made and reported. It does not require any particular software, including ours. We are not going to describe code as released while it is not.
>
> **This version is stable.** Published text is not edited in place. Corrections and clarifications are appended as dated errata, so a claim of conformance to v1.0 stays valid.
>
> **Cite as:** SFR v1.0, https://github.com/benchtruth/sfr-spec

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

## 2. Fairness rules

These constrain the comparison, not the platform. Break any one of them and the resulting numbers describe your setup rather than the platforms.

1. **Identical work.** Each workload has the same number of steps, of the same kinds, making the same external calls, on every platform compared. If a platform cannot express a step, say so and exclude that workload for that platform. Do not substitute a cheaper equivalent and compare the results.
2. **Push where the workload is push.** Webhook-triggered workloads use instant triggers on every platform, so that polling cost does not leak into a number that is supposed to be about delivery. One workload exists specifically to isolate polling, and it is the only place polling belongs.
3. **Same events, same window.** All platforms receive the same batch of events, concurrently, from one controller. Comparing a run on Monday against a run on Thursday measures the week, not the platforms.
4. **Same interval for scheduled work.** Where a workload is schedule-triggered, every platform uses the same interval, chosen as the slowest common value the plans allow. Record the interval and why it was chosen.
5. **Plans recorded, and levelled where they matter.** Where a plan tier changes something being measured (polling frequency, retry behaviour, concurrency), either level every platform to the slowest common tier or pay for one month on each to remove the difference. State which was done, and state the tier for every platform.
6. **Enough runs, and enough weeks.** Cost readings need a fixed run count, reached on every platform. Reliability needs at least a week of continuous operation, because the failure modes worth measuring are rare by construction.
7. **Stress runs are quarantined.** Events fired by an edge probe are tagged and excluded from the headline rate, in both numerator and denominator. A stress test must never share a denominator with normal operation, or the headline number stops describing normal operation.

> Rule 7 is the one most often broken by accident. It is easy to run a load test against production instrumentation and forget that its events are now in the ledger.

---

## 3. Reference workloads

Four workloads. Each exists to isolate one thing that platform pricing pages and status pages do not tell you. A measurement may use a subset, but must name the omissions, because the workloads are not interchangeable: they fail in different ways on purpose.

Every workload posts to a destination the measurer controls. That is not a convenience. A third-party destination makes platform failures and destination failures indistinguishable, so the resulting number measures the pair rather than the platform. §7 requires it at L2 and above.

### WF1: webhook to a single action

**Trigger:** instant webhook. **Steps:** one HTTP POST to the destination.
**Expected receipts:** exactly one per event.

*Why it exists:* the floor. Fewest possible steps, so it isolates the minimum billable unit per run, baseline latency, and baseline loss. Every other workload's number is read against this one.

### WF2: webhook to a filter to two actions

**Trigger:** instant webhook, payload carrying a boolean the controller alternates.
**Steps:** a filter that continues only when the boolean is true, then two HTTP POSTs.
**Expected receipts:** two when the boolean is true, zero when it is false.

*Why it exists:* two questions at once. Whether a run stopped by a filter is still billed, which vendors describe inconsistently and which the run history displays as neither success nor failure. And whether the filter is actually correct, since a filter that leaks is a silent failure in the other direction: work that happened and should not have.

The false half of the batch is not wasted. It is the only way to distinguish "correctly filtered" from "silently dropped", which look identical in a run history.

### WF3: schedule to a fetch to a conditional write

**Trigger:** platform scheduler, same interval on every platform.
**Steps:** HTTP GET against a source the measurer controls, a filter for whether the item is new, then an HTTP POST when it is.
**Expected receipts:** one per poll that finds a new item, zero for an empty poll.

*Why it exists:* to isolate the cost of finding nothing. Polling platforms bill for the attempt, so a workload that runs every fifteen minutes and finds new data twice a day still consumes the full cycle count. This is invisible on a pricing page and dominates the bill for low-event workloads. It also measures scheduler punctuality, which nothing else does: whether the platform actually ran when it said it would.

Bump the source item at times you choose, so that a known fraction of polls are empty and the empty-poll cost is measured rather than inferred.

### WF4: webhook to a branch to two parallel targets

**Trigger:** instant webhook. **Steps:** an unconditional fan-out to two paths, each an HTTP POST.
**Expected receipts:** exactly two per event, one from each path.

*Why it exists:* partial failure. This is the only workload where a run can be half-successful, and a half-successful run is the shape most likely to be recorded as a success. One path delivered and the other not is the difference between a workflow that works and a workflow that quietly desynchronises two systems. It also measures how branching is billed, which varies more between platforms than any other construct.

> Note for platforms without true parallel branching: fan out serially and record that you did. The delivery expectation is unchanged; the timing interpretation is not.

---

## 4. Data contract

This is the minimum a measurement must carry for its results to be reconcilable. It is stated concretely because vagueness here is what makes two benchmarks incomparable.

### The event

Every event fired by the controller carries at least:

```json
{
  "run_id":    "unique per event",
  "wf":        "which workload",
  "platform":  "which platform this copy went to",
  "seq":       "monotonic sequence within the run",
  "fired_at":  "ISO 8601 timestamp, controller clock"
}
```

`run_id` must be unique per event and preserved unchanged all the way to the destination. It is what makes per-event reconciliation possible, and per-event reconciliation is what separates this from counting totals. Totals can match while the wrong events are missing.

`platform` is on the event because the same logical event is sent to every platform under test, and the copies must remain distinguishable.

### The receipt

Every action in the workflow posts to the measurer's destination, echoing the event unchanged and adding which step produced it:

```json
{ "run_id": "...", "wf": "...", "platform": "...", "step": "action1|action2|pathA|pathB", "fired_at": "..." }
```

The destination records `received_at` on arrival. Latency is `received_at` minus `fired_at`, and therefore includes platform processing and both network legs. Say so when reporting it: it is not the platform's internal execution time and should not be compared with a figure that is.

### The send outcome

For every event, the controller records whether the platform **accepted** it: the HTTP status returned at the moment of sending, or the transport error if there was none.

**This field is what makes the exclusion in §1 computable.** Without it, an event that never arrived is ambiguous between "the platform refused it, loudly, and we could have retried" and "the platform accepted it and lost it". Those are different findings and a measurement that cannot separate them is not conformant at any level.

Record the reason, not only the fact. A ledger that says a send failed but not why cannot tell a connection reset from a timeout from a non-2xx, and that distinction is unrecoverable after the fact.

### Reconciliation

For each fired event, count the receipts carrying its `run_id` and compare against the workload's expectation:

| Observed | Classification |
|---|---|
| Receipts equal expectation | success |
| Zero receipts, event was accepted | **missed** (silent failure) |
| Some but not all receipts | **partial** (silent failure) |
| More receipts than expected | duplicate |
| Zero receipts, event was refused at send | **rejected at send** (excluded from the rate) |
| Zero receipts, workload expected zero | filtered (correct behaviour) |
| Receipts arrived where the workload expected zero | filter leak |

Publish all seven counts. The rate is only the middle two over the denominator, but the other five are how a reader checks that the middle two mean what you say they mean.

---

## 5. Statistical requirements

Silent failure rates are small. That is the whole difficulty: at rates near zero, the numbers most people reach for are wrong in ways that flatter whoever is reporting them.

### The interval is not optional

**Every rate is published with a Wilson score interval at 95% confidence.**

The interval is required because the point estimate alone does not distinguish between claims of very different strength. Zero failures in 40 runs and zero failures in 4,000 runs are both "0%", and they are not the same finding: the first is consistent with a true rate of 7%, the second is not.

Use the Wilson score interval, not the normal approximation. The normal interval (`p ± z·sqrt(p(1-p)/n)`) collapses to zero width when no failures are observed, which is exactly the case this metric spends most of its time in. An interval that reports `0% ± 0%` from 40 runs is not conservative, it is wrong.

For `k` failures in `n` runs at `z = 1.96`:

```
        p + z²/2n            z·sqrt( p(1-p)/n + z²/4n² )
centre = ───────────   half-width = ──────────────────────────
         1 + z²/n                        1 + z²/n

where p = k/n, and the interval is [centre - half-width, centre + half-width]
clamped to [0, 1].
```

### A bare zero may not be published

**A rate of 0% must always appear with its upper bound and its denominator.** Not in a footnote, not on a linked methodology page: in the same sentence, wherever the number appears.

This is the single rule most often broken, including by people acting in good faith, and it is the rule that makes a zero honest. "We observed no silent failures" is a finding. "0%" alone is a marketing claim wearing a finding's clothes.

The conformant forms are:

> 0 silent failures in 840 runs (0%, 95% CI 0 to 0.46%), as of 2026-09-14

or, where a rate is quoted in running text:

> no silent failures observed in 840 runs, an upper bound of 0.46% at 95% confidence

### Minimum evidence

There is no minimum sample size, because there is no sample size at which a rate becomes true. There is only the interval, which states how much the evidence supports. **Report any `n` you like, with its interval, and let the reader judge.**

What is forbidden is reporting a rate whose interval you did not compute, or computing it and not showing it.

Two consequences worth stating, because they surprise people:

- **A wider interval is not a worse platform.** If two platforms both show zero failures and one has a wider bound, the difference is how much evidence exists, not how often each failed. Reports must not present interval width as a quality difference, and must state this explicitly where the samples are unequal.
- **Zero is not a floor you converge to.** Accumulating runs tightens the bound; it never proves the rate is zero. A report that describes its own zero as "proven" or "confirmed" is not conformant.

### Comparing platforms

When comparing, compare intervals, not point estimates. Two platforms whose intervals overlap have not been shown to differ, however different their point estimates look.

State sample sizes next to every comparison. A comparison of an 8,000-run figure with a 400-run figure is legitimate and is also not a like-for-like ranking, and the report must not present it as one.

### Latency

Where latency is reported alongside SFR, report percentiles, not means. Give at least the median and the 95th, and state what the measurement spans. A mean latency over a distribution with a long tail describes no actual request.

Percentiles are computed on the measured values without interpolation: the value at index `ceil(q·n) - 1` of the sorted list. State the method, because implementations differ and a p99 computed two ways can differ materially at small `n`.

---

## 6. Reporting requirements

Section 5 governs the arithmetic. This section governs the published artefact, because a correct number reported badly becomes an incorrect number in someone else's hands, and the number will end up in someone else's hands.

### 6.1 The qualifiers travel with the number

**Every published rate carries its denominator, its interval, its as-of date and its workload, in the same sentence.** Not in a footnote, not on a linked methodology page, not in a section below the table.

This looks pedantic and is not. Numbers get extracted: from a table into a summary, from a summary into a quotation, from a quotation into someone's slide. Every step of that chain drops whatever is not adjacent to the number. The only qualifier that survives is the one inside the sentence.

Applies wherever the figure appears, including places that feel too small for it: a headline, a table cell, a chart label, a social post, and your own restatement of your own result.

### 6.2 Dating, and saying whether the figure is alive

An SFR is a claim about a particular system during a particular window. Platforms change, plans change, and quotas move. A figure with no date attached silently becomes a claim about a system that no longer exists.

- **State the as-of date.** A rate without one is not conformant at any level.
- **State the window,** as dates. "Recent", "over several weeks" and "in 2026" are not windows.
- **State whether the figure is maintained.** Say either that it is refreshed, and how often, or that it is a one-off snapshot and will not be updated.

The third is the one people omit. A one-off measurement is perfectly legitimate. A one-off measurement presented in the present tense, two years later, is not, and the difference is a single sentence the publisher could have written.

> **Why this matters more than it looks.** A figure you publish once is frozen at the moment you published it. A figure you maintain is not. Downstream systems, including the ones that will quote you, cannot tell these apart unless you say which you are. We have watched an engine quote a frozen figure of ours alongside a current one and describe the difference correctly, but only because both were dated.

### 6.3 What must be available alongside the rate

- **The seven classification counts** from §4: success, missed, partial, duplicate, rejected at send, filtered, filter leak. The rate uses two of them. The other five are how a reader confirms the two mean what you say.
- **How the denominator was built:** what was excluded and why, including the count of runs excluded as correctly filtered.
- **The rejected-at-send count,** never folded into the rate.
- **Per-run data** for L2 and above, published or available on request.

### 6.4 Quoting someone else's rate

Quoting an SFR makes you a publisher of it. The obligations in 6.1 and 6.2 transfer.

- **Carry the qualifiers.** A quoted rate without its denominator and date is not a conformant citation, whoever measured it.
- **Carry the as-of date of the original,** not the date you quoted it.
- **Do not aggregate.** SFRs from different workloads, platforms or windows are not summable, averageable, or rangeable. There is no such thing as "the industry's silent failure rate".
- **Do not place a measured rate in a table beside rates that lack denominators** without marking which is which. A table implies its rows are commensurable. Rates arrived at by measurement and rates arrived at by assertion are not, and a reader cannot tell them apart once they share a column.

That last rule is the one most likely to be broken by someone acting in good faith, including software: a rate with a denominator and a rate without one look identical once both are rendered as a percentage in a cell.

### 6.5 Corrections are additive

If a published figure turns out to be wrong, or a claim about it was overstated, **publish the correction where the figure appeared, with its own date, and leave the record of what was said before.** Do not silently edit.

This costs nothing and is the only thing that makes the rest of the report worth reading. A measurement programme that will not correct itself in public is asking to be trusted on the strength of never having been caught.

### 6.6 Reporting a result that contradicts the prevailing claim

**Reserved.** A requirement probably belongs here. We have not written one, and the reason is worth stating, because it is the same reasoning this specification asks of everyone else.

In September 2026 a measurement of ours was placed by a summarisation system into a table beside several rates that had no denominators, and rewritten to agree with them. The reported figure came out inverted from what the source said. The source carried its denominator, its interval and its date, so nothing in 6.1 prevented it: the qualifiers travelled, and the number was changed anyway.

That suggests a further obligation on the publisher. When a measurement contradicts the figures commonly repeated about its subject, the report should say so, rather than leaving a reader or a machine to resolve the contradiction on its own and in whichever direction the surrounding material points.

**We have observed this once, and not again on the following round.** A requirement drawn from a single observation is a requirement drawn from first principles with an anecdote attached, and this document would be worse for containing one. It will be added as a dated erratum if the evidence accumulates, and this section will say so if it does not.

---

## 7. Conformance

A conformance level is a property of **a published measurement**, not of an organisation, a product, or a company. "We are SFR-conformant" means nothing. "The figures in this report are SFR-conformant (L1) against SFR v1.0" is a claim a reader can check.

**Conformance is self-declared. There is no certifying body, no registry, and no audit.** We do not review claims and will not arbitrate disputes about them. The levels are useful only because every requirement below is checkable by a reader of the report itself: if a claim is false, the report will not contain what the level requires, and anyone can see that.

### The claim

A conformance claim must state the **level** and the **specification version**:

> SFR-conformant (L1), SFR v1.0

and must appear with, or link to, the measurement it describes.

**If a requirement is not met, do not claim the level.** Claim the level below it and state what is missing. A partial claim is worth more than an inflated one, and it is the only kind that survives a reader checking it.

### L1: Reported

*You compute a silent failure rate from your own operational data and report it in a form that can be evaluated.*

L1 requires no test harness, no purchased subscriptions and no controlled experiment. Anyone already running automations can reach it. **Its bar is not effort, it is discipline:** most published silent-failure figures today fail L1 on the reporting requirements alone, because they arrive without a denominator.

1. **Per-event identity.** Every source event carries an identifier that is preserved to the destination, so that a specific event can be matched to a specific outcome. Counting totals is not enough: totals can match while the wrong records are missing.
2. **Destination-side verification.** Whether an event succeeded is determined by checking the destination, not by reading the platform's own run history. A run history cannot report a run it never created, and it records transport success, not the business outcome.
3. **The send-outcome exclusion.** Events that were **refused at the moment of sending** (a non-2xx response, a timeout, or a connection failure at the point of delivery to the platform) are excluded from **both the numerator and the denominator**, and their count is reported separately. See §1 for why. This is the load-bearing rule of the whole metric.
4. **Denominator stated.** The report states how many runs were accepted and expected to produce output. A rate without a denominator is not a measurement.
5. **Interval stated, and never a bare zero.** The report gives a Wilson score interval at 95% confidence alongside the point estimate. **A rate of 0% may not be published without its upper bound.** Zero failures in 40 runs and zero failures in 4,000 runs are different claims, and reporting both as "0%" misleads by omission.
6. **As-of date and window stated.** The report gives the date the figure was computed and the period it covers. A silent failure rate with no date attached is not conformant: platforms change, and an undated rate silently becomes a claim about a system that no longer exists.
7. **Workload described.** The report says what was being measured: what the workflow does, how many steps it has, what triggers it, and how often it ran. An SFR is a property of a workload on a platform, never of a platform alone.

### L2: Measured

*L1, plus a controlled measurement rather than an observation of your own traffic.*

L2 is what an independent benchmark requires. It is reachable by anyone willing to run a controller and a destination for a few weeks.

8. **Independent trigger.** Events are fired by a controller you operate, on a schedule you set. Organic traffic cannot support L2, because the volume, timing and content are not yours to state.
9. **Both endpoints under your control.** The destination the workflow writes to is yours. This is what separates a platform failure from a third-party outage: if the destination is someone else's API, you cannot tell which one failed, and the resulting number measures the pair, not the platform.
10. **Reference workloads.** The measurement uses the workloads defined in §3, or a documented subset. Omissions are named. Substitutions are described in enough detail that a reader can judge comparability.
11. **Method published.** Enough detail that a competent reader could re-run the measurement without asking you anything.
12. **Per-run data available.** The record of individual runs is published, or provided on request. Aggregates alone cannot be audited.

### L3: Stress

*L2, plus the three edge behaviours that ordinary operation never exercises.*

Normal-operation measurement answers "does it drop things when nothing is wrong". L3 asks what happens at the boundaries, which is where the failures that hurt actually live. Each probe must be run and its result published, **including when the result is that nothing interesting happened.**

13. **Destination outage.** Make your own destination unavailable for a stated window and record, per platform: whether it retries at all, how many times, with what backoff, whether pending retries are visible anywhere in the platform's interface, whether anything notifies you, and whether recovery produces duplicates.
14. **Success-wrapped failure.** Have your destination answer with a genuine 2xx status carrying a failure in the body, and record what the platform writes into its run history, what it bills, and what it takes to detect it.
15. **Sustained load.** Hold a stated multiple of the workload's normal rate for a stated duration, and report delivery, latency drift and refusals across the window. State the absolute rate, not only the multiple: a large multiple of a small number is still a small number.

### On claiming a level you do not hold

The failure mode this section is designed against is a report that reads as rigorous while omitting the one thing that would make its number checkable. Every requirement above is a thing a reader can look for and fail to find.

If you meet L1 and want L2, run the measurement. Do not claim L2 because your data is large.

---
