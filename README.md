# DarijaDoc

**Darija-first medical guidance — a local test bench for symptom and prescription understanding.**

DarijaDoc turns a patient's own words — Moroccan Darija, French, Arabic, or a mix — into a
structured, safety-checked answer in their own language, and reads a photographed prescription
(ordonnance) back to them in plain terms. It is built as a local test bench: every response is
scored by the same validator the command-line harness uses, so the UI can never quietly disagree
with the tests.

> **Not a medical device.** DarijaDoc is a research and evaluation prototype. It does not
> diagnose, prescribe, or replace a doctor. Every response ends with a disclaimer, and the app
> never invents a dose that is not printed on the document.

---

## Why it exists

Patients in Morocco describe symptoms in Darija, but health advice is usually written in French
and standard Arabic. DarijaDoc closes that gap in both directions:

- **Understands** Darija, Arabic script, French and mixed input.
- **Answers** with a short explanation, 2–5 next steps, and an urgency level — in Latin Darija,
  Arabic script and French, so the patient can read and **hear** it.
2. Escalates real emergencies (with the 150 ambulance line first) and never invents advice.
3. Reads a photographed prescription without adding, rounding, or inferring a dose.

## Highlights

- **Darija-first dialogue** — free-text symptoms, scored against a 112-case accuracy suite.
- **Prescription reading** — a phone photo of an ordonnance becomes a structured, per-medication
  breakdown, always shown beside the source photo.
- **Voice output** — one continuous spoken clip per response via OpenAI TTS, with the browser
  `speechSynthesis` as fallback. Names which part is which language for correct pronunciation.
- **Pharmacy & hospital lookup** — real OpenStreetMap data through the keyless Overpass API, with
  a no-network manual city path so a demo never depends on geolocation permission.
- **Safety-first validator** — 11 server-side invariants (emergency routing, emergency number
  control, script and diacritics rules, TTS priority, and more) scored on every response.
- **Light & dark themes**, responsive layout, no build step.

## Project structure

```
darijadoc-ai/
├── serve.py                 # Local test UI + HTTP API + places backend
├── test_prompt.py           # Test harness, model calls, reference validator, CLI
├── make_vision_fixture.py   # Generates the synthetic ordonnance fixtures
├── requirements.txt         # openai, python-dotenv, Pillow
├── .env                     # OPENAI_API_KEY (gitignored — never commit)
│
├── prompts/
│   ├── system_prompt.txt    # The single source of truth for model behaviour
│   ├── response_schema.json # JSON Schema every response is validated against
│   ├── voice.js             # Dependency-free voice playback module
│   └── README.md            # The full AI contract + findings (read this too)
│
├── tests/
│   ├── test_inputs.json     # 112 labelled accuracy cases
│   ├── fixtures.json        # Ground truth for the ordonnance fixtures
│   └── voice.test.mjs       # 18 Node tests for the voice contract
│
├── web/                     # The local UI
│   ├── index.html           # App shell + SVG icon sprite
│   ├── style.css            # Design system (light + dark)
│   ├── app.js               # UI logic, quality check, places, rendering
│   ├── map.js               # Leaflet map + geolocation rules
│   └── vendor/leaflet/      # Vendored Leaflet (works offline)
│
└── outputs/                 # Per-case JSON saved by the harness
```

## Quickstart

Requires **Python 3.10+**. Node is optional and only needed for the voice tests.

```bash
# 1. Install
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 2. Add your key
echo 'OPENAI_API_KEY=sk-...' > .env

# 3. Run the UI  ->  http://127.0.0.1:8000
.venv/bin/python serve.py
```

The server binds to `127.0.0.1` only and refuses any other host: it has no auth and no rate
limit, so do not expose it. The API key stays in the server process and is never sent to the
browser.

## How it works

```
                    ┌─────────────────────────────────────────────┐
  type text  ─────► │  POST /api/ask   ─▶  test_prompt.ask_gpt()  │
                    │                                    │        │
  photo       ─────►│  POST /api/rx  -------------------->|        │
                    │                                    v        │
                    │            gpt-4o  (JSON mode)  ───┘        │
                    │                     │                       │
                    │              apply_sanitizer()              │
                    │             validate()  -> PASS/FAIL        │
                    └───────────────┬─────────────────────────────┘
                                    v
             explanation (Darija / Arabic / French)
             next_steps + next_steps_arabic
             urgency_level · emergency · location_required
             requires_pharmacy_lookup | prescription object
```

