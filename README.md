# SFR: Silent Failure Rate

A specification for measuring how often an automation platform accepts work and then
fails to complete it without telling anyone.

**Status: draft, in progress.** Nothing here is stable until a `v1.0` tag exists.

- [`SPEC.md`](SPEC.md) - the specification
- `spec/sfr-v1.json` - machine-readable form (planned)

**Reference implementation: planned, not yet published.** This specification constrains
how a measurement is made and reported. It does not require any particular software,
including ours.

## What this is not

It is not a tool. Nothing here is installed or run. It is a definition of a metric and a
set of conditions a measurement has to meet before its number means anything.

## Maintenance floor

This is maintained by one person alongside other work, and the guarantees are deliberately
small so that they can be kept:

- **The specification is additive.** Published text is not edited in place. Corrections and
  clarifications are appended as errata with their own dates, so a claim of conformance to a
  version stays valid.
- **No support.** Issues are for two things only: a factual error in the specification, and
  an ambiguity that makes a requirement impossible to apply. Pull requests may sit unreviewed.
- **No cross-platform promises.** The reference workloads describe webhook-to-HTTP work. We
  do not undertake to extend them to other shapes.

## Licence

- Specification text: [CC BY 4.0](LICENSE-SPEC). Attribution is a licence term, not a courtesy.
- Code, when any is published: [Apache-2.0](LICENSE).
