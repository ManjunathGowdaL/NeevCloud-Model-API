# NeevCloud Model API — Automated Tests (Postman)

A Postman collection covering functional, negative, edge-case, auth, and
rate-limit scenarios for the NeevCloud **Model API** (OpenAI-compatible
inference endpoints: chat completions, models, audio/speech).

- Console: https://console.ai.neevcloud.com
- Docs: https://docs.ai.neevcloud.com/ai-inference/overview-1
- OpenAPI reference: https://docs.ai.neevcloud.com/api-reference/model-routing

## Why Postman/Newman

The Model API is a plain OpenAPI-compatible REST/JSON service — no UI
involved — so Postman/Newman gives a GUI-runnable suite for manual
exploration plus a CLI story for CI, without needing a code runtime
beyond Node.

## Repository layout

```
.
├── NeevCloud-Model-API.postman_collection.json    # the collection (62 requests)
├── NeevCloud-Model-API.postman_environment.json   # environment (fill in api_key)
├── build_collection.py                            # generator script (see below)
├── data/                                          # Newman/Runner data files for
│   ├── malformed-model-ids.json                   # parametrized requests
│   ├── bad-temperature-values.json
│   ├── bad-top-p-values.json
│   ├── bad-max-tokens-values.json
│   ├── temperature-boundaries.json
│   ├── top-p-boundaries.json
│   ├── speed-values.json
│   └── bad-speed-values.json
├── results/
│   └── latest-run.json                            # most recent live test-run export
├── LICENSE
└── .gitignore
```

`build_collection.py` generates the collection JSON programmatically.
It exists so the (large, easy-to-corrupt-by-hand) collection file can be
regenerated safely — edit `build_collection.py` and rerun it rather than
hand-editing the JSON when adding new requests.

## Structure of the collection

| Folder | Endpoint | Contents |
|---|---|---|
| **1. Authentication** | all | missing/invalid keys, malformed `Authorization` header, empty bearer token, leak check |
| **2. Models** | `GET /v1/models`, `GET /v1/models/{id}` | Functional / Negative / Edge Cases sub-folders |
| **3. Chat Completions** | `POST /v1/chat/completions` | Functional / Negative / Edge Cases sub-folders |
| **4. Audio Speech** | `POST /v1/audio/speech` | Functional / Negative / Edge Cases sub-folders |
| **5. Rate Limiting** | `POST /v1/chat/completions` | opt-in burst request, run via Runner/Newman iterations |

Every request has a `Tests` script (`pm.test(...)`) asserting status code
and, where relevant, response schema (checked field-by-field rather than
via a JSON-Schema library, so it's portable across Postman/Newman
versions). A collection-level test script also runs after **every**
request to check response time stays reasonable.

Requests that must send bad/missing credentials use `"auth": "noauth"`
plus an explicit header override, so they don't inherit the
collection-level Bearer auth — this is what lets both "valid key works"
and "invalid key is rejected" live in the same collection safely.

Negative/edge coverage includes: missing required fields, wrong types,
out-of-range values (temperature, top_p, max_tokens, n, speed,
penalties), malformed JSON bodies, wrong `Content-Type`, oversized/
unicode input, path-injection-style model IDs, and consistency checks
between `/v1/models` and `/v1/models/{id}`.

## Setup

1. Import both JSON files into Postman (**Import → File**), or point
   Newman at them directly (no import needed for CLI use).
2. In the **NeevCloud Model API - Local** environment, set `api_key` to a
   real key from console.ai.neevcloud.com → AI Inference → Security → API
   Keys. Everything else has a sensible default (see table below).
3. Select that environment in Postman's environment dropdown before
   running requests in the GUI.

**Never commit a real API key.** The environment file ships with a
placeholder (`sk-nc-your-real-key-here`) — set your real key locally via
Postman's environment editor, which keeps it out of version control.

### Variables

| Variable | Purpose | Default |
|---|---|---|
| `base_url` | Model Routing API base URL | `https://inference.ai.neevcloud.com` |
| `api_key` | **Required.** Valid key used for all functional/negative/edge requests | _(fill in)_ |
| `invalid_api_key` | Deliberately bogus key for auth-failure tests | `sk-nc-invalid...` |
| `chat_model` | Model id used to drive chat tests | `llama-3.1-8b-instant` |
| `nonexistent_model` | Syntactically valid but absent model id | `definitely-not-a-real-model-xyz-123` |
| `tts_model` / `tts_voice` | Model/voice for audio tests | `tts-1` / `alloy` (**unconfirmed** — see Assumptions) |
| `default_max_tokens` | Kept small to minimize token spend | `8` |

## Running

### Postman GUI

Open the collection, right-click any folder → **Run folder**, or run the
whole collection with the Collection Runner. For the data-driven requests
(see below), attach the matching file from `data/` in the Runner's
**Data** tab.

### Newman (CLI / CI)

```bash
npm install -g newman

newman run NeevCloud-Model-API.postman_collection.json \
  -e NeevCloud-Model-API.postman_environment.json \
  --env-var api_key="sk-nc-your-real-key"
```

Run a single folder:

```bash
newman run NeevCloud-Model-API.postman_collection.json \
  -e NeevCloud-Model-API.postman_environment.json \
  --folder "3. Chat Completions"
```

Generate an HTML report:

```bash
newman run NeevCloud-Model-API.postman_collection.json \
  -e NeevCloud-Model-API.postman_environment.json \
  -r htmlextra --reporter-htmlextra-export report.html
```
(`npm install -g newman-reporter-htmlextra` first.)

## Data-driven (parametrized) requests

A handful of requests are written once and driven by a data file rather
than duplicating near-identical requests for each value — e.g. one
request reads `{{bad_temperature}}` and iterates over
`data/bad-temperature-values.json`:

