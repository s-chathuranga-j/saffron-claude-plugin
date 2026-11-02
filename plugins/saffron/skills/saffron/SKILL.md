---
name: saffron
description: Create, translate, or edit Saffron test scenarios in .saffron or .feature files, browser UI tests and HTTP API tests (@api) alike, including validating their recordings and replay. Use when asked to author or fix these test files in a Saffron project. Do not activate for ordinary application development, generic Cucumber projects, or requests only to run existing tests, inspect reports, review proposals, or configure Saffron.
license: Saffron Free Use License v1.0 (see the saffron-ai package LICENSE)
metadata:
  saffron-ai: "1.3.1"
  author: saffron-ai
  homepage: https://saffron-ai.io
---

# Saffron

This skill describes saffron-ai 1.3.1 (see the compatibility page: https://saffron-ai.io/docs/compatibility).

Apply this workflow only when asked to create, translate, or edit Saffron
scenarios in `.saffron` or `.feature` files. A Saffron mention or an
existing test file in the project does not activate authoring work.
The run, review, and configuration commands below support that authoring
task; they are not instructions to change unrelated tests or settings.
Respect draft-only or no-execution requests and report the remaining
validation explicitly.

Saffron runs end-to-end tests written in plain language, in a real
browser. A `.saffron` file reads like Given/When/Then and goes further:
StepSets, several people in one scenario (`As "buyer"`), data tokens and
data-driven outlines; a plain `.feature` file runs as it is. There are
**no step definitions and no glue code**: on the first run an AI agent performs
each step and records what it did into a JSON cache; every later run
is replayed by Saffron in a real browser: zero AI calls, zero tokens, at
the speed of the browser and the app. (The cache is Saffron's own JSON, run
on Playwright's browser engine; Playwright MCP is used only by the agent
while recording and healing.) When the UI drifts, the agent heals the failing
step mid-run and files a reviewable proposal.

A scenario tagged `@api` tests an HTTP API instead, with no browser: the
agent records each request and the checks on its response, against the
project's pinned OpenAPI contract. The rules and workflow below hold for
both; [API tests](references/api.md) has what is specific to them. Read
it before writing or reviewing an `@api` scenario, or a step that calls
the API from a browser scenario.

Your job when writing tests is to make that economy work. Three rules
matter more than everything else:

## 1. Exact step text is the cache identity: reuse the vocabulary

A step is looked up by its exact wording. `When I visit the login page`
and `When I open the login page` are two different steps: the second one
costs a fresh AI recording (~$0.10–$2) even though the first already
replays for free.

**Before writing any step, look at what the project already knows:**

```bash
npx saffron steps                 # every step: ● recorded, ● divergent, ○ unrecorded
npx saffron steps "login"         # search
npx saffron steps --json          # machine-readable
```

Prefer a `●` recorded wording verbatim. Quoted values may differ
(`I enter "admin"` seeds from `I enter "bob"`), so parameterize with
quotes rather than inventing new phrasings. A recording that still
names the old value elsewhere (a `delete-bob` test id, a `#row-bob`
selector) is not reused for the new one: that step records afresh. `saffron init` registers the
`saffron` MCP server with Claude Code, Cursor and VS Code; its
`search_steps` / `list_step_sets` / `project_status` tools give the same
answers without a shell, `run_plan` says what a run would replay and
record (and which new steps to record and accept first) before it costs
anything, `api_contract` lists the operations of the pinned API
contract for `@api` scenarios, and `verify` replays what you changed at
zero AI and reports any check you weakened against the reviewed baseline.

**Know the project before touching it:**

```bash
npx saffron status                # files and scenarios (● cached, ◐ proposal pending, ○ unrecorded), tags, proposals, last run, health
npx saffron status --json         # the same as data (what the IDE panels and the project_status MCP tool read)
npx saffron run --plan            # what a run would replay, record or refuse, and the agent sessions it needs, without running
```

Use it to pick tags that already exist, to see which scenarios are
cached (editing their text re-records them), which proposals await
review, and whether the vocabulary has divergent steps or duplicate
wordings to converge before adding more.

## 2. `Then` steps are the verdict

Final `Then` steps must check the state reached by the actions. Never
change guests, rooms, filters, or other application state inside an
assertion to manufacture its expected result. Mid-scenario assertions
may adapt only under the project's explicit policy; final assertions
remain strict. State exactly what must be true using stable text or
named captures for dynamic values (prices, dates, counters). A `Then`
that records no check at all fails on replay, so every assertion step must
end up with something recorded against it. Put
the final assertions **last**; the trailing block of `Then` steps is
strict under every policy. Use `Given`/`When` for actions. A text check
passes only on text the page shows: hidden elements, hidden parts of an
element and hidden dropdown options never count, so "should be shown"
means visible.

## 3. Never bake volatile or secret values into files

- Secrets: `{env:VAR}`, as in `When I enter "{env:ADMIN_PASSWORD}" in the password field`.
- Shared test data (accounts, names, enum values): put it in `data/*.json` and write `{data:users.admin.email}` (file, then dotted path; one value, not a list). The recording stores the token, so editing the file changes the next replay at zero tokens. Never for secrets: data files are committed. To check a whole list (an enum), point a Then step at it: `Then the status filter should list every {data:enums.OrderStatus}`; replay iterates the file's list, so adding a value needs no re-record. `Examples: {data:roles}` reads outline rows from `data/roles.csv` or a JSON list (`.saffron` only). A value that must be new on every run (an email for a sign-up) is `{unique:name}`: fresh per run, the same within it.
- State the application must be in before the suite (seeded users, a reset database): do not script it through the UI in every scenario. The project's `saffron.config.json` takes `"setup"` and `"teardown"` shell commands, run once around `saffron run` (teardown always runs). Propose a seed script there; keep scenario steps for the behaviour under test.
- Dates: write intent when the day should move ("1 day from today" records as `{date+1}`); a date you write out (`I enter 2026-10-10`, in a table, a data file or `{env:...}`) is entered as written on every run. An Examples cell such as `{date+2:DD.MM.YYYY}` gives each row its own day. A date the app shows in words or as a range is a token too (`{date+7:D. MMM:de-DE}`, `{date+7 - +9:MMM D, YYYY}`), never a setup script.
- Dynamic display values: capture and compare, e.g. `I record the total as "first"` … `"first" should differ from the displayed total`.

## Authoring workflow

1. **Survey once.** Read the relevant scenarios and project config, then
   `saffron status --json` (or `project_status`). Search the vocabulary
   with `saffron steps "<words>"` / `search_steps` and check `list_step_sets`.
   Reuse recorded wording only when it expresses the same behavior.
2. **Inspect the behavior.** For a new or uncertain flow, use the available
   browser tools to walk the relevant actions and observe their results.
   Check the actual labels, initial state, comparison target, and loading
   signal. Exercise each transition you will assert; seeing the controls
   alone is insufficient. Existing tests and source code help locate the
   flow, but do not establish its current UI behavior. Reuse recent,
   relevant browser evidence instead of exploring the whole application.
   If access or test data is missing, identify the gap and produce a draft
   with explicit assumptions; do not invent observations. For an `@api`
   scenario, read the pinned contract (`api_contract`) instead of the
   page, and where access allows, send the request once to see a real
   response; pin a contract first if the project has none.
3. **Design one behavior per scenario.** When translating Cypress or other
   tests, preserve the intended coverage, reconsider assumptions, and
   split independent changes. Compare the same target immediately before
   and after the change under test. Do not assume every intermediate value
   must differ from every earlier value. Keep required setup and waits;
   there is no fixed step-count limit.
4. **Write to the contract below.** Add to the feature for that area. In
   `.saffron`, reuse StepSets for repeated flows; `.feature` uses standard
   Gherkin without StepSets or file-backed Examples. Resolve syntax and
   data diagnostics and check existing wording before recording.
5. **Record the affected scenarios.** Select each new or changed scenario
   by line or name, for example `saffron run features/x.saffron:<line>`.
   If a shared StepSet changes, include its affected callers. Avoid a
   whole-suite recording or disabling reuse by default. Re-record a faulty
   recording only after diagnosing its failed step and proof evidence.
   **Record shared setup once, and accept it before the rest.** Step reuse
   copies steps only from accepted recordings (`.saffron/cache/`). A
   pending proposal replays for its own scenario and its Examples rows,
   never for other scenarios, by design: an unreviewed recording must not
   spread. So when new scenarios share a Background or StepSet, record one
   of them, review and accept its proposal (steps 6 and 7), then record
   the others: they replay the shared steps at zero tokens and the agent
   records only what differs. Recording them all before accepting anything
   pays for the shared steps in every scenario.
6. **Review what was recorded.** Use `saffron diff` and the report/trace.
   Check that captures and comparisons use the same names and intended
   elements, waits observe the real signal, and final assertions contain
   no state-changing actions or replacement baselines. A `verified ✓`
   proposal proves replay passed; it does not prove the recording faithfully
   tests the written requirement. Investigate `UNVERIFIED` proposals; do
   not accept them to finish the task or weaken the expected behavior.
7. **Prove completion.** Accept only the reviewed, verified proposals for
   this task, then run the affected scenarios with `--no-agent`. Complete
   means those scenarios pass with zero AI calls and zero tokens, with
   their intended assertions preserved. Report exactly what passed and
   any scenarios or prerequisites not checked. If acceptance is reserved
   for someone else or execution is unavailable/excluded, deliver the
   draft or pending proposals and name the unfinished validation. Keep
   feature files, shared data, and accepted caches together for review;
   commit only when the task or project workflow calls for it.

## Writing steps that record reliably

- **One clear intent per step.** Use visible labels, roles, and names:
  `the "Save" button`, `the email field`. Keep CSS selectors, XPath, test
  ids, and component names out of scenario prose. Granularity means an
  independently understandable action or check, not necessarily one click.
- **Name captured values explicitly.** Write `I record the displayed price
  for the selected date as "beforePrice"`, then `the displayed price for
  the selected date should differ from "beforePrice"`. Reuse the exact
  name; avoid `the price I noted` or `the previous value`. Record a baseline
  before the action, and never recreate it inside the final assertion.
- **Keep the target and value comparable.** Compare the same date, item,
  or row across the change. If testing an amount change, capture amount-only
  text, not a whole button label containing a date, currency, or guest count.
- **State the arithmetic when the amount matters.** "Changed" does not
  prove "added the add-on's price". Capture each input, then say the sum:
  `I record the basket total as "before"`, `I record the breakfast price as
  "breakfast"`, then `Then the basket total should be "before" plus
  "breakfast"`. Replay reads the one number each element shows, in either
  decimal style, and compares to the cent. Minus and "times 2" work too;
  percentages, rounding rules and taxes do not.
- **Check the route, not only the destination.** When the requirement is
  a path (a 3DS challenge, a redirect, a confirmation dialog), add a
  mid-scenario `Then` for each checkpoint, such as `Then I should see the
  card authentication challenge`, before the final verdict. A success page
  alone does not prove which way the flow went. Name the exact control a
  variant uses (`the "Guarantee your booking by card" button`, not "pay");
  two similar buttons are not interchangeable.
- **Put a step's values in a data table when there are more than two.**
  A step reads as one sentence of intent; the values it enters or sends go
  in a 2-column table on the lines below it (a key and its value), not in
  one long line of quoted parameters. This holds for an API request's
  headers, query parameters and body fields as much as for a form:

  ```gherkin
  When I validate the no-show reservation with these request details
    | reservation    | {data:no-show.chargeable} |
    | API key        | {env:NOSHOW_API_KEY}      |
    | correlation ID | {unique:correlation}      |
  ```

  Each cell records as its `<table:key>`, so editing a value replays at
  zero AI; renaming or adding a key re-records the step. Use a header
  table for a list of records, and a `"""` doc string for free text or a
  whole JSON body. Keep one or two values inline when they read as part of
  the sentence (`When I read customer "cust-42" through the API`).
- **Name the section a step acts in** when the page repeats labels (a
  guest form and a booker form): `I replace "E-mail" in "Guest information"
  with "..."`. The recording is then scoped to that section, so it acts on
  the right form even when both are shown.
- **Wait for an observed result.** State the relevant visible state or API
  response instead of `finished loading` without a signal or a fixed sleep.
  A currency option called `GBP` may display `£` on prices. Use the observed
  display value for the price wait; a changed selector alone does not prove
  prices refreshed. Do not wait for the expected assertion outcome merely
  to make it pass; the final check must still establish the requirement.
- **Keep test inputs explicit.** Use the data, secret, and date rules above.
  Establish the starting guests/rooms or other relevant state. Name required
  seeded data instead of silently assuming it exists. Conditional actions
  such as `accept cookies if prompted` record a no-op when absent; that is
  not a promise that replay will discover every future branch.

For example, after selecting a date and observing its price:

```gherkin
Scenario: Changing two adults to one updates the selected date's price
  Given the rooms and guests button shows "1 room, 2 adults"
  When I record the displayed calendar price for the selected date as "beforePrice"
  And I open the rooms and guests panel
  And I reduce the adults in room 1 from 2 to 1
  And I wait for the calendar price refresh indicator to disappear
  Then the rooms and guests button should read "1 room, 1 adult"
  And the displayed calendar price for the selected date should differ from "beforePrice"
```

Use this shape only where the app actually has that refresh indicator and
pricing rule. Setup must select a date with suitable test availability.
An independent child or second-room price change belongs in its own
scenario with its own baseline. For currency changes, distinguish the
selector's code from the observed display symbol and capture amount-only
text. See [syntax.md](references/syntax.md#a-complete-file) for a full file.

`saffron author` drafts from supplied requirements and vocabulary without
browser inspection. Include observed labels, waits, prerequisites, and
expected behavior in its input. Its output must still go through steps
5 to 7 above before it is described as a working test.

## Several people in one scenario: `As "name"`

```gherkin
Given As "seller" I am on the orders page
When As "buyer" I order the blue lamp        # the buyer's own browser
Then As "seller" I see an order for the blue lamp
And the order is marked new                  # still the seller
```

Each name is its own browser session (cookies, login, tabs). It carries
over to the next steps until another `As "..."`; `As "default"` goes
back. Give each person a login with `"sessions": { "buyer": {
"storageState": ".auth/buyer.json" } }`; a person without an entry
starts from the top-level `storageState` (two such people start as the
same user, and the run warns). Keep a step's `As "..."` when rewording
it, and never copy one person's prefix onto another's step: a recording
is reused only within its own session, so the buyer's typed address
never seeds the seller's step.

## API tests: `@api`

```gherkin
@api
Feature: Customers API

  Scenario: Subscribe a new customer to a plan
    When I create a customer named "cust-{unique:name}" through the API
    And I put the customer on the "Basic" plan for 3 months
    Then the customer should be on the "Basic" plan for 3 months
    And the customer should match the contract
```

One request intent per `When` step. A value or two can sit in the step
text; more (an API key, a correlation id, query parameters, body fields)
go in a 2-column data table under the step, one row each, never one long
line. Records
that must be new each run take `{unique:name}`. Refer to an earlier
result by intent ("the customer"), never by an id you saw: the agent keeps
the fields later steps need. `Then` steps state a status, a field's value
or "should match the contract", and name the status of an error case.
Pin the contract first (`npx saffron api pin <url or file>`), and read
[API tests](references/api.md) for the recording, review and run details.

## Repeated step groups → StepSets (`.saffron` files only)

`.saffron` is a superset of `.feature`: rename the file, nothing else
changes, and you gain the `StepSet:` keyword.

```gherkin
StepSet: Complete guest information            # define: colon, like Scenario:
    Given I am on the guest information page
    When I enter guest name "Chathuranga"
    And I click the continue button
    Then I am not on the guest information page

Scenario: Checkout happy path
    Given I am on the cart page
    StepSet Complete guest information         # invoke: no colon, like a step
    Then I am on the checkout page
```

- Definition: `StepSet: <name>`. Invocation: `StepSet <name>`, the bare
  name without the keyword is a silent parse error.
- Names are project-wide unique; sets resolve across files; no nesting.
- Application-wide flows (login, cookie banner) live in a sets-only
  library file: `features/shared.steps.saffron` (still needs a
  `Feature:` header; it yields no runnable scenarios).
- Start a set with a guard step, end it with an exit assertion.

## Commands for completing an authored test

```bash
npx saffron run features/login.saffron:12   # record/replay one affected scenario
npx saffron diff                            # inspect pending recordings
npx saffron trace "<scenario>"              # inspect a failed or adapted step (trace enabled)
npx saffron accept <proposal-file>          # accept a reviewed, verified proposal
npx saffron run features/login.saffron:12 --no-agent  # prove cached replay
npx saffron status --json                   # confirm affected scenario state
npx saffron api pin <url or file>           # pin the API contract @api scenarios check against
```

Results: **green** = cached pass · **yellow** = pending review · **red** =
failed. `verified ✓` means the proposal's zero-AI proof replay passed;
review its meaning before accepting. `UNVERIFIED ✗` needs investigation.
A STALE proposal belongs to an older scenario or recording: run the
scenario again for a fresh one. Keep accepted `.saffron/cache/` files
with the test; ignore `.saffron/reports/`.

For outlines, check every affected Examples row with `--no-agent`, not
only the row used to record. Quote the row's expected value in `Then`
(e.g. `Then I should see "<message>"`) so the recording keeps the parameter.
One recording serves every row, and it cannot tell whether a row value
glued inside other text belongs to the row (an email built from `<user>`,
`INV20261013`, `orders-US`) or a field lists every row's value. Write
`<column>` in every step whose text or typed value depends on the row;
otherwise the recording is bound to its row and the other rows are
refused (red, at zero tokens) with guidance naming the column and the
step. That includes dates without a year and one- or two-character
values. A blank cell is refused the same way, or, when its step types
`<column>`, the first filled row records once more and serves the blank
rows too. Keep a check that lists all rows' values out of the Outline.
Tests live in the folder named by `features` in `saffron.config.json`
(`features/` by default).

**Never hand-edit generated `.saffron/cache/` or `.saffron/proposals/` to
make a test pass.** Change incorrect scenario text and record again, or
use `saffron run <scenario-selector> --rerecord` when the recording is wrong.
Inspect failures before another paid recording; report an application,
fixture, or unsupported-operation blocker instead of repeatedly retrying
or weakening assertions. For CI artifacts, browser matrices, provider
setup, and other configuration needed by the requested test, read
[Configuration & CI](references/config.md).

## When Saffron cannot express a value: a small supporting script

Use Saffron itself first: `{date...}` tokens (ranges and locales included), captures and comparisons, `{data:...}` and `{unique:...}`, the checks in the syntax reference, and `setup`/`teardown`. A recording that freezes a value or checks a pattern any value would pass is a recording to reword or re-record, not a missing capability. Reach for a script only when the runner cannot express a requirement at replay, and ask the user first when it would add a dependency, a hook, or a file outside the project's existing scripts folder.

- Keep it small: one file in the project's existing scripts folder, in its runtime and module style, using the standard library or dependencies already installed. Clicks and checks stay in the scenario; the script only prepares data.
- Expected values come from the test's inputs or an independent source (a contract, a fixture), never from the output being checked. Generate volatile values at run time, and state locale, time zone, format and day-boundary assumptions in the script.
- Run it as a `setup` command in `saffron.config.json`, keeping the hooks already there. It writes a git-ignored file under `data/`, which scenarios read through named `{data:...}` tokens: Saffron reads data after setup, so every replay sees fresh values. Commit the script and its configuration, never what it generates, and never write secrets into it or its output. On failure it exits non-zero with a message: setup failing stops the run before any scenario. `--no-hooks` skips it: scenarios then read its last output, or stop naming the missing file; say so.
- Never weaken a check to fit, and never hand-edit caches or proposals.
- Check it in proportion: its logic at the boundaries it handles (month end, year end, the time zone), the recording's checks and `{data:...}` tokens reviewed, only verified proposals accepted, and the scenario passing with `--no-agent`.
- Report which files you wrote, which the script generates and which recordings Saffron made, why the script is needed, and which Saffron capability could replace it.

## Anti-patterns that cost money or hide bugs

| Don't | Do |
|---|---|
| Invent a synonym for an existing step | Reuse the `●` wording from `saffron steps` |
| Assert inside a `When` ("When I see the dashboard") | Act in `When`, assert in `Then` |
| Hard-code a changing price observed during authoring | Capture and compare for a change; use an exact price only when the requirement fixes it |
| Put a password literal in a step | `{env:VAR}` |
| Sleep/wait "for 3 seconds" | Wait for a UI signal or an API response |
| Several independent changes and all-pairs comparisons in one scenario | One behavior and its own baseline per scenario; reuse setup with StepSets in `.saffron` |
| Edit cache JSON to fix a test | Edit the scenario text and re-run, or `saffron run --rerecord` if the recording itself is wrong |
| Write an id you saw in a response into a later API step | Refer to it by intent ("the customer I created"); the recording keeps it as `{captured:name}` |
| Create a fixed record (`"cust-1"`) in an API step | `{unique:name}`: every run, and the proof replay, creates its own |
| Check an API response's status in a `When` | Name the status in a `Then` ("should answer 404") |
| One long step line carrying an API key, ids and several quoted parameters | A short step and a 2-column data table under it, one value per row |

## References

- [Syntax cheat sheet](references/syntax.md): tables, doc strings,
  outlines, secrets, dates, network waits, page furniture, StepSets.
- [Configuration & CI](references/config.md): `saffron.config.json`,
  flags, `--strict`, cross-browser and browser matrix, devices, workers
  and agent workers, heal model, secrets setup, the API settings.
- [API tests](references/api.md): `@api` scenarios, the pinned contract,
  writing and reviewing request steps, API calls from a browser scenario.
