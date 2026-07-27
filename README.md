# Knowledge Automation Platform — V1

Transforms existing software development knowledge (Jira stories,
engineering notes, design documents, metadata) into stakeholder-specific
representations — without generating any new knowledge, and without any
AI or external API calls. V1 is deterministic, template-driven, and
reads from a local dataset only.

## A note on the dataset

The prompt this project was built from describes a dataset at
`data/v1/NF-101 ... NF-120` as already existing, but no dataset was
provided along with it. To keep this a complete, runnable deliverable, a
**synthetic sample dataset for three issues (NF-101, NF-102, NF-103)**
was created for this V1, covering three different shapes of issue on
purpose:

- **NF-101** — a customer-facing feature (Story)
- **NF-102** — a customer-facing bug fix (Bug)
- **NF-103** — an internal, non-customer-facing technical task

This mix exercises every branch of the transformation business rules
(e.g. sales summaries and release notes behave differently for
non-customer-facing work). To extend the dataset to NF-104...NF-120,
just add more folders under `data/v1/` following the exact same
structure — the backend needs no code changes to pick them up.

## Project Structure

```
.
├── backend/
│   ├── api/
│   │   ├── routes.py            # FastAPI route definitions (thin)
│   │   └── schemas.py           # Request/response models (HTTP contract)
│   ├── connectors/
│   │   ├── base_connector.py    # BaseConnector interface + RawArtifacts
│   │   └── jira_connector.py    # V1: reads the local dataset
│   ├── services/
│   │   ├── context_builder.py       # Merges raw artifacts -> IssueContext
│   │   └── transformation_service.py# IssueContext -> requested representation
│   ├── models/
│   │   └── context.py           # Internal domain models (JiraStory, IssueMetadata, IssueContext)
│   ├── utils/
│   │   ├── exceptions.py        # Domain exception hierarchy
│   │   ├── logging_config.py
│   │   └── markdown_utils.py    # Shared markdown section-extraction helper
│   ├── config.py                # Paths, filenames, supported transformations
│   ├── main.py                  # App entrypoint + exception -> HTTP mapping
│   └── requirements.txt
└── data/
    └── v1/
        ├── NF-101/
        │   ├── jira_story.json
        │   ├── engineering_notes.md
        │   ├── design_document.md
        │   └── metadata.json
        ├── NF-102/ ...
        └── NF-103/ ...
```

## How data flows

```
JiraConnector          ContextBuilder            TransformationService
(reads data/v1/*)  →   (merges 4 files into  →   (IssueContext → one of 5
                        one IssueContext)          stakeholder-specific outputs)
```

Each component has exactly one job:

- **JiraConnector** only reads raw files for an issue key. It doesn't
  merge or interpret anything.
- **ContextBuilder** only merges the four raw artifacts into one
  validated `IssueContext`. It contains no business logic.
- **TransformationService** only turns a `IssueContext` into the
  requested representation, using deterministic templates and simple
  rules over fields like `issue_type`, `customer_facing`, `priority`.

## Installation

Requires Python 3.10+.

```bash
cd knowledge-automation-platform   # this project's root directory
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r backend/requirements.txt
```

## Running locally

Run from the project root (the directory containing both `backend/` and
`data/`):

```bash
uvicorn backend.main:app --reload
```

The API will be available at `http://127.0.0.1:8000`, with interactive
docs (Swagger UI) at `http://127.0.0.1:8000/docs`.

## API

### `GET /story/{issueKey}`

Loads an issue and returns its complete, merged Context.

```bash
curl http://127.0.0.1:8000/story/NF-101
```

### `POST /generate`

Generates a stakeholder-specific representation for an issue.

```bash
curl -X POST http://127.0.0.1:8000/generate \
  -H "Content-Type: application/json" \
  -d '{"issueKey": "NF-101", "transformation": "sales_summary"}'
```

Supported values for `transformation`:

- `sales_summary`
- `customer_release_notes`
- `support_guide`
- `faq`
- `internal_summary`

Response:

```json
{
  "success": true,
  "issueKey": "NF-101",
  "transformation": "sales_summary",
  "content": "..."
}
```

### `GET /health`

Basic liveness check, returns service name/version/status.

## Error handling

All errors are returned as JSON in a consistent envelope:

```json
{
  "success": false,
  "error": "issue_not_found",
  "detail": "Issue 'NF-999' was not found."
}
```

| Scenario                          | HTTP status |
|------------------------------------|:-----------:|
| Malformed issue key                | 400         |
| Unsupported transformation         | 400         |
| Request body fails validation      | 422         |
| Issue key not found in dataset     | 404         |
| Issue folder exists but a file is missing | 500  |
| Artifact file exists but is malformed (invalid JSON / wrong shape) | 500 |
| Any other unexpected error         | 500         |

## Extending to a real data source (V2+)

To replace the local dataset with a real Jira REST API, GitHub,
SharePoint, Confluence, or Azure DevOps source, implement
`BaseConnector` (see `backend/connectors/base_connector.py`) and wire the
new connector into `backend/api/routes.py` in place of `JiraConnector`.
`ContextBuilder` and `TransformationService` do not need to change, since
they only depend on the `RawArtifacts` / `IssueContext` shapes, not on
where the data came from.

## Out of scope for V1 (by design)

Authentication/authorization, a real database, Docker/Kubernetes, a
frontend, OAuth, LLM/AI integration, and connectors to real external
systems (Jira REST API, GitHub, SharePoint, Confluence, Azure DevOps),
background jobs, and caching. These are intentionally left for future
versions; V1 focuses on a clean, extensible foundation.