- **Mode is the routing key:** a typed question is `chat`; an attached photo is `explain`.
  `prescription` is non-null only in `explain`, and `requires_pharmacy_lookup` is true only
  there too.
- **Every response goes through the harness.** `serve.py` calls `test_prompt.ask_gpt` and scores
  with `test_prompt.validate`, so the UI cannot disagree with the CLI.
- **The model is never trusted on safety rules.** Eleven invariants (emergency ⇒ high urgency
  and location required, `next_steps[0]` carries the 150 call, no Arabic script in Latin fields,
> no invented hotline numbers, etc.) are asserted server-side after parsing.

### Text responses

`mode: "chat"`. Explains the likely concern in Darija, Arabic and French, and returns 2–5 next
steps. Emergency routing sets `urgency_level: "high"` and puts `150` (SAMU) as step one.

### Prescription photos

An attached image switches the request to `mode: "explain"`. The model reads the document into a
`prescription` object — doctor, date, and a list of medications with dose, form, frequency,
timing, duration, purpose and confidence — and never invents an unprinted dose.

Three safeguards the UI enforces (see `prompts/README.md` for why):

1. The **source photo is always pinned** beside the extraction.
- A **confirm-before-acting gate**: next steps and audio stay locked until the patient confirms
  they can read the name and dose on their own paper copy. Never gated in an emergency.
- A **pre-upload quality check** measures luma detail, brightness and resolution in-browser and
  warns before sending; the patient can always send anyway.

## HTTP API

All JSON. Started by `serve.py`.

| Method | Path | Body / params | Returns |
|---|---|---|---|
| GET  | `/api/health`    | — | `{ ok, api_key, model, tts, places }` |
| GET  | `/api/cities`    | — | 25 Moroccan cities with coordinates (manual fallback) |
| GET  | `/api/samples`   | — | The 112 labelled test cases |
| GET  | `/api/fixtures`  | — | The generated ordonnance fixtures |
| POST | `/api/ask`       | `{ text, id?, sanitize? }` | scored chat/emergency response |
| POST | `/api/rx`        | `{ image, filename? }` | ordonnance extraction + verdict |
| POST | `/api/tts`       | `{ text, instructions? }` | `audio/mpeg` clip |
| POST | `/api/pharmacies`| `{ city }` or `{ lat, lng }` | nearest pharmacies (OpenStreetMap) |
| POST | `/api/hospitals` | `{ city }` or `{ lat, lng }` | nearest hospitals (OpenStreetMap) |

```bash
curl localhost:8000/api/health
curl -X POST localhost:8000/api/ask -d '{"text":"3bandi wja3 f 9lb"}'
curl -X POST localhost:8000/api/pharmacies -d '{"city":"Casablanca"}'
```

## Testing & quality

```bash
# Full accuracy suite (112 cases). Add --quiet to skip the TTS preview.
.venv/bin/python test_prompt.py

# One case, or a one-off question
.venv/bin/python test_prompt.py --only t07
.venv/bin/python test_prompt.py --ask "3bandi wja3 f 9lb"

# Ordonnance extraction against ground truth (all 4 fixtures)
.venv/bin/python test_prompt.py --rx-suite

# Voice contract — no API key, no network, ~3s, 18 tests
node --test tests/voice.test.mjs
```

Every run writes `outputs/<id>.json`, so tone and phrasing can be reviewed by hand, not just
scored. The harness and UI share the same validator, so **if the UI says PASS, the CLI agrees.**

Known limits and measured data quirks are documented honestly in
[`prompts/README.md`](prompts/README.md) — including where photo extraction can still be
confidently wrong, and why the UI always pins the source photo beside it.

## Design

- A calm, clinical design system with **light and dark** themes from one token set.
- No CSS framework, no build step, no external fonts — it works on a bad connection.
- Leaflet is vendored and markers are inline SVG, so the map survives poor wifi and non-root
  serving.
- Accessibility touches: focus rings, reduced-motion support, ARIA roles on tabs and labelled
  controls.

## Security notes

- `.env` and `.venv/` are gitignored. **Never commit `OPENAI_API_KEY`.** Rotate any key that has
  been exposed.
- The server is local-only by design (`127.0.0.1`); do not deploy it as-is.
- Prescription photos are sent to OpenAI for extraction; treat them as sensitive patient data.

## Further reading

The full behavioural contract — prompt rules, response schema, finding-by-finding notes on the
places data and the prescription extraction, and the voice module's design — lives in
[`prompts/README.md`](prompts/README.md).

---

**DarijaDoc is a test bench, not a doctor.** `hadchi mojard char7, machi  doctor ola tbeb.`
