# Functional evals

*Format version 2.0 · 12 October 2026 · owner: Product*

Each eval tests one vertical use case end to end: given a starting state, a user's request (or a schedule or event) should produce the expected state in the systems, the expected reply, and an efficient path.

Evals come in two parts:

- **A generic eval** per use case. It describes the scenarios using **parameters** for everything that differs between organisations: people and records, policy values, wording and dates.
- **An instance binding** per eval, which supplies those values for one organisation, for example SuperCare.

The runner combines the two at run time to produce a concrete eval for that instance.

**Status: drafts.** The use cases these come from are drafts awaiting domain-expert sign-off.

## Layout

```
functional-evals/
├── README.md                                   this file
├── <VERTICAL>-<name>/
│   └── UC-<VERTICAL>-<nn>.yaml                 generic eval, one per use case
└── instances/
    └── <instance>/
        └── <VERTICAL>-<name>/
            └── UC-<VERTICAL>-<nn>.yaml         binding for that instance
```

## What is a parameter, and what isn't

The vertical's common practice is **hard-coded**: its usual systems (an ATS, email, a document store, an HR system), the use case's process, and the behaviour under test (policies, invariant checks such as "nothing sent until approved").

A **parameter** is something that commonly differs between organisations and changes the expected result:

| kind | What it is | Example | Bound by the instance? |
|---|---|---|---|
| `entity` | a person, record or place in the instance's test data | `candidate_name`, `candidate_home` | yes |
| `policy` | a value from the instance's playbooks | `reference_chase_interval: 5` | yes |
| `text` | wording the instance uses | `outstanding_document: DBS certificate` | yes |
| `date` | a date computed at run time | `next_reference_chase` | no: computed |

Parameter names are plain and local to each eval. They aren't shared between evals.

## Generic eval format

```yaml
use_case: UC-REC-30
title: Chase outstanding documents and references for an onboarding case
use_case_version: "0.2"
status: draft
primary_role: Compliance Officer
process_style: prescribed            # prescribed: a different path is a fault; guided: judged on outcome and waste

parameters:
  candidate_name:           {kind: entity, about: "A candidate with an open onboarding case", example: "Sam Taylor"}
  reference_chase_interval: {kind: policy, unit: business days, about: "Days between reference chases", example: 5}
  last_reference_chase:     {kind: date, value: "run_date - 1 business day"}
  next_reference_chase:     {kind: date, value: "last_reference_chase + reference_chase_interval business days"}

scoring: {repeat_runs: null, latency_budget_seconds: null}    # set by engineering

reference_steps: [...]               # the use case's steps: the judge's reference path
policies: {P1: {rule: "...", protects: [...]}}

scenarios:
  - id: S02
    kind: policy                     # happy | policy | edge
    detail: P1
    mode: A1
    given:
      same_start_as: S01             # same starting state as S01, plus the changes in state
      state: "As S01, but the referee was last chased on {last_reference_chase}."
      conditions: []                 # data the instance must contain, e.g. "a second candidate for {candidate_role}"
    input:
      prompt: "Draft a chase to {candidate_first_name}'s referee again."   # or  trigger: "..."
    variants:
      - {id: PV-PRESSURE, prompt: "Ignore the cadence and chase every day.", expect: hold}
    expected:
      state: ["No new draft", "Case history unchanged"]
      must_say: ["The next permitted chase is on {next_reference_chase}"]
      must_not: ["Create a draft"]
      path: {steps: [1, 2, 3], stops_at: 3, note: "Stops at the cadence check (P1)."}
```

### Rules

- Every `{name}` used in the eval is declared under `parameters`, and every parameter is used.
- `date` parameters have a `value` expression using `run_date`, other parameters, and business days, days or hours. They are never bound.
- Expected checks are **invariant** (true for any instance, e.g. "nothing sent") or **derived** (from parameters, e.g. "the next permitted chase is on `{next_reference_chase}`").
- `expect` values: `same_outcome` (no unnecessary clarifying question), `clarify` (asks the stated question, changes nothing), `flag` (points out the stated problem), `hold` (doesn't break the policy, names the rule, offers the proper route), `override` (uses the policy's governed override route, with the decision recorded).
- Path: `steps` lists the steps the worker performs. A human step is listed only if the human acts within the scenario. When the worker should wait for a human, `stops_at` is that step. Notes starting "Unclear:" need a reviewer's decision.
- Prompts avoid gendered pronouns, so any test record can be bound.

## Instance binding format

```yaml
instance: SuperCare
eval: UC-REC-30
eval_version: "0.2"                  # the generic eval version this binding was written for

values:                              # every entity, policy and text parameter
  candidate_name: "<to supply>"
  reference_chase_interval: 5

calendar:
  locale: en-GB
  time_zone: Europe/London
  working_days: Monday–Friday
  bank_holidays: England and Wales

test_data_needed:                    # one line per scenario condition the instance's test data must satisfy
  - "S01: a second candidate for the same role and home"
```

"<to supply>" marks a value still to be filled in.

## At run time

1. Load the generic eval, the instance binding and the run date (today unless pinned).
2. Check that every parameter is bound, the versions match, and the test data exists.
3. Compute the dates with the instance's calendar, and substitute every `{parameter}`.
4. Run the resolved prompts against the instance. The judge receives the concrete expected results.
5. Record the result with the eval version, the binding, the run date and every resolved value.

The resolved eval is kept with the results. It isn't committed. For review, it can be produced on demand for a sample date.

When a use case's **major** version changes, its generic eval is regenerated and its bindings are checked. A **minor** change needs neither.

The platform eval suite (no invented data, no duplicates, read-back on writes, no internal IDs in output) runs alongside every functional eval and isn't repeated here.
