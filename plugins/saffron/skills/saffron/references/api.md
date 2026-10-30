# API tests (`@api`)

A scenario tagged `@api` tests an HTTP API with no browser. The agent
records each request once, every later run replays it at zero tokens, and
`Then` steps are never healed, as for browser scenarios. A browser
scenario can also call the API in a step; see the end of this page.

## Before writing

1. **Pin the contract.** Schema checks validate responses against an
   OpenAPI 3 document committed to the project, never one fetched at run
   time:

   ```bash
   npx saffron api pin https://api.example.test/docs   # a Swagger UI page, a document URL, or a file
   ```

   It writes `api/openapi.json`. Add it to `saffron.config.json`, with the
   API's address when it differs from the app's:

   ```json
   { "api": { "baseURL": "https://api.example.test", "contract": "api/openapi.json" } }
   ```

2. **Read what the API offers.** The `api_contract` MCP tool (or the
   pinned file) lists every operation with its parameters, body fields and
   response codes. Write steps for operations the API has, with the fields
   it accepts and the statuses it answers. Do not invent endpoints.
3. **Several APIs?** The default one is `api.baseURL`; others are named
   under `api.services` (`"orders": { "baseURL": ..., "contract":
   "api/orders.json" }`, pinned with `npx saffron api pin <url> --service
   orders`). `api_contract` lists each. A step that calls a named one says
   so ("through the orders API"); the recording keeps the name, never the
   host.
4. **Reuse the vocabulary**, as for any scenario: `npx saffron steps
   "through the API"` and `search_steps` show the wordings already recorded.

## Writing API steps

```gherkin
@api
Feature: Customers API

  Scenario: Subscribe a new customer to a plan
    When I create a customer named "cust-{unique:name}" through the API
    And I put the customer on the "Basic" plan for 3 months
    Then the customer should be on the "Basic" plan for 3 months
    And the customer should match the contract

  Scenario: A missing customer answers 404
    When I read customer 999999 through the API
    Then the response status should be 404
```

- **One request intent per `When` step**, with the values it sends in
  the step text: quoted strings, numbers, `<column>`s, `{data:...}`. The
  recording keeps them as the step writes them, and a number stays a
  number in the request body on every Examples row.
- **More than two values go in a data table** under the step, one row
  each: headers (an API key, a correlation id), query parameters, body
  fields. The step stays one readable sentence, and each cell records as
  its `<table:key>` wherever the request sends it (its path, a header,
  the body, a form field), so editing a cell replays at zero AI:

  ```gherkin
  When I validate the no-show reservation with these request details
    | reservation    | {data:no-show.chargeable} |
    | API key        | {env:NOSHOW_API_KEY}      |
    | correlation ID | {unique:correlation}      |
  Then the response status should be 200
  ```

  A whole JSON body the step sends as written goes in a `"""` doc string.
- **Records that must be new each run** take `{unique:name}`. A proof
  replay is a second execution, and every later run is another: each
  creates its own customer instead of colliding with the last one. Clean
  up with the project's `teardown` command, not with steps.
- **Refer to earlier results by intent** ("the customer", "the order I
  placed"), never by an id you saw while exploring. The agent keeps the
  response fields later steps need and records them as `{captured:name}`,
  so replay uses the id its own request got.
- **`Then` steps say what must be true of a response**: its status, a
  field's value, or "should match the contract" for a schema check. A
  `Then` that needs fresh data reads it itself (the agent sends a GET
  under that step). Never put a write in a `Then`.
- **Name the status of an error case** in the request's step or the
  `Then` right after it ("should answer
  404", "is refused with 409", "I get a 404", "the customer is not
  found"): a number elsewhere ("I order 450 units") is no status. A response that is no
  2xx is recorded only when its step names that status; otherwise the
  step fails, so a failure seen while recording never replays as the
  expected result.
- **Secrets stay `{env:VAR}` tokens**, typically in a step that names the
  credential: `Given I use the API token "{env:API_TOKEN}"`. The request
  sends the value and the recording keeps the token.
- **Outlines** work as for browser scenarios: write `<column>` in every
  step whose request or check depends on the row.

## What gets recorded, and what to check in `saffron diff`

| Action | Holds |
|---|---|
| `request` | method, path, the named API it goes to (`service`, absent for the default), headers, the JSON body, the fields it keeps (`extract`: name to JSON Pointer, a segment spelling a column the step names written as `<column>`), and the status it expects |
| `expectStatus` | the status of the latest response in that step's session, exact (`404`) or a class (`2xx`) |
| `expectJson` | field checks by JSON Pointer, typed (`/months` equals `3` as a number) or present/absent; an entry of a list with `item` (`{"path": "/messages", "item": [{"path": "/name", "equals": "ann"}]}`), wherever it stands |
| `expectSchema` | the latest response against the pinned contract: the operation's response schema, or a named component schema |

Before accepting a proposal, check that:

- no id, token or timestamp from the recording run is written literally
  in a path, header or body (a kept value reads `{captured:name}`);
- a request to a named API carries `"service"` and a relative path, never
  that API's host;
- secrets read `{env:...}`, and values the step wrote keep their tokens;
- every `Then` has a status, field or schema check that states the step,
  and "should match the contract" became an `expectSchema`;
- an error case expects the status the step names;
- an entry of a list is an `item` check, not a position (`/0/name`),
  unless the step names the position ("the first order").

The agent verifies each check against the response before it records it,
so a recorded check held at recording time; review whether it is the
check the step asks for.

## Running and reading results

```bash
npx saffron run --filter @api                  # API scenarios alone: no browser needed
npx saffron run features/customers-api.saffron:4 --no-agent
npx saffron trace "A missing customer answers 404"   # the requests and responses, per step
```

- A write (`POST`, `PUT`, `PATCH`, `DELETE`) is never retried on replay:
  a timeout fails the step instead of creating a second record. Reads are
  retried as usual.
- A recording with a schema check is checked again when the pinned
  contract changes, or when `api.contract` names another file: a zero-AI
  replay of the same recording, filed for review (yellow) when it passes,
  red when the API breaks the new contract, and no agent session either way.
  It never passes green over a contract it was not checked against.
  `saffron run --rerecord` records it again.
- `api.transcript` sets what the request transcript keeps: `"on-failure"`
  (default), `"on"` (green scenarios too) or `"off"` (statuses and timings
  only). The report lists each scenario's requests per step, and
  `saffron trace` replays an API scenario from its transcript. Nothing is
  drawn while a scenario runs.

## API calls in a browser scenario

A browser scenario (no `@api` tag) can call the API in a step, for setup
the UI would make slow or for a check the page does not show:

```gherkin
Scenario: A customer created through the API shows in the admin list
  Given I am signed in as an admin
  When I create a customer named "cust-{unique:name}" through the API
  And I open the customers page
  Then the customer I created should be listed
```

The request goes through the browser session's own client, so it carries
the cookies the UI signed in with, and a value it kept can be checked on
the page. Keep each step either a browser step or an API step. The agent
has the API tools in a browser scenario only when `saffron.config.json`
sets `api` (`"api": {}` is enough), or once the scenario's replay has
called the API.
