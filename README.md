# local-rag

A complete, private retrieval-augmented-generation app that runs on your own machine.
Upload a PDF, ask a question, get an answer that is **only** built from passages of your
document — and shows you exactly which passages were used.

No cloud services, no API keys, no telemetry. Every model runs locally through
[Ollama](https://ollama.com), every vector lives in [Qdrant](https://qdrant.tech),
and the pipeline itself is an [n8n](https://n8n.io) workflow you can open and inspect
node by node.

```
  browser  ──POST /webhook/rag/ingest──▶  n8n ──▶ Extract PDF text
                                            │
                                            ├─▶ chunk (1 200 chars, 200 overlap)
                                            ├─▶ Ollama /api/embed   (nomic-embed-text, 768-d)
                                            └─▶ Qdrant  /collections/local_rag_chunks

  browser  ──POST /webhook/rag/query───▶  n8n ──▶ Ollama /api/embed   (question → vector)
                                            ├─▶ Qdrant  /points/search (top 5, score ≥ 0.25)
                                            ├─▶ Ollama /api/chat       (llama3.1 + context only)
                                            └─▶ answer + sources back to the browser
```

| Piece | What it is | Where it runs |
| --- | --- | --- |
| Frontend | 4 static files (HTML/CSS/JS), no build step | nginx container, port 8080 |
| Orchestration | 4 n8n workflows with real Code nodes | n8n container, port 5678 |
| Metadata store | n8n's own database — never SQLite | PostgreSQL 17 container, port 5432 |
| Vector store | 768-d Cosine vectors, payload indexes | Qdrant container, port 6333 |
| LLM | `llama3.1` (8B) via Ollama `/api/chat` | **host** Ollama, port 11434 |
| Embeddings | `nomic-embed-text` via Ollama `/api/embed` | **host** Ollama, port 11434 |

Ollama runs on the host (where your models already live); the containers reach it through
`host.docker.internal`. The n8n container talks to Ollama and Qdrant with plain
**HTTP Request** nodes — no credential nodes, no LangChain, fully transparent payloads.

---

## Requirements

* Fedora (or Debian/Ubuntu) with `systemd` — Docker is installed by the bootstrap script.
* [Ollama](https://ollama.com/download) already installed and running on the host.
* ~10 GB free disk (models + images + data), 8 GB+ RAM. **A GPU is not required** — it
  is just slower: an 8B model on CPU needs roughly 1–2 minutes per question.
* `python3` 3.9+ and `curl` on the host (everything else is stdlib or in a container).

## Quickstart

```bash
cd /path/to/local-rag

make init                 # .env with a random DB password and n8n encryption key
sudo make bootstrap       # ONE TIME: installs Docker, exposes host Ollama to containers
newgrp docker             # or log out and back in, so the docker group applies

make models               # pull llama3.1 + nomic-embed-text (~5 GB, once)
make up                   # start postgres, qdrant, n8n, frontend
make import               # import the 4 workflows and publish them
make qdrant               # create the local_rag_chunks collection + payload indexes
make sample               # write sample/sample.pdf used by the test
make test                 # end-to-end test: 20 checks over all 10 pipeline stages
make health               # status of every component

make migrate              # apply the temporal schema (idempotent)
make qdrant-btm           # create the btm_vectors collection
make test-backend         # unit checks: chunking, dates, extraction
make test-parsers         # all 9 file formats parse
make test-ingest          # live ingest contract: parse -> extract -> embed -> register
make test-n8n-ingest      # the same contract driven through the n8n BTM webhook
make seed-corpus          # 6 documents whose facts change over time
make test-retrieval       # as-of correctness: no future knowledge can leak in
make llm-probe            # which Groq models this key can actually use
```

`make seed-corpus` and `make test-retrieval` are a pair. The corpus is six
revisions of the same company's facts, so "what was the revenue in 2023?" has
one answer for 2023 and a different one for 2025. `test-retrieval` then asserts
both halves: the in-scope document is returned, and documents dated after the
window never are. A system that returns the 2025 report for a 2023 question has
not ranked badly, it has answered a different question.

Retrieval itself is hybrid. Keyword (Postgres full-text) and vector (Qdrant)
results are fused with Reciprocal Rank Fusion rather than a weighted sum of
scores, because BM25 and cosine similarity are not on comparable scales. A
small multiplicative recency term then breaks ties toward the newer in-scope
document, which is what makes "customer churn" return the current figure rather
than last year's. The temporal filter is applied inside both systems, before
anything is scored.

`make llm-probe` exists because model names rot: it lists what your key can reach and
runs the real extraction prompt against each candidate, so `GROQ_MODEL` is chosen by
measurement instead of guesswork.

`make import` is the command to remember after changing any workflow or Code file: it
regenerates the JSON, imports it, publishes each workflow (n8n 2.x separates drafts from
published versions), restarts n8n so the webhook paths register, and finally probes the
query webhook. It fails loudly if the webhook does not answer.

On n8n 2.x, PostgreSQL 17+ is required and `$env` access inside expressions must be
enabled (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`); both are handled by
`docker-compose.yml`. See [docs/troubleshooting.md](docs/troubleshooting.md) if a webhook
answers 404 after an import.

Then open:

| URL | What |
| --- | --- |
| <http://localhost:8080> | the app (upload, ask, library) |
| <http://localhost:5678> | n8n — open the `RAG 1 … 4` workflows and watch each run |
| <http://localhost:6333/dashboard> | Qdrant dashboard — inspect the indexed vectors |

`make bootstrap` is safe to re-run. It installs the distro Docker packages
(`docker-cli`, `moby-engine`, `docker-compose`), starts the daemon, adds your user to
the `docker` group, rebinds Ollama to `0.0.0.0:11434` via a systemd drop-in, and (on
Fedora) opens port 11434 to the Docker bridge networks only.

## The ten checks (what `make test` proves)

`python3 scripts/test_pipeline.py` uploads `sample/sample.pdf`, runs 20 assertions over
these ten stages and deletes the document again at the end (`--keep` leaves it indexed):

| # | Check | Evidence |
| --- | --- | --- |
| 1 | PDF accepted by the pipeline | `POST /webhook/rag/ingest` returns `status: indexed` |
| 2 | Text extracted | `extracted_chars > 0`, `page_count: 4` |
| 3 | Chunks created | `chunk_count > 0`, each with a page number |
| 4 | Embeddings created | point vector length 768, `payload.embedding_model: nomic-embed-text` |
| 5 | Vectors stored in Qdrant | `points_count` grows by `chunk_count` |
| 6 | Question embedded | query workflow completes, `usage.prompt_tokens > 0` |
| 7 | Relevant chunks retrieved | `sources[]` non-empty, each with a cosine `score` |
| 8 | Context handed to Llama 3.1 | `context_chars > 0` and the answer is non-empty |
| 9 | Answer is grounded | the answer contains a fact that exists **only** in the retrieved text (`18%`), a second question retrieves a different chunk, and an unanswerable question is refused |
| 10 | Index is persistent and cleanable | the document appears in `GET /webhook/rag/documents` and `DELETE /webhook/rag/document` removes it |

Check 9 is the important one: the sample document's facts (revenue `18%`, HQ
`Rotterdam`, founder `Elena Fischer`, `3 480` employees) were invented for this repo and
are not in any pretraining set, so a correct answer proves the model read *your* context
rather than reciting something it already knew.

## Using it

**Web UI** — drop a PDF on the upload card, wait for `Indexed`, then ask a question.
Answers list their sources; expand one to read the exact passage. The Library tab lists
every indexed document and deletes them.

**curl**

```bash
# 1. index a PDF (sha256 is optional but makes the document_id stable)
curl -sS -X POST http://localhost:5678/webhook/rag/ingest \
  -F "data=@sample/sample.pdf" | python3 -m json.tool

# 2. ask a question
curl -sS -X POST http://localhost:5678/webhook/rag/query \
  -H 'Content-Type: text/plain' \
  --data '{"question":"By how much did revenue increase during FY2025?"}' | python3 -m json.tool

# 3. ask about one document only
curl -sS -X POST http://localhost:5678/webhook/rag/query \
  -H 'Content-Type: text/plain' \
  --data '{"question":"Where is the company headquartered?","document_id":"doc_1a2b3c4d5e6f7890"}'

# 4. library and deletion
curl -sS http://localhost:5678/webhook/rag/documents | python3 -m json.tool
curl -sS -X DELETE "http://localhost:5678/webhook/rag/document?document_id=doc_1a2b3c4d5e6f7890"
```

## HTTP API

All four endpoints are n8n production webhooks (`/webhook/…`, no auth, localhost only).

### `POST /webhook/rag/ingest` — multipart

| Field | Required | Description |
| --- | --- | --- |
| `data` | yes | the PDF file |
| `sha256` | no | hex digest of the file; the browser computes it, so re-uploading the same bytes reuses the same `document_id` |

Example response (`sample/sample.pdf`):

```json
{
  "status": "indexed",
  "document_id": "doc_1a2b3c4d5e6f7890",
  "document_name": "sample.pdf",
  "sha256": "3f786850e387550fdab836ed7e6dc881de23001b…",
  "size_bytes": 2883,
  "page_count": 4,
  "extracted_chars": 1877,
  "chunk_count": 4,
  "vector_dim": 768,
  "embedding_model": "nomic-embed-text",
  "collection": "local_rag_chunks",
  "points_in_collection": 4,
  "duration_ms": 3120
}
```

Errors are explicit: `invalid_pdf` (400), `file_too_large` (400), `extraction_failed`
(422, password-protected or corrupt), `no_text_layer` (422, scanned/image-only PDF —
OCR it first), `no_chunks_produced` (422), `qdrant_upsert_failed` (502). Every error
body is `{ "status": "error", "error": { "code", "message", "hint" } }`.

### `POST /webhook/rag/query` — JSON

Send it as `text/plain` (what the browser does, which avoids a CORS preflight) or as
`application/json`.

| Field | Default | Description |
| --- | --- | --- |
| `question` | required | the question, ≤ `MAX_QUESTION_CHARS` |
| `top_k` | `TOP_K` (5) | 1–20 chunks to retrieve |
| `score_threshold` | `MIN_SCORE` (0.25) | per-request override of the cosine floor |
| `document_id` | – | restrict retrieval to a single document |

```json
{
  "status": "answered",
  "answer": "Revenue increased by 18% during FY2025… [1]",
  "sources": [
    {
      "rank": 1,
      "document": "sample.pdf",
      "page": 1,
      "chunk_id": "doc_1a2b3c4d5e6f7890-p1-c2",
      "score": 0.6123,
      "text": "Revenue increased by 18% during FY2025 to 412 million euro…"
    }
  ],
  "documents": ["sample.pdf"],
  "context_chars": 1180,
  "llm_model": "llama3.1:latest",
  "usage": { "prompt_tokens": 421, "completion_tokens": 88, "retrieval_ms": 34, "generation_ms": 51230 }
}
```

If nothing clears the score threshold the model is **not called at all** and you get
`{ "status": "no_context", "answer": "I could not find this information in the provided documents.", "hint": … }`.
That is deliberate: without it an 8B model happily answers from its own knowledge and
the citations become fiction.

### `GET /webhook/rag/documents`

```json
{
  "collection": "local_rag_chunks",
  "total_chunks": 4,
  "total_documents": 1,
  "documents": [
    { "document_id": "doc_…", "document_name": "sample.pdf", "chunk_count": 4,
      "page_count": 4, "page_known": true, "indexed_at": "2026-09-25T10:04:11.000Z" }
  ]
}
```

### `DELETE /webhook/rag/document?document_id=doc_…`

Deletes every chunk of that document with one filtered delete and returns
`{ "status": "deleted", "document_id": "doc_…" }`.

## Temporal queries

Three endpoints answer three different questions, all under the same no-future-leakage rule.

`POST /api/v1/query` — ranked evidence. Hybrid keyword and vector retrieval, RRF fusion, parent
expansion, with `as_of` applied in SQL and in Qdrant before scoring. Returns chunks, not prose.

Ranking runs in two stages. Fusion is good at recall and blind to *why* a passage matched, so a
reranking pass scores coverage (how much of the question it addresses), proximity (whether the
terms sit together), section-heading match, and density. All four are deterministic — no model, no
embedding call — so ranking works when the model does not. Every hit reports the features and the
base score that produced its position.

Recency is applied **only inside a relevance band** (`RERANK_RELEVANCE_BAND`, default 10%): where
relevance is genuinely tied, the newer document wins; past that band, relevance decides alone. The
previous behaviour multiplied the fused score by up to 1.15, which could promote a document ranked
8th by both retrievers above one ranked 1st by both — a 10.3% gap is smaller than a 15% bonus.
Set `RERANK_ENABLED=false` to restore it.

`POST /api/v1/answer` — a claim you can check. The same retrieval plus synthesis. Every citation
is validated against the retrieved set: a model citing `[7]` out of four supplied passages has
that citation discarded and the attempt reported in `rejected_citations`. Check `mode`:

| mode | meaning |
| --- | --- |
| `generated` | written by the model, every claim cited to a retrieved chunk |
| `extractive` | no model; passages quoted verbatim, so no number can be misstated |
| `unsupported` | the model answered without citing anything; the answer is demoted to a quote |
| `no_context` | nothing was known on that date, and it says so |

`GET /api/v1/state?as_of=2023-12-31` — what was known on a date. A bookmarkable snapshot: the
documents that existed, their passages, the revision chains they belong to, and every document
withheld with its reason. Built entirely from stored dates, so it is identical whether or not a
model is reachable. `excluded.not_yet_known` lists the later documents; they are never mixed
into the state. Snapshots report their own weaknesses in `warnings` — a corpus that has gone
quiet, a document included on upload date alone, an empty window that was empty rather than
broken.

`GET /api/v1/compare?from=2023-12-31&to=2025-12-31` — what changed. Both sides are reconstructed
with the same rules as `/state`, so the diff cannot contradict a snapshot. Metrics are matched by
name and unit and compared as levels only: `up 18 percent` is never diffed against a level.
Every change carries the sentence it came from on both sides, so each row can be checked against
the source in seconds.

| outcome | meaning |
| --- | --- |
| `changes` | stated in both windows with a different value — an actual change |
| `appeared` / `disappeared` | stated on one side only; a disappearance is missing evidence, not a retraction |
| `conflicts` | two documents **valid at the same time** disagree; no date filter can reconcile these |

Conflicting values are dated within `CONFLICT_WINDOW_DAYS` (default 30) of each other on purpose.
Three annual reports giving three revenues is a revision history, not a contradiction — reporting
that as a conflict would bury the one that matters. Undated statements are only compared with other
undated ones, since assuming they are contemporaneous would manufacture conflicts.

Metric extraction is deterministic pattern matching, not a model, so comparison works when the
model does not. It is a heuristic, which is why `method.verify` points at the source sentences.

```
make test-retrieval   # 14 checks: as-of correctness, no future leakage, hybrid contribution
make test-answer      #  7 checks: citation validity, abstention, no leakage in any mode
make test-state       # 10 checks: exclusions, revision chains, staleness, determinism
make test-compare     # 11 checks: change detection, revision-vs-conflict, retractions
```

## Configuration

Everything is in `.env` (created by `make init`; never commit it — it is in `.gitignore`
and `chmod 600`).

| Variable | Default | Change it when |
| --- | --- | --- |
| `LLM_MODEL` | `llama3.1` | you want faster answers (`llama3.2:3b`) or a different model |
| `EMBEDDING_MODEL` | `nomic-embed-text` | you switch embedding models — **recreate the collection afterwards** |
| `QDRANT_VECTOR_SIZE` | `768` | ditto; `make health` verifies it against the real model |
| `QDRANT_COLLECTION` | `local_rag_chunks` | you want a second, independent index |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | `1200` / `200` | retrieval quality is poor or contexts are too long — see [docs/chunking.md](docs/chunking.md) |
| `STRIP_REPEATED_MARGINS` | `true` | your PDFs should keep their headers/footers |
| `TOP_K` / `MIN_SCORE` | `5` / `0.25` | answers miss relevant passages (raise) or invent loosely related ones (raise) |
| `MAX_CONTEXT_CHARS` | `8000` | raise to give the model more evidence, lower to keep answers tight |
| `EMBED_BATCH_SIZE` / `QDRANT_UPSERT_BATCH` | `8` / `64` | chunks per `/api/embed` call, and points per Qdrant request. Lower both if a very large PDF runs out of memory |
| `NUM_CTX` / `MAX_TOKENS` | `8192` / `1200` | context overflow or truncated answers |
| `MAX_UPLOAD_MB` | `25` | larger PDFs; also raise `client_max_body_size` in `nginx/default.conf` |
| `OLLAMA_URL_INTERNAL` | `http://host.docker.internal:11434` | you run Ollama elsewhere (e.g. another host: `http://192.168.1.10:11434`) |
| `GENERIC_TIMEZONE` | `Asia/Kolkata` | — |

Two URLs exist for every service on purpose: `OLLAMA_URL`/`QDRANT_URL` for host
scripts, and `OLLAMA_URL_INTERNAL`/`QDRANT_URL_INTERNAL` for the n8n container (they
are injected into the container in `docker-compose.yml`).

After editing `.env`: `make restart` (the container only reads it at start). After
editing the prompt or the Code nodes: `make import` (regenerates and re-imports).

## Changing the prompt or the logic

* **Prompt** — edit `prompts/system_prompt.txt`, then `make import`. The builder injects
  it into the Query workflow as a JS string literal, so the file stays readable and
  diff-friendly instead of being buried in JSON.
* **Pipeline logic** — every Code node is a real file in `n8n/code/`, named
  `01_…`, `02_…`, `03_…` in execution order. Edit one, run `make import`, and the n8n
  canvas shows the same source you just edited.
* **Static checks** — `make validate` parses the workflow JSON, syntax-checks every JS
  and Python file, and verifies that each `$('Node Name')` and `$env.KEY` reference
  actually exists.

## Layout

```
local-rag/
├── docker-compose.yml          postgres, qdrant, n8n, nginx (+ named volumes)
├── .env / .env.example         configuration and secrets
├── Makefile                    every command you need
├── frontend/                   index.html, style.css, app.js, config.js
├── nginx/default.conf          static hosting, upload size limit
├── n8n/
│   ├── code/                   one .js per Code node — the real pipeline logic
│   └── workflows/              generated, importable workflow JSON
├── prompts/system_prompt.txt   the RAG system prompt
├── scripts/                    init, models, qdrant, import, healthcheck, test
├── docs/                       architecture, chunking, troubleshooting
└── sample/sample.pdf           4-page test document with known facts
```

## Day-to-day commands

| Command | What it does |
| --- | --- |
| `make up` / `make down` | start / stop the stack (data volumes are kept) |
| `make logs` | follow the n8n log — the place to look when a run fails |
| `make ps` | container status |
| `make health` | one report: Ollama, models, embedding size, Qdrant, collection, Postgres, n8n, container→Ollama path |
| `make test` | the 20 end-to-end checks over the ten stages (`ARGS=--keep` leaves the test document indexed) |
| `make workflows` | regenerate `n8n/workflows/*.json` only |
| `make import` | regenerate + import + publish + restart + probe the webhook |
| `make qdrant` / `make qdrant-reset` | create/verify the collection, or wipe and recreate it |
| `make validate` | offline static checks (no Docker needed) |
| `make clean` | **destructive**: removes containers *and* all volumes |

## Security notes

* Every published port is bound to `127.0.0.1` in `docker-compose.yml`, so nothing is
  reachable from the network and the only firewall rule added is Ollama's port for the
  Docker bridge ranges. `make validate` fails if a port is published on all interfaces.
  To use the app from another machine, remove the `127.0.0.1:` prefix and put a reverse
  proxy with TLS and authentication in front of it first.
* The webhooks have no authentication by design (they are meant for `localhost`).
  If you ever expose port 8080 or 5678, put a reverse proxy with TLS and auth in front
  of it, or restrict the n8n `N8N_ALLOWED_ORIGINS` value to your frontend origin.
* n8n runs with the onboarding flow disabled so the editor opens without an account —
  fine on a single-user machine, not fine on a shared one. Create an owner account if
  your n8n version asks for one.
* Document text is stored in Qdrant unencrypted at rest (inside a Docker volume) and
  sent to the local Ollama process. Keep the volume backups private.
* The system prompt tells the model to ignore instructions found inside retrieved
  documents, so a PDF that says "ignore your instructions" is treated as data.

## Operations

`docs/runbook.md` covers what to do when something breaks: the first commands to
run, why answers cite future documents, how the model being unavailable is meant
to look, how to rotate the Groq key, and what no check in this repository
actually covers.

The short version of the credential story: the Groq key exists only in `.env`
(mode `0600`, gitignored). It is in no committed file, no n8n workflow, and no
log — verified by scanning the tree for `gsk_`-prefixed strings. Rotation is a
one-line `sed` plus `docker compose up -d backend`, and a revoked key degrades
to quoted passages rather than an error page.

## Further reading

* [docs/architecture.md](docs/architecture.md) — every node, every payload, in order
* [docs/chunking.md](docs/chunking.md) — how text is split, and how to tune it
* [docs/troubleshooting.md](docs/troubleshooting.md) — symptom → cause → fix

### Operations and limitations

`docs/runbook.md` — failure modes, credential rotation, and an explicit list of
what the green suite does **not** prove.

### Frontend

Open `http://localhost:8080`. Two tab sets share one strip:

* **Ask / Library** are the original ingest-and-ask UI. They talk to n8n through `/webhook/rag/*`
  and are unchanged.
* **Time machine / Change** are the temporal views. They call the backend API directly
  (`frontend/time.js`), not the webhook layer.

The split is deliberate. `app.js` has a working contract that people depend on, so the temporal
features live in a separate file that cannot break it, and the two never share an endpoint. The
backend already allows cross-origin requests, so no proxy was added.

**Time machine** reconstructs the library as of any date. The part worth reading is *Withheld*:
documents dated after the query date are listed with the reason they were excluded, rather than
disappearing. A snapshot that silently drops a future document looks exactly like one that never
had it, which is the failure mode a time machine exists to avoid. Asking a question here sends
`as_of` with the request, so a later document cannot answer a question about an earlier date.

**Change** compares two dates. Changed, newly reported, no longer reported, and disagreeing are
kept separate, because they are different claims: a number changing is not a contradiction, and a
metric vanishing from the corpus is not a retraction. A changed row shows both source sentences
with their documents and dates.

The temporal panels show the fields they render from — base score and ranking features on evidence,
mode and provider on an answer, extraction method on a comparison. An answer you cannot trace is
not worth much.

Point the UI at a non-default backend with `?api=http://host:8000`, persisted in
`localStorage` under `local-rag-backend-url`.

## Evaluation

`make seed-corpus && make eval` scores the system over `backend/eval/dataset.json` and prints a
scorecard rather than a single verdict.

The corpus is built to be unsound in a useful way: every annual report restates and revises the
previous year's figures, so churn is 4.1, then 3.6, then 3.2, and headcount 180, 214, 260. That
makes "which document did this answer come from" a question with a right answer.

Nineteen cases across twelve dimensions. The distinction that matters:

| dimension | threshold | what it means |
|---|---|---|
| `temporal_leakage` | 1.0 | no document dated after the window reached the caller |
| `answer_leakage` | 1.0 | no citation dated after the window |
| `context_precision` | 1.0 | no forbidden figure in retrieved evidence |
| `context_recall` | 1.0 | the expected figure was in the evidence |
| `abstention` | 1.0 | the system declined when the corpus could not answer |
| `state_exact` | 1.0 | the reconstructed document set was exactly right |
| `compare` | 1.0 | extracted before/after values were right |
| `answer_value` | 0.8 | the generated answer carried the expected figure |

`/query` and `/answer` are scored separately. A wrong answer built on correctly retrieved evidence
is a generation problem; a right answer built on a future document is a temporal failure that looks
fine. Conflating them hides both, and the second is the one that matters.

The thresholds are deliberately asymmetric, and that should not be "tidied up". The model is
rate-limited; when it is, it says so and quotes the sources, which is correct and should not fail a
build. Quoting a figure from a document that did not exist yet is never acceptable, so those
dimensions are pinned at 1.0. Injecting a wrong expectation into the dataset fails the run with a
non-zero exit, so a green scorecard means something.

Figures are compared as parsed numbers, not substrings. `"36.5 million"` is not the figure 36, and
`"47.0 percent"` is the figure 47; matching on characters reported both as temporal faults when the
document was in fact in scope.

To add a case, append to the dataset. A useful one states a fact the corpus contradicts later, and
lists the later figure under `answer_excludes` - that is the assertion doing the work.