```bash
newman run NeevCloud-Model-API.postman_collection.json \
  -e NeevCloud-Model-API.postman_environment.json \
  --folder "3. Chat Completions" \
  -d data/bad-temperature-values.json
```

Requests marked this way (name includes "data file: ..."):

| Request | Data file | Values |
|---|---|---|
| Retrieve Model - Malformed/Malicious ID | `malformed-model-ids.json` | path traversal, XSS, SQLi, null-byte, spaces, slashes |
| Invalid Temperature Value | `bad-temperature-values.json` | `-1, 2.5, "hot", null` |
| Invalid top_p Value | `bad-top-p-values.json` | `-0.1, 1.1, "high"` |
| Invalid max_tokens Value | `bad-max-tokens-values.json` | `0, -5, "many"` |
| Temperature Boundary | `temperature-boundaries.json` | `0, 2` |
| top_p Boundary | `top-p-boundaries.json` | `0, 1` |
| Speed Within Bounds | `speed-values.json` | `0.25, 1, 4` |
| Invalid speed Value | `bad-speed-values.json` | `0.1, 4.1, -1, "fast"` |

These numeric fields are typed correctly (not sent as quoted JSON
strings) via a pre-request script that reads the value from
`pm.iterationData` and stores a properly-typed companion variable — see
`build_collection.py`'s `typed_var_prerequest()` if you're adding more.

Without an attached data file, these requests still run fine using the
default collection-variable value baked in (a single representative
case) — the data file just multiplies that one request across the full
value table.

## Rate limiting

The **5. Rate Limiting** folder holds one request meant to be run many
times, not once:

```bash
newman run NeevCloud-Model-API.postman_collection.json \
  -e NeevCloud-Model-API.postman_environment.json \
  --folder "5. Rate Limiting" \
  -n 20
```

or, in the GUI, run that folder via the Collection Runner with
**Iterations = 20** and **Delay = 0ms**. The test script only asserts the
*contract* — never a raw `5xx`, and any `429` is well-formed — since exact
limits are plan/tier-dependent and not part of the public API contract.

## Assumptions

1. **Base URL & auth scheme.** Taken directly from the published OpenAPI
   spec and developer guide: base URL
   `https://inference.ai.neevcloud.com`, auth via
   `Authorization: Bearer <api-key>` (key format `sk-nc-...`).
2. **Error format.** All documented error responses (`400`, `401`, `403`,
   `404`, `429`, `500`) follow the OpenAI-style
   `{"error": {"message", "type", "param", "code"}}` shape; tests validate
   against that schema rather than exact wording.
3. **`chat_model` is confirmed real** — `llama-3.1-8b-instant`, verified
   present via a live `GET /v1/models` call (catalog also included
   `glm-4-7`, `glm-5-2`, `deepseek-v3-2`, `kimi-k3`, `gpt-oss-120b/20b`,
   `minimax-m2.7/m3`, `llama-3.3-70b-versatile`).
4. **`tts_model`/`tts_voice` are NOT confirmed** — that same live catalog
   response contained zero audio/TTS entries, so there's no discoverable
   source for a valid id via `/v1/models`. Expect the Audio Speech
   folder's functional tests to fail until real values are supplied from
   the console's catalog UI or NeevCloud support.
5. **Cost-consciousness.** Functional tests deliberately use small
   `max_tokens` values and short prompts. The rate-limit/burst suite is
   opt-in and its burst size is small and configurable.
6. **Rate limiting is asserted as a contract, not a fixed number** —
   limits are plan/tier-dependent, so the test asserts "never a raw
   5xx, any 429 is well-formed" rather than a specific threshold.
7. **No UI/console automation.** The console at console.ai.neevcloud.com
   is where a human obtains the API key, but isn't exercised by this
   suite.

## Known findings from live runs against a real key

**Run 1:** mostly explained by `chat_model` being a placeholder OpenAI
name (`gpt-4o-mini`) not present in the account's catalog.

**Run 2** (after correcting `chat_model` to `llama-3.1-8b-instant`,
confirmed present via `GET /v1/models`): the model-name fix worked
exactly where expected — `GET /v1/models` and `GET /v1/models/{id}`
checks all passed. But **every single `POST /v1/chat/completions`
request failed with a 500**, including fully valid requests, and these
were fast failures (193–605ms), not timeouts. `POST /v1/audio/speech`
showed the same split: negative/validation requests 500, requests with a
real invalid model/voice correctly 400.

**Run 3** (after also fixing a real Postman bug — numeric fields like
`temperature`/`top_p`/`max_tokens`/`speed` were being sent as quoted JSON
strings instead of numbers, verified fixed with `newman --verbose`):
three additional assertions passed (`Nonexistent Model`, `Extra Unknown
Fields`, `tool_choice`), but the core problem was unchanged — nearly
every well-formed chat-completion request still 500s.

The GET-works / POST-inference-broken split, consistent across three
runs and unaffected by two independent client-side fixes, points away
from a test-data problem and toward one of:
- **Account provisioning** — KYC/wallet balance gating billable
  (inference) endpoints while catalog browsing stays open.
- **A platform-side incident** specific to the inference backend.
- **Missing `OrgID`/`ProjectID` headers**, if the key requires them.

Separately, these look like genuine API robustness gaps (500 instead of
the documented 4xx):
- `GET /v1/models/{id}` with a path-traversal/XSS/SQLi-style or empty id
- `POST /v1/models` (wrong HTTP method)
- `POST /v1/chat/completions` with a `model` id the account doesn't have

And `401` responses across endpoints have the right status code but
don't match the documented `{error: {message, type}}` shape.

The latest raw test-run export is kept at
[`results/latest-run.json`](./results/latest-run.json) for reference.


