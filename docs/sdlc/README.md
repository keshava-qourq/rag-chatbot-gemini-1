# RAG-chatbot-Gemini-1 — SDLC Documentation

> Maintained by QuorQ AI SDLC. Text between `sdlc:auto` markers is regenerated; anything outside them is yours and is kept.

<!-- sdlc:auto:start overview -->
RAG chatbot with chat history and session ID use gemini-2.5-flash for answers and gemini-embedding-001 for embeddings

**Documentation version:** v1
<!-- sdlc:auto:end overview -->

## Contents

<!-- sdlc:auto:start contents -->
| Document | State |
|---|---|
| [Requirements](requirements/README.md) | v1 approved |
| [Architecture](architecture/README.md) | 1 entry |
| [Development](development/README.md) | 1 entry |
| [Testing](testing/README.md) | 1 entry |
| [Deployment](deployment/README.md) | 1 entry |
| [Maintenance](maintenance/README.md) | 1 entry |
| [Data model](architecture/data-model.md) | generated |
| [API (OpenAPI)](architecture/api/openapi.yaml) | generated |
| [Decisions (ADRs)](architecture/adr/README.md) | 4 decisions |
| [Sprint plan](development/sprint-plan.md) | not generated yet |
| [Developer guide](development/developer-guide.md) | generated |
| [API reference](development/api-reference.md) | 1 route |
| [Test report](testing/test-report.md) | not run yet |
| [Tests](testing/test-cases.md) | 0 tests |
| [Deployment guide](deployment/deployment-guide.md) | not deployed yet |
| [Runbook](deployment/runbook.md) | not deployed yet |
| [Release notes](deployment/release-notes.md) | 0 releases |
| [Traceability matrix](traceability-matrix.md) | 29 requirements |
| [Changelog](CHANGELOG.md) | 1 version |
<!-- sdlc:auto:end contents -->

## Open gaps

<!-- sdlc:auto:start open-gaps -->
### Requirements

- No approval of the user stories or epics is recorded. Only the technology stack and services are stated as approved by the product owner.
- No date is given for requirements version 1.
- The frontend rationale names the document statuses 'processing -> indexed -> failed'. The stories (AC-027, AC-043, AC-045) use queued, processing, ready, failed. 'Indexed' and 'ready' need to be reconciled.
- The queue library is undecided: the input says 'RQ or Celery'.
- There is a possible bucket conflict. Textract is described as reading from the S3 bucket uploads already land in, yet a separate TEXTRACT_S3_BUCKET variable is required. It is unclear whether this is the same bucket as S3_BUCKET.
- SendGrid is approved for email verification, password reset and processing-complete notifications, but no user story or acceptance criterion covers any of these flows. Either the stories or the service list are incomplete.
- Redis is listed as container redis:7-alpine. The input does not say how it runs on AWS (e.g. a Fargate container or ElastiCache).
- REDIS_URL appears in the API key list and NFR-006, but it is not listed as an env variable on the Redis service entry.
- The 'stated minimum strength rules' for passwords are not defined.
- No email verification step is specified, although SendGrid was approved partly for email verification.
- The session validity period is not specified.
- No password reset story exists, although SendGrid was approved partly for password reset.
- AC-014 says the document appears in a 'queued/processing' state. The functional summary says 'queued'. The initial status should be confirmed.
- It is not stated whether '.markdown' or other Markdown extensions are accepted. Only '.md' is listed.
- The actual maximum file size value is not specified. It is only described as 'generous', and services mention 'multi-hundred-megabyte' scans.
- AC-025 describes serving the original file back to the user, but no story or screen exposes a 'download/open original' action. AC-089 also excludes an original-page viewer. It is unclear where AC-025 is used.
- AC-027 allows 'reloading more than once' to see the status change. AC-046 and the functional summary require in-place refresh without navigation. The expected refresh behaviour should be confirmed.
- The retry limit and backoff values are not specified. NFR-026 says they are configurable.
- The exact form of the 'location marker' for DOCX, TXT and Markdown (e.g. block index or line range) is not defined.
- The threshold for 'usable embedded text layer' is not defined.
- The chunk target size and overlap values are not specified. They are configurable per NFR-026.
- It is not specified how a passage that spans a page boundary records its page.
- The input does not say which failure reasons count as 'retryable'.
- AC-016 adds an upload-failure case that is not in AC-048's list of reasons. For that case there may be no stored S3 file to retry processing against.
- The functional summary says the record, passages, embeddings and S3 object are removed as one cascading transaction. NFR-008 limits the single transaction to the database portion. How S3 deletion is ordered or recovered relative to the DB transaction is unspecified.
- Old answers that cited a now-deleted document still show their original citations (AC-084, AC-088). It is ambiguous how this squares with AC-052's 'never cited afterwards'.
- The method for deriving a 'readable title' from the first question (e.g. truncation or LLM summary) is not specified.
- It is not specified whether an empty session appears in the chat list before the first message. AC-057 implies it does not.
- The input does not say which model or component performs the query rewrite.
- The number of recent turns is not specified. It is configurable per NFR-026.
- Full acceptance-criteria detail for US-023 to US-026 is not reproduced in this corrected response and must be re-documented from the source requirements.
- The top-k value is not specified. It is configurable per NFR-026.
- The full wording of AC-076 should be re-checked against the source, including any guidance it gives about widening the search.

### Architecture

- Architecture phase has not started or no input was provided.

### Development

- Development phase has not started or no input was provided.

### Testing

- Testing phase has not started.

### Deployment

- No deployment recorded yet.

### Maintenance

- No post-deployment maintenance activity recorded yet.
<!-- sdlc:auto:end open-gaps -->
