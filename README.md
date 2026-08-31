# Transition Monitor

**A business analysis case study: taking a manual, 15-working-day reporting process to a governed, auditable specification a build team could pick up tomorrow.**

Real open data · fictional client · every artifact a BA job advert asks for.

[Case study write-up](https://jolaandco.com/transition-monitor) · [jolaandco.com](https://jolaandco.com)

---

## Status — read this first

This repository contains **analysis and design deliverables**, not a running system.

| | |
|---|---|
| ✅ **Delivered** | AS-IS analysis · TO-BE process design · BRD (20 business requirements) · FRD (43 functional requirements) · star schema · wireframes · Jira backlog with acceptance criteria · decision log |
| ⬜ **Not built** | The ingestion pipeline, the scoring engine and the Power BI report. Every Jira story is in **To Do**. |

That split is deliberate. The point of this project is the analysis phase — the part that
happens *before* code, and the part a business analyst is actually accountable for. The
pipeline and report are Phase 2, deferred on the record as decision
[D-009](docs/09-decision-log.md).

**Disclosure.** Terrawatt Advisory is a fictional consultancy, and its analysts are
invented. The data is real (Our World in Data, CC-BY), the method is real, and the
scenario is constructed. Naming the client rather than anonymising it was a deliberate
choice, recorded as decision [D-008](docs/09-decision-log.md) — a case study that hides
what is invented is not a case study, it is a claim.

---

## The problem

Terrawatt Advisory's analysts produce a Country Transition Scorecard on request. Today the
process is entirely manual, and the elicitation turned up numbers worth stating plainly:

- **~15 working days elapsed** per scorecard, against roughly **24 hours of touch time** — about **80% of the elapsed time is queueing, not working**
- Two analysts scoring the same country produced results **11 points apart** on a 100-point scale
- The scoring method **has never been written down** — it exists in one senior analyst's memory and in the cell formulas of whichever workbook was used last
- Two published scorecards disagreed on Spain and **neither could be reproduced**, because nobody recorded which data download they used
- Review waits averaged **4.1 working days** with no queue and no due date
- When the data source publishes an update, **every delivered scorecard silently goes stale**

The full elicitation write-up — interviews, observation, the workshop, the pain-point
register ranked by the participants — is in
[docs/04-as-is-process-narrative.md](docs/04-as-is-process-narrative.md).

### AS-IS process

![AS-IS process model in BPMN 2.0](diagrams/as-is-process.png)

## The solution

Transition Monitor ingests the OWID dataset automatically on every release, validates and
shapes it into a governed star schema, applies one standardised scoring method to every
country, routes each draft through in-system human review with an explicit approval
gateway, and publishes approved scorecards automatically with full versioning and audit
trail. A scorecard can only be published from an approved state, and superseded versions
are retained, never deleted.

The human review gate is kept on purpose. The target is to remove the queueing, not the
judgement.

### TO-BE process

![TO-BE process model in BPMN 2.0](diagrams/to-be-process.png)

---

## The traceability chain

The spine of the project. Every element traces in both directions:

```
20 business requirements   (BRD)
        ↓
43 functional requirements (FRD)
        ↓
 5 epics
        ↓
14 stories                 (each with Given/When/Then acceptance criteria)
        ↓
20 planned tests
```

Backwards from any screen element you reach a functional requirement, a business
requirement, and a TO-BE process activity. The full matrix is the
[traceability table](docs/06-frd.md#traceability-table-br--fr--epic--story--test).

The backlog was reconciled *after* it was built, not assumed: the plan called for 20
stories, the built backlog has 14, and the difference is written down rather than tidied
away — BR-09 was consolidated into the same story as BR-08 because both cover the same
transformation work. That reconciliation is decision
[D-010](docs/09-decision-log.md).

## Governance

Ten decisions are logged with rationale and impact. **Two are still open**, deliberately:

- **D-001** — should nuclear count as "clean"? It changes the score of every country with a nuclear fleet. That is a position the sponsor owns, not the analyst.
- **D-002** — which region standard for roll-ups, World Bank or OWID? Deferred until there is a consumer who can say which one they report against.

Closing those quietly would recreate exactly the pain point this project exists to fix: an
undocumented methodology that lives in somebody's head.

---

## Data model

A single-fact star schema — one fact table at a declared grain of *country × indicator ×
date × version*, with four conformed dimensions. Modelled as a UML class diagram in
Enterprise Architect.

![Star schema as a UML class diagram](diagrams/star-schema.png)

A richer design would split energy observations and index scores into two fact tables. One
fact covers the reporting core, and calling it a star rather than a constellation keeps
the label honest — recorded as decision [D-007](docs/09-decision-log.md), with the
constellation documented as the evolution path.

The governance flags live on the fact table (`insufficient_data_flag`,
`excluded_from_ranking_flag`, `micro_state_flag` on the country dimension), so a country
that cannot be scored is *flagged, not silently dropped* — decision
[D-003](docs/09-decision-log.md).

## Wireframes

![Scorecard view wireframe, with callouts tracing each element to a functional requirement](diagrams/wireframe-1-scorecard-view.png)

Deliberately low-fidelity: these specify layout, content and behaviour for handoff, not
visual design. Callouts trace every screen element back to its functional requirement, and
the approval controls mirror the "Approved?" gateway in the TO-BE model.

Note the deliberate *absence* of a Publish button on the reviewer screen
([wireframe 2](diagrams/wireframe-2-review-approval.svg)) — publication is automatic on
approval, so offering a manual publish would contradict FR-37.

## Methodology as configuration

The five TTI component weights are in [`config/weights.yml`](config/weights.yml), not in
code — decision [D-004](docs/09-decision-log.md). Weights are a business decision that
changes with methodology versions, so they are governed as configuration and the run fails
if they do not sum to 1.0.

The same file carries the rules that stop an index misleading: a minimum of three of five
components before a country is scored at all, outlier capping before normalising, a
generation floor below which a country is scored but not ranked, and an interpolation gap
limit that flags whatever it fills.

---

## Repository contents

| Path | What it is |
|---|---|
| [`docs/03-brd.md`](docs/03-brd.md) | Business Requirements Document — 20 BRs mapped to TO-BE activities and epics |
| [`docs/04-as-is-process-narrative.md`](docs/04-as-is-process-narrative.md) | Elicitation write-up, business rules discovered, ranked pain points, open questions |
| [`docs/06-frd.md`](docs/06-frd.md) | Functional Requirements Document — 43 FRs, acceptance-criteria pattern, full traceability table |
| [`docs/07-data-dictionary.md`](docs/07-data-dictionary.md) | Columns in scope, non-null profiling, five data-quality findings |
| [`docs/08-jira-backlog.md`](docs/08-jira-backlog.md) | Jira export — 5 epics, 14 stories, every story with Given/When/Then AC |
| [`docs/09-decision-log.md`](docs/09-decision-log.md) | Ten decisions with rationale, status and impact |
| [`docs/10-glossary.md`](docs/10-glossary.md) | Domain, modelling and delivery terms |
| [`diagrams/`](diagrams) | BPMN 2.0 source (`.bpmn`), SVG and PNG renders, star schema, wireframes |
| [`model/`](model) | Enterprise Architect repository (`.qea`) |
| [`config/weights.yml`](config/weights.yml) | TTI methodology configuration |
| [`data/`](data) | OWID codebook; see [`data/README.md`](data/README.md) for the dataset itself |

The BPMN files are portable and open-standard — drop
[`diagrams/as-is-process.bpmn`](diagrams/as-is-process.bpmn) into
[demo.bpmn.io](https://demo.bpmn.io) to open the model in your own tool rather than taking
a picture's word for it.

## Method and tools

BPMN 2.0 · UML · Enterprise Architect · bpmn.io · Confluence · Jira · star-schema
(Kimball) dimensional modelling · AgilePM / DSDM

## Phase 2

The pipeline (pandas) and the Power BI report implementing this star schema. Deferred
deliberately — the analysis phase is the deliverable here, and shipping a half-built
pipeline alongside it would blur what this project is demonstrating.

---

**Jolanta Tacij** — Business Analyst · Agile Delivery
[jolaandco.com](https://jolaandco.com) · [LinkedIn](https://www.linkedin.com/in/jola-tacij)

Analysis and documentation © Jolanta Tacij. Underlying energy data © Our World in Data,
[CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/).
