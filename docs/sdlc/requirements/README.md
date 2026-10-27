# Product Requirements

> Maintained by QuorQ AI SDLC. Text between `sdlc:auto` markers is regenerated; anything outside them is yours and is kept.

<!-- sdlc:auto:start narrative -->
## Summary

### REQ-SCOPE · draft · source: requirements:scope

Version 1 of the requirements, at a lightweight delivery scope. The product is a RAG chatbot with chat history and session IDs. It uses gemini-2.5-flash for answers and gemini-embedding-001 for embeddings.

- Delivery scope: lightweight.
- Requirements version: 1.
- Project description: RAG chatbot with chat history and session ID. Uses gemini-2.5-flash for answers and gemini-embedding-001 for embeddings.
- Capabilities in scope:
  - Individual account sign-in; email and password sign-in.
  - Private per-user document library.
  - Document upload (PDF, DOCX, TXT, Markdown).
  - Scanned PDF text recognition (OCR); Amazon Textract reads scanned pages.
  - Background document processing; Redis queue runs processing in the background.
  - Document status list with failure state.
  - Document deletion with index cleanup.
  - Citation-ready passage tracking.
  - Multiple resumable chat sessions; saved chat history per session.
  - Streaming chat answers.
  - Search across the whole library by default; per-chat document filter.
  - Follow-up question understanding; last few chat turns sent with each question.
  - Answers grounded in uploaded documents; source citations on every answer.
  - Explicit 'cannot answer from your documents' reply.
  - Citation passage preview.
  - Library and recent chats dashboard.
  - Setup and usage guide.
  - Uploads stored privately in Amazon S3.
  - Document passages searched via pgvector in PostgreSQL.
  - Gemini embeds passages and writes answers.
  - Fixed generous file size cap, no quotas.
  - Desktop browser layout first.
- Out of scope:
  - Automated syncing or crawling of external content sources (Google Drive, Notion, S3, website crawl).
  - Spreadsheet/CSV ingestion with row- and column-aware tabular handling.
- Also excluded across the epics and NFRs:
  - Team workspaces, shared corpora and admin roles.
  - Voice input, summarisation and export.
  - Native mobile app, public API product and embeddable widget.
  - An in-app original-page viewer.

**Related:** EPIC-001, EPIC-002, EPIC-003, EPIC-004, EPIC-005, EPIC-006, EPIC-007

- ⚠ No approval of the user stories or epics is recorded. Only the technology stack and services are stated as approved by the product owner.
- ⚠ No date is given for requirements version 1.

### REQ-STACK · approved · source: requirements:technology-stack

The product owner approved the technology stack and said to design against exactly these choices: FastAPI 0.115, AWS, PostgreSQL 16 and React 19. Each choice comes with a stated rationale tied to the streaming, ingestion, isolation and UI needs.

- Backend: FastAPI 0.115.
  - Both hot paths are I/O-bound waits on Gemini: embedding a query, then streaming gemini-2.5-flash tokens.
  - Async handling plus StreamingResponse serves many concurrent streaming chats on modest hardware.
  - Python owns the ingestion toolchain: pypdf/PyMuPDF, python-docx, Markdown parsers and the google-genai SDK.
  - The API and the ingestion worker share one codebase, one chunking implementation and one provenance schema, which keeps page numbers consistent from parser to citation.
- Cloud: AWS.
  - S3 holds original uploads, which are needed to re-render a cited page.
  - RDS for PostgreSQL with the pgvector extension.
  - ECS Fargate runs the API and the ingestion worker as separate scalable services.
  - Amazon Textract covers scanned-PDF OCR.
  - Worker count scales on queue depth, independently of chat traffic.
- Database: PostgreSQL 16.
  - All entities are relational: users, documents with status, chat sessions, messages, and chunks with provenance.
  - Vectors live in the same database via pgvector.
  - Per-user isolation is a WHERE user_id = ? clause on the same query as the vector search.
  - The per-chat filter is AND document_id = ANY(...).
  - Deleting a document and its embeddings is one cascading transaction.
- Frontend: React 19.
  - Built for a streaming chat with inline expandable citations and a document library with live status changes.
  - The rationale names the statuses 'processing -> indexed -> failed'.
  - Uses react-markdown for answers and TanStack Query for polling.
  - A plain SPA is sufficient: desktop-first, no SEO need, no server rendering.

**Related:** NFR-015, NFR-016, NFR-025, EPIC-003, EPIC-006

- ⚠ The frontend rationale names the document statuses 'processing -> indexed -> failed'. The stories (AC-027, AC-043, AC-045) use queued, processing, ready, failed. 'Indexed' and 'ready' need to be reconciled.

### REQ-SERVICES · approved · source: requirements:services

The product owner approved six runtime services: pgvector, Google Gemini, Redis as a queue, Amazon S3, Amazon Textract and SendGrid. The input also lists the required API keys and environment variables.

- Vector store: pgvector in PostgreSQL (known).
  - Chunk vectors sit beside their provenance (document_id, title, page) and user_id.
  - Retrieval filters by user and optionally by selected documents in one indexed query.
  - Deleting a document removes rows and vectors atomically.
  - Vectors are stored at 1536 dimensions, not the 3072 default, so HNSW indexing is possible.
- AI provider: Google Gemini (SaaS, env GEMINI_API_KEY).
  - gemini-2.5-flash generates answers; its streaming API feeds the chat UI token by token.
  - Its large context window holds retrieved passages plus recent turns.
  - Grounding is enforced by the prompt and by refusing to answer below a retrieval score threshold, not by the model itself.
- Queue: Redis (container redis:7-alpine).
  - Runs asynchronous ingestion: parse -> OCR -> chunk -> embed -> index.
  - Uses RQ or Celery for retries and a dead-letter path, which makes the 'failed' state meaningful.
  - Also caches query embeddings.
- File storage: Amazon S3 (SaaS, env AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, S3_BUCKET).
  - Original uploads are retained to reach the source page and to allow re-processing.
  - Presigned URLs let browsers upload large files directly.
- OCR: Amazon Textract (custom, env AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, TEXTRACT_S3_BUCKET).
  - Uses the async DetectDocumentText job, reading from the S3 bucket uploads land in.
  - Returns per-page blocks with geometry.
  - Priced per page.
  - Tesseract was noted as a weaker free fallback.
- Email: SendGrid (SaaS, env SENDGRID_API_KEY).
  - Intended for email verification, password reset and processing-finished notifications.
  - Amazon SES was noted as a cheaper alternative inside AWS, but it requires a sending-limit review.
- API keys required: AWS_ACCESS_KEY_ID, AWS_REGION, AWS_SECRET_ACCESS_KEY, GEMINI_API_KEY, REDIS_URL, S3_BUCKET, SENDGRID_API_KEY, TEXTRACT_S3_BUCKET.

**Related:** NFR-006, NFR-009, NFR-016, EPIC-002, EPIC-003

- ⚠ The queue library is undecided: the input says 'RQ or Celery'.
- ⚠ There is a possible bucket conflict. Textract is described as reading from the S3 bucket uploads already land in, yet a separate TEXTRACT_S3_BUCKET variable is required. It is unclear whether this is the same bucket as S3_BUCKET.
- ⚠ SendGrid is approved for email verification, password reset and processing-complete notifications, but no user story or acceptance criterion covers any of these flows. Either the stories or the service list are incomplete.
- ⚠ Redis is listed as container redis:7-alpine. The input does not say how it runs on AWS (e.g. a Fargate container or ElastiCache).
- ⚠ REDIS_URL appears in the API key list and NFR-006, but it is not listed as an env variable on the Redis service entry.

### EPIC-001 · draft · source: requirements:EPIC-001

Accounts and Private Library Access. Individual email-and-password accounts, each with a private per-user document library and chat history. The product is single-user-per-account with no admin roles.

- Individual email/password accounts with a private library and private chat history.
- Assumption carried from discovery: single user per account. There are no team workspaces, shared corpora or admin roles.
- No specific compliance regime (HIPAA, GDPR residency, SOC 2) applies at this scope. Handling sensitive documents would change storage and retention requirements.
- Delivered as a web app targeting desktop browsers first.
- Stories: US-001, US-002, US-003, US-004.

**Related:** US-001, US-002, US-003, US-004, NFR-001, NFR-002, NFR-003, NFR-007, NFR-010

### US-001 · draft · source: requirements:US-001

Critical. Register an account with email and password. A new visitor creates an account to get a private space for documents and chats.

- Story: As a new visitor, I want to create an account using my email address and a password, so that I have a private space to upload my documents and keep my chats.
- AC-001: A valid email and a password meeting the stated minimum strength rules creates an account. An empty private library and chat list are provisioned, and the user is signed in and taken to the dashboard.
- AC-002: An already-registered email creates no second account. The error says the email is in use and links to sign in.
- AC-003: A malformed email or weak password is rejected with a field-level message. No account is created.
- AC-004: The password is persisted only as a salted one-way hash and is never retrievable or shown.

**Related:** EPIC-001, NFR-001, NFR-023, AC-001, AC-002, AC-003, AC-004

- ⚠ The 'stated minimum strength rules' for passwords are not defined.
- ⚠ No email verification step is specified, although SendGrid was approved partly for email verification.

### US-002 · draft · source: requirements:US-002

Critical. Sign in and sign out. A returning user signs in with email and password, and signs out to end the session securely.

- Story: As a returning user, I want to sign in with my email and password and sign out when finished, so that I can get back to my documents and conversations and leave the session securely.
- AC-005: Correct credentials establish an authenticated session. The user lands on a dashboard showing their own library and recent chats.
- AC-006: Incorrect email or password shows a single generic 'email or password is incorrect' message. The message does not reveal whether the email exists.
- AC-007: Sign out invalidates the session. Reopening any app page redirects to sign-in.
- AC-008: Returning within the session validity period after closing the browser keeps the user signed in. Documents and chats are exactly as left.

**Related:** EPIC-001, NFR-001, NFR-002, NFR-003, AC-005, AC-006, AC-007, AC-008

- ⚠ The session validity period is not specified.
- ⚠ No password reset story exists, although SendGrid was approved partly for password reset.

### US-003 · draft · source: requirements:US-003

Critical. Block access to application pages and data when not signed in. Every document, chat and answer requires authentication.

- Story: As a user, I want every document, chat and answer to require authentication, so that nobody can reach my content without my credentials.
- AC-009: Unauthenticated direct URLs for the dashboard, library, a chat session or a document redirect to sign-in. No content is rendered.
- AC-010: Unauthenticated requests to any endpoint that reads or writes documents, chunks or chat messages are rejected and return no data.

**Related:** EPIC-001, NFR-002, NFR-003, AC-009, AC-010

### US-004 · draft · source: requirements:US-004

Critical. Keep each user's library and chats isolated from other users. Retrieval and record access are constrained to the owning user.

- Story: As a user, I want my documents, passages and conversations to be visible and searchable only to me, so that I can trust that answers never draw on anyone else's content and mine is never exposed.
- AC-011: If a question's answer exists only in another user's documents, no passages from those documents are returned. The assistant replies that it cannot answer from the user's documents.
- AC-012: Requests for another account's document ID, chat session ID or citation are refused as not found or not permitted. No content or metadata is disclosed.
- AC-013: Library list, chat session list and retrieval queries are always constrained by the user identifier.
- Functional summary adds: the user_id constraint must be in the same SQL statement that fetches the data.

**Related:** EPIC-001, NFR-002, NFR-007, AC-011, AC-012, AC-013

### EPIC-002 · draft · source: requirements:EPIC-002

Document Upload and Private Storage. Users upload files from their own machine only, and files are stored privately in Amazon S3. A fixed, generous size cap applies, with no quotas.

- Content arrives only through uploads from the user's machine.
- Files are stored privately in Amazon S3.
- Supported types: text-based PDF, scanned/image PDF, .docx, .txt, Markdown.
- Out of scope:
  - Connectors or crawlers (Google Drive, Notion, S3 sync, website crawl).
  - Spreadsheet/CSV tabular ingestion.
- A fixed, generous file size cap applies. There are no per-user document counts, query volumes, billing or usage quotas.
- Stories: US-005, US-006, US-007, US-008.

**Related:** US-005, US-006, US-007, US-008, NFR-004, NFR-005, NFR-013

### US-005 · draft · source: requirements:US-005

Critical. Upload documents from my computer. One or more files can be selected or dragged in, and each appears immediately in the library with per-file progress.

- Story: As a signed-in user, I want to upload one or more files from my computer into my library, so that I can ask questions about their contents.
- AC-014: Selecting or dragging in supported files and confirming uploads each file. A document record is created with the filename as default title, and it appears immediately in a queued/processing state.
- AC-015: Per-file upload progress is shown. The rest of the app remains usable during transfer.
- AC-016: A network or storage error marks the document failed with an explanatory message and offers a retry.
- Functional summary adds: uploads go to S3 via application-issued presigned URLs.

**Related:** EPIC-002, NFR-004, NFR-013, NFR-019, AC-014, AC-015, AC-016

- ⚠ AC-014 says the document appears in a 'queued/processing' state. The functional summary says 'queued'. The initial status should be confirmed.

### US-006 · draft · source: requirements:US-006

High. Accept only supported file types. Unsupported files are rejected before transfer, and content mismatches are caught server-side.

- Story: As a user, I want unsupported files to be rejected clearly at upload time, so that I am not left waiting for processing that can never succeed.
- AC-017: .pdf, .docx, .txt or .md files with a matching content type are accepted.
- AC-018: Any other type, including .csv and .xlsx, is rejected before transfer. The message lists the supported formats, and no document record is created.
- AC-019: If a file passes the extension check but its bytes do not match a supported format, the server marks it failed as 'unsupported or corrupt file'. It does not enter the processing queue.

**Related:** EPIC-002, NFR-005, NFR-019, AC-017, AC-018, AC-019

- ⚠ It is not stated whether '.markdown' or other Markdown extensions are accepted. Only '.md' is listed.

### US-007 · draft · source: requirements:US-007

Medium. Enforce a generous fixed file size cap. The cap is shown up front, oversized files fail fast, and no quotas apply.

- Story: As a user, I want a clearly stated maximum file size, so that I know what I can upload and oversized files fail fast instead of stalling the pipeline.
- AC-020: The maximum file size and supported formats are stated on screen before a file is chosen.
- AC-021: An oversized file is rejected with a message giving its size and the allowed maximum. No document record or stored object is created.
- AC-022: No per-user document count, storage quota or query quota applies.

**Related:** EPIC-002, NFR-013, NFR-019, NFR-026, AC-020, AC-021, AC-022

- ⚠ The actual maximum file size value is not specified. It is only described as 'generous', and services mention 'multi-hundred-megabyte' scans.

### US-008 · draft · source: requirements:US-008

Critical. Store original files privately in Amazon S3. Objects are namespaced per user and document, and served only through short-lived signed URLs after an ownership check.

- Story: As a user, I want my original files held in private object storage that only the application can read on my behalf, so that my uploads are not publicly reachable.
- AC-023: The original is written to S3 under a key namespaced by user ID and document ID. The key is stored on the document record.
- AC-024: Fetching an object without application-issued, time-limited credentials is denied. There is no public or anonymous read.
- AC-025: Serving the original back uses a short-lived signed URL scoped to one object, issued only after an ownership check.

**Related:** EPIC-002, NFR-004, NFR-013, AC-023, AC-024, AC-025

- ⚠ AC-025 describes serving the original file back to the user, but no story or screen exposes a 'download/open original' action. AC-089 also excludes an original-page viewer. It is unclear where AC-025 is used.

### EPIC-003 · draft · source: requirements:EPIC-003

Asynchronous Document Processing Pipeline. A Redis-backed background pipeline parses files, runs OCR, chunks text with provenance, embeds and indexes it. A document is answerable only when ready.

- Background ingestion is driven by a Redis job queue.
- Pipeline steps:
  - Parse the file.
  - Run Amazon Textract OCR on scanned or image-only PDFs.
  - Chunk the text with provenance (document ID, title, page/location).
  - Embed chunks with gemini-embedding-001.
  - Index them into pgvector in PostgreSQL.
- A document is answerable only once indexing completes.
- Failures surface as a clear, actionable state.
- Stories: US-009 to US-013.

**Related:** US-009, US-010, US-011, US-012, US-013, NFR-005, NFR-015, NFR-016, NFR-017, NFR-025, NFR-027

### US-009 · draft · source: requirements:US-009

Critical. Queue ingestion work in the background after upload. Uploads return immediately, and documents are processed independently with idempotent retries.

- Story: As a user, I want processing to start automatically in the background once my file is uploaded, so that I can keep working instead of waiting on a blocked page.
- AC-026: After S3 storage and record creation, an ingestion job is enqueued on Redis. The HTTP response does not wait for parsing, OCR or embedding.
- AC-027: When a worker picks up the job, the status moves from queued to processing. The change is visible without reloading more than once.
- AC-028: Documents uploaded together are processed independently. One failure does not stop or fail the others.
- AC-029: After a worker crash or transient error, the job retries up to the configured limit without creating duplicate chunks. The document is marked failed only after retries are exhausted.
- Functional summary adds: exhausted jobs go to a dead-letter path.

**Related:** EPIC-003, NFR-005, NFR-015, NFR-017, NFR-018, NFR-027, AC-026, AC-027, AC-028, AC-029

- ⚠ AC-027 allows 'reloading more than once' to see the status change. AC-046 and the functional summary require in-place refresh without navigation. The expected refresh behaviour should be confirmed.
- ⚠ The retry limit and backoff values are not specified. NFR-026 says they are configurable.

### US-010 · draft · source: requirements:US-010

Critical. Extract text from text-based PDFs, DOCX, TXT and Markdown, with page numbers or location markers. Empty results fail clearly.

- Story: As a user, I want the system to read the text out of each supported document format, so that its content can be searched and quoted in answers.
- AC-030: For a text-based PDF, per-page text is extracted with page numbers.
- AC-031: For .docx, body text is extracted in reading order. Headings are kept as text, and a sequential location marker is recorded per block.
- AC-032: For .txt and Markdown, text is extracted with Markdown structure preserved as readable text. Location markers are recorded.
- AC-033: If no extractable text is found, the document is marked failed with 'no readable text found'. No empty chunks are indexed.

**Related:** EPIC-003, NFR-019, NFR-025, AC-030, AC-031, AC-032, AC-033

- ⚠ The exact form of the 'location marker' for DOCX, TXT and Markdown (e.g. block index or line range) is not defined.

### US-011 · draft · source: requirements:US-011

High. Read scanned and image-only PDFs with OCR. Pages without a text layer go to Amazon Textract, and page numbers are kept for citations.

- Story: As a user, I want scanned or photographed PDFs to be put through text recognition, so that I can ask questions about documents that contain no embedded text layer.
- AC-034: PDF pages with no usable embedded text, or empty extracted text, are routed to Amazon Textract instead of being discarded.
- AC-035: Recognised text is associated with the correct page number.
- AC-036: In mixed PDFs, embedded text is used where available and OCR only where needed. The combined text goes to chunking.
- AC-037: An OCR error or timeout after configured retries marks the document failed with 'text recognition failed' and offers a retry.
- Functional summary adds: uses the async DetectDocumentText job reading from the upload S3 bucket.

**Related:** EPIC-003, NFR-009, NFR-017, NFR-019, NFR-027, AC-034, AC-035, AC-036, AC-037

- ⚠ The threshold for 'usable embedded text layer' is not defined.

### US-012 · draft · source: requirements:US-012

Critical. Chunk extracted text while preserving provenance. Passages of a configured size with overlap each store owner, document, title, location, ordinal and text.

- Story: As a user, I want document text split into retrievable passages that each remember where they came from, so that every answer can cite a precise source location.
- AC-038: Passages have a configured target size with overlap. None is empty, and together they cover the whole document.
- AC-039: Each passage stores user ID, document ID, document title, page or location marker, ordinal position and text.
- AC-040: Stored title and page/location render citations without re-parsing the original.

**Related:** EPIC-003, NFR-025, NFR-026, AC-038, AC-039, AC-040

- ⚠ The chunk target size and overlap values are not specified. They are configurable per NFR-026.
- ⚠ It is not specified how a passage that spans a page boundary records its page.

### US-013 · draft · source: requirements:US-013

Critical. Embed and index passages for vector search. Passages are embedded with gemini-embedding-001 into pgvector, with resumable retries. A document becomes ready only when every passage is indexed.

- Story: As a user, I want my passages embedded and stored in a searchable vector index, so that questions can find the relevant parts of my documents.
- AC-041: Each passage is embedded with gemini-embedding-001. The vector is written to the pgvector table with provenance and user ID.
- AC-042: Rate-limit or transient errors retry with backoff up to the limit. Only passages still missing vectors are re-embedded.
- AC-043: When all passages are indexed, the status becomes ready and the document is immediately eligible for retrieval.
- AC-044: Queued, processing or failed documents are never retrieved or cited.
- Functional summary and services add: vectors are stored at 1536 dimensions and HNSW-indexed.

**Related:** EPIC-003, NFR-009, NFR-016, NFR-017, AC-041, AC-042, AC-043, AC-044

### EPIC-004 · draft · source: requirements:EPIC-004

Document Library Management. Users list their documents with live status, see failure reasons, retry, and delete with full index cleanup.

- Covers the assumption confirmed in discovery:
  - Users list uploaded documents with live processing status.
  - Failed documents show a clear failure state with a reason.
  - Users can delete a document so its chunks and embeddings are removed and it stops appearing in answers and citations.
- Stories: US-014, US-015, US-016.

**Related:** US-014, US-015, US-016, NFR-008, NFR-014, NFR-019

### US-014 · draft · source: requirements:US-014

High. See my documents with their processing status. The library list shows title, type, date and a status that updates live, plus an empty state.

- Story: As a signed-in user, I want a list of every document I have uploaded with its current status, so that I know what is ready to be asked about and what is still being prepared.
- AC-045: Each document shows title, file type, upload date and status (queued, processing, ready or failed).
- AC-046: Status updates in place when a job completes or fails. No sign-out or navigation is needed.
- AC-047: An empty library shows a first-upload prompt with the supported formats and size cap.
- Functional summary adds: refresh uses TanStack Query polling.

**Related:** EPIC-004, NFR-014, AC-045, AC-046, AC-047

### US-015 · draft · source: requirements:US-015

High. Understand and retry a failed document. Failed documents show a plain-language reason and can be reprocessed from the stored file.

- Story: As a user, I want a failed document to tell me why it failed and let me retry, so that I can fix or re-upload the file rather than silently losing content.
- AC-048: Failed documents are visibly marked and show a reason: unsupported or corrupt file, no readable text found, text recognition failed, or processing error.
- AC-049: For retryable failures, 'retry processing' enqueues a new job for the same stored file. The status returns to queued, and partial passages are discarded.
- AC-050: Failed documents are excluded from retrieval and never cited.

**Related:** EPIC-004, NFR-017, NFR-019, AC-048, AC-049, AC-050

- ⚠ The input does not say which failure reasons count as 'retryable'.
- ⚠ AC-016 adds an upload-failure case that is not in AC-048's list of reasons. For that case there may be no stored S3 file to retry processing against.

### US-016 · draft · source: requirements:US-016

High. Delete a document and purge it from the index. Confirmed deletion removes the record, passages, embeddings and the S3 original. It also cancels in-flight jobs and updates chat filters.

- Story: As a user, I want to delete a document and have everything derived from it removed, so that it can no longer influence or be cited in any answer.
- AC-051: Delete, after a confirmation prompt, removes the document record, all passages and embeddings in pgvector, and the S3 original.
- AC-052: Deleted documents are never retrieved or cited afterwards.
- AC-053: A deleted document is silently removed from any chat's filter selection. The chat continues against the remaining documents.
- AC-054: Deleting a queued or processing document cancels its job or discards results, so no orphaned passages remain.
- AC-055: Deleting a document ID that is not the requester's is refused, and nothing is deleted.

**Related:** EPIC-004, NFR-002, NFR-008, AC-051, AC-052, AC-053, AC-054, AC-055

- ⚠ The functional summary says the record, passages, embeddings and S3 object are removed as one cascading transaction. NFR-008 limits the single transaction to the database portion. How S3 deletion is ordered or recovered relative to the DB transaction is unspecified.
- ⚠ Old answers that cited a now-deleted document still show their original citations (AC-084, AC-088). It is ambiguous how this squares with AC-052's 'never cited afterwards'.

### EPIC-005 · draft · source: requirements:EPIC-005

Chat Sessions and Conversation History. Users have multiple resumable sessions, each with its own ID and saved history. Recent turns are used conversationally to resolve follow-ups.

- Multiple distinct, resumable chat sessions per account.
- Each session is identified by a session ID and keeps its own saved history.
- History is used conversationally, not merely displayed: the last few turns are sent with each question, and follow-ups are resolved before retrieval.
- Stories: US-017 to US-020.

**Related:** US-017, US-018, US-019, US-020, NFR-011, NFR-026

### US-017 · draft · source: requirements:US-017

Critical. Start a new chat session. A new session gets its own ID and is titled automatically from the first question.

- Story: As a signed-in user, I want to start a fresh conversation, so that I can keep separate lines of enquiry about my documents apart from each other.
- AC-056: 'New chat' creates a session with its own ID, owned by the user, with empty history and a default title.
- AC-057: The first message gives the session a readable title derived from that question. The session then appears in the chat list.
- AC-058: A new session has no messages from, and no effect on, other sessions.

**Related:** EPIC-005, AC-056, AC-057, AC-058

- ⚠ The method for deriving a 'readable title' from the first question (e.g. truncation or LLM summary) is not specified.
- ⚠ It is not specified whether an empty session appears in the chat list before the first message. AC-057 implies it does not.

### US-018 · draft · source: requirements:US-018

High. Resume a previous chat session. The chat list is ordered by most recent activity, and a reopened session restores full history with citations.

- Story: As a returning user, I want to reopen an earlier conversation exactly where I left it, so that I can carry on without repeating context.
- AC-059: The chat list shows each session's title and most recent message time, most recent first.
- AC-060: After signing out and back in, selecting a past session restores its full ordered history with citations. The user can continue the conversation.
- AC-061: Each question and answer, with citations, is persisted and survives reloads and later sign-ins.

**Related:** EPIC-005, US-024, AC-059, AC-060, AC-061

### US-019 · draft · source: requirements:US-019

Low. Rename and delete chat sessions. Owners can rename or delete sessions, and cross-account requests are refused.

- Story: As a user, I want to rename or delete my conversations, so that I can keep my chat list meaningful and discard ones I no longer need.
- AC-062: A rename is saved and shown in the chat list.
- AC-063: Delete, with confirmation, removes the session and its messages. Documents are untouched.
- AC-064: Open, rename or delete requests for another account's session are refused. No data is returned or changed.

**Related:** EPIC-005, NFR-002, AC-062, AC-063, AC-064

### US-020 · draft · source: requirements:US-020

High. Resolve follow-up questions against recent turns. Referential follow-ups are rewritten into self-contained queries before retrieval, and a bounded window of recent turns goes into the prompt.

- Story: As a user, I want follow-up questions that refer back to earlier messages to be understood, so that I can ask naturally without restating the full question each time.
- AC-065: A follow-up like 'what about the second one?' is rewritten into a self-contained query using recent turns before retrieval. Retrieved passages relate to the referenced item.
- AC-066: The last few turns are included in the gemini-2.5-flash prompt alongside the passages.
- AC-067: Only the configured number of recent turns is sent. Older turns stay stored and visible.
- AC-068: If the rewritten question finds no adequate passages, the assistant gives the cannot-answer reply, not an answer from memory or general knowledge.

**Related:** EPIC-005, US-026, NFR-011, NFR-026, AC-065, AC-066, AC-067, AC-068

- ⚠ The input does not say which model or component performs the query rewrite.
- ⚠ The number of recent turns is not specified. It is configurable per NFR-026.

### EPIC-006 · draft · source: requirements:EPIC-006

Retrieval Scope and Grounded Answering. Questions search the whole library by default, with an optional per-chat filter. Answers stream, are strictly grounded and always cited, with an explicit cannot-answer reply when nothing fits.

- Questions search the user's whole ready library by default. An optional per-chat filter narrows to selected documents.
- Retrieval uses gemini-embedding-001 query embeddings against pgvector.
- Answers come from gemini-2.5-flash, are streamed into a standard chat UI, are strictly grounded in retrieved passages, and always carry citations.
- When nothing adequate is retrieved, the assistant says so explicitly instead of using general knowledge.
- No voice input, summarisation or export.
- Stories: US-021 to US-026.

**Related:** US-021, US-022, US-023, US-024, US-025, US-026, NFR-007, NFR-011, NFR-012, NFR-020

- ⚠ Full acceptance-criteria detail for US-023 to US-026 is not reproduced in this corrected response and must be re-documented from the source requirements.

### US-021 · draft · source: requirements:US-021

Critical. Ask a question across my whole library. With no filter, a question searches all of the user's ready documents.

- Story: As a signed-in user, I want my question to search all of my ready documents by default, so that I get an answer without first having to pick which file it is in.
- AC-069: With no filter, the question is embedded with gemini-embedding-001 and top passages are retrieved from pgvector across all ready documents.
- AC-070: Only the user's own passages are considered. Queued, processing, failed and deleted documents are excluded.
- AC-071: With no ready documents, the reply says nothing is indexed yet and prompts an upload.
- Functional summary adds: retrieval is a single query filtered by user_id, and query embeddings are cached in Redis.

**Related:** EPIC-006, NFR-007, NFR-012, NFR-016, AC-069, AC-070, AC-071

- ⚠ The top-k value is not specified. It is configurable per NFR-026.

### US-022 · draft · source: requirements:US-022

Medium. Narrow a chat to selected documents. A per-chat filter restricts retrieval and citations to the chosen ready documents and never widens silently.

- Story: As a user with a large library, I want to restrict a conversation to the documents I choose, so that irrelevant material does not pollute the answers.
- AC-072: The filter lists ready documents. One or more can be selected.
- AC-073: With a filter active, retrieval and citations are limited to the selected documents.
- AC-074: The active filter and the selected count are visible. The filter can be cleared to return to whole-library search.
- AC-075: The filter is scoped per chat. Other sessions default to the whole library.
- AC-076: If the selected documents contain nothing relevant, the reply says it cannot answer from the selected documents.

**Related:** EPIC-006, US-016, NFR-007, NFR-012, AC-072, AC-073, AC-074, AC-075, AC-076

- ⚠ The full wording of AC-076 should be re-checked against the source, including any guidance it gives about widening the search.
<!-- sdlc:auto:end narrative -->

<!-- sdlc:auto:start summary -->
**Requirements version:** v1  
**Delivery scope:** lightweight  
**Approved:** 2026-10-27

**Capabilities in scope:** Individual account sign-in, Private per-user document library, Document upload (PDF, DOCX, TXT, Markdown), Scanned PDF text recognition (OCR), Background document processing, Document status list with failure state, Document deletion with index cleanup, Citation-ready passage tracking, Multiple resumable chat sessions, Saved chat history per session, Streaming chat answers, Search across whole library by default, Per-chat document filter, Follow-up question understanding, Answers grounded in uploaded documents, Source citations on every answer, Explicit 'cannot answer from your documents' reply, Citation passage preview, Library and recent chats dashboard, Setup and usage guide, Email and password sign-in, Uploads stored privately in Amazon S3, Amazon Textract reads scanned pages, Redis queue runs processing in background, Document passages searched via pgvector in PostgreSQL, Gemini embeds passages and writes answers, Last few chat turns sent with each question, Fixed generous file size cap, no quotas, Desktop browser layout first

**Approved technology stack:**

| Layer | Choice | Rationale |
|---|---|---|
| backend | FastAPI | Your two hot paths are both I/O-bound waits on Gemini — embedding a query, then streaming gemini-2.5-flash tokens back. FastAPI's async request handling plus StreamingResponse serves many concurrent streaming chats on modest hardware. Python also owns the ingestion toolchain you need (pypdf/PyMuPDF for page-accurate text extraction, python-docx, markdown parsers) and the google-genai SDK, so the API and the ingestion worker share one codebase, one chunking implementation and one provenance schema — critical when a page number has to survive from parser to citation. |
| cloud | AWS | The pieces your pipeline needs are all first-party here: S3 for original uploads (needed to re-render a cited page), RDS for PostgreSQL with pgvector available as an extension, ECS Fargate to run the API and the ingestion worker as separate scalable services, and Amazon Textract to solve the scanned-PDF OCR requirement that nothing else in your stack covers. Worker count can scale on queue depth independently of chat traffic, which matches your bursty ingest versus steady chat pattern. |
| database | PostgreSQL | Every entity you described — users, documents with status, chat sessions, messages, chunks with provenance — is relational, and the vectors can live in the same database via pgvector. That buys you two things you specifically need: per-user isolation becomes a plain `WHERE user_id = ?` on the same query as the vector search (no risk of a second system returning another user's chunk), and the per-chat document filter is just an extra `AND document_id = ANY(...)`. Deleting a document and its embeddings is one cascading transaction, so a deleted doc cannot linger in citations. |
| frontend | React | Your core screen is a streaming chat with inline citations that expand to show the supporting passage, plus a document library whose rows change status (processing → indexed → failed) while the user watches. That is stateful, interactive UI that React's component model and ecosystem (react-markdown for answer rendering, TanStack Query for polling ingestion status) handle directly. Desktop-browser-first with no SEO need means a plain SPA is sufficient — you don't need the server-rendering machinery of a framework. |
<!-- sdlc:auto:end summary -->

<!-- sdlc:auto:start epics -->
## EPIC-001: Accounts and Private Library Access

Individual email-and-password accounts with a private per-user document library and chat history. Assumption carried from discovery: this is a single-user-per-account product — there are no team workspaces, shared corpora, or admin roles. No specific compliance regime (HIPAA, GDPR residency, SOC 2) applies at this scope; sensitive-document handling would change storage and retention requirements. Delivered as a web application targeting desktop browsers first.

### US-001: Register an account with email and password

**Priority:** critical

> As a new visitor, I want to create an account using my email address and a password, so that I have a private space to upload my documents and keep my chats.

| ID | Given | When | Then |
|---|---|---|---|
| AC-001 | I am on the sign-up page and not signed in | I submit a valid email address and a password that meets the stated minimum strength rules | an account is created, an empty private library and chat list are provisioned for me, and I am signed in and taken to my dashboard |
| AC-002 | I am on the sign-up page | I submit an email address that is already registered | no second account is created and I see an error telling me the email is already in use with a link to sign in |
| AC-003 | I am on the sign-up page | I submit a malformed email or a password that fails the strength rules | the form is rejected with a field-level message explaining what is wrong and no account is created |
| AC-004 | an account has been created | the password is stored | it is persisted only as a salted one-way hash and is never retrievable or shown in plain text |

### US-002: Sign in and sign out

**Priority:** critical

> As a returning user, I want to sign in with my email and password and sign out when finished, so that I can get back to my documents and conversations and leave the session securely.

| ID | Given | When | Then |
|---|---|---|---|
| AC-005 | I have a registered account and am signed out | I submit my correct email and password | an authenticated session is established and I land on my dashboard showing my own library and recent chats |
| AC-006 | I am on the sign-in page | I submit an incorrect email or password | I am not signed in and I see a single generic 'email or password is incorrect' message that does not reveal whether the email exists |
| AC-007 | I am signed in | I choose sign out | my session is invalidated and any attempt to reopen an application page redirects me to the sign-in page |
| AC-008 | I signed in earlier and closed the browser | I return to the application within the session validity period | I am still signed in and my documents and chat sessions are exactly as I left them |

### US-003: Block access to application pages and data when not signed in

**Priority:** critical

> As a user, I want every document, chat and answer to require authentication, so that nobody can reach my content without my credentials.

| ID | Given | When | Then |
|---|---|---|---|
| AC-009 | I am not signed in | I open a direct URL for the dashboard, library, a chat session, or a document | I am redirected to the sign-in page and no content is rendered |
| AC-010 | I am not signed in | a request is made to any application endpoint that reads or writes documents, chunks, or chat messages | the request is rejected as unauthenticated and returns no data |

### US-004: Keep each user's library and chats isolated from other users

**Priority:** critical

> As a user, I want my documents, passages and conversations to be visible and searchable only to me, so that I can trust that answers never draw on anyone else's content and mine is never exposed.

| ID | Given | When | Then |
|---|---|---|---|
| AC-011 | two accounts each have uploaded documents | one user asks a question whose answer exists only in the other user's documents | retrieval returns no passages from the other user's documents and the assistant replies that it cannot answer from the user's documents |
| AC-012 | I am signed in | I request a document ID, chat session ID or citation that belongs to another account | the request is refused as not found or not permitted and no content or metadata is disclosed |
| AC-013 | I am signed in | my library list, chat session list, or retrieval query is executed | every query is constrained by my user identifier so only records owned by me are ever returned |

## EPIC-002: Document Upload and Private Storage

Users bring content exclusively by uploading files from their own machine; files are stored privately in Amazon S3. Supported types are text-based PDF, scanned/image PDF, .docx, .txt and Markdown. Automated connectors or crawlers (Google Drive, Notion, S3 sync, website crawl) and spreadsheet/CSV tabular ingestion are explicitly out of scope. A fixed, generous file size cap applies; there are no per-user document counts, query volumes, billing or usage quotas at this scope.

### US-005: Upload documents from my computer

**Priority:** critical

> As a signed-in user, I want to upload one or more files from my computer into my library, so that I can ask questions about their contents.

| ID | Given | When | Then |
|---|---|---|---|
| AC-014 | I am signed in and on the library or upload view | I select or drag in one or more supported files and confirm the upload | each file is uploaded, a document record is created in my library with its filename as the default title, and each appears immediately in my document list in a queued/processing state |
| AC-015 | an upload is in progress | the file is being transferred | I see per-file upload progress and can continue using the rest of the application |
| AC-016 | an upload fails because of a network or storage error | the transfer does not complete | the document is marked as failed with a message explaining the upload did not complete and I am offered the option to retry |

### US-006: Accept only supported file types

**Priority:** high

> As a user, I want unsupported files to be rejected clearly at upload time, so that I am not left waiting for processing that can never succeed.

| ID | Given | When | Then |
|---|---|---|---|
| AC-017 | I am on the upload control | I choose a file with a .pdf, .docx, .txt or .md extension and matching content type | the file is accepted for upload |
| AC-018 | I am on the upload control | I choose a file of any other type, including .csv and .xlsx spreadsheets | the upload is rejected before transfer with a message listing the supported formats, and no document record is created |
| AC-019 | a file passes extension checks but its actual content does not match a supported format | the server validates the uploaded bytes | the document is marked failed with an 'unsupported or corrupt file' reason rather than entering the processing queue |

### US-007: Enforce a generous fixed file size cap

**Priority:** medium

> As a user, I want a clearly stated maximum file size, so that I know what I can upload and oversized files fail fast instead of stalling the pipeline.

| ID | Given | When | Then |
|---|---|---|---|
| AC-020 | a fixed maximum file size is configured for the product | I view the upload control | the maximum file size and the supported formats are stated on screen before I choose a file |
| AC-021 | I select a file larger than the configured maximum | I attempt to upload it | the upload is rejected with a message giving the file's size and the allowed maximum, and no document record or stored object is created |
| AC-022 | I am within the size cap | I upload any number of documents or ask any number of questions | no per-user document count, storage quota or query quota blocks me, since no usage quotas or billing apply at this scope |

### US-008: Store original files privately in Amazon S3

**Priority:** critical

> As a user, I want my original files held in private object storage that only the application can read on my behalf, so that my uploads are not publicly reachable.

| ID | Given | When | Then |
|---|---|---|---|
| AC-023 | an upload completes | the original file is persisted | it is written to an Amazon S3 bucket under a key namespaced by my user identifier and document identifier, and the document record stores that key |
| AC-024 | a file is stored in S3 | anyone attempts to fetch the object URL without application-issued, time-limited credentials | access is denied because the bucket and objects allow no public or anonymous read |
| AC-025 | a stored document belongs to me | the application needs to serve the original file back to me | it issues a short-lived signed URL scoped to that single object only after confirming I own the document |

## EPIC-003: Asynchronous Document Processing Pipeline

Ingestion runs in the background after upload, driven by a Redis-backed job queue: parse the file, run Amazon Textract OCR on scanned or image-only PDFs, chunk the text while preserving provenance (document ID, title, page/location), embed chunks with gemini-embedding-001, and index them into pgvector in PostgreSQL. A document becomes answerable only when indexing completes; failures surface as a clear, actionable state.

### US-009: Queue ingestion work in the background after upload

**Priority:** critical

> As a user, I want processing to start automatically in the background once my file is uploaded, so that I can keep working instead of waiting on a blocked page.

| ID | Given | When | Then |
|---|---|---|---|
| AC-026 | a file has been stored in S3 and a document record created | the upload request completes | an ingestion job for that document is enqueued on the Redis queue and the HTTP response returns without waiting for parsing, OCR or embedding |
| AC-027 | an ingestion job is enqueued | a background worker picks it up | the document status moves from queued to processing and the change is visible in my library list without me reloading more than once |
| AC-028 | several documents are uploaded at once | the worker pool processes the queue | each document is processed independently and the failure of one does not stop or fail the others |
| AC-029 | a worker crashes mid-job or a transient external error occurs | the job is retried up to the configured retry limit | processing resumes without creating duplicate chunks for the document, and the document is marked failed only after retries are exhausted |

### US-010: Extract text from text-based PDFs, DOCX, TXT and Markdown

**Priority:** critical

> As a user, I want the system to read the text out of each supported document format, so that its content can be searched and quoted in answers.

| ID | Given | When | Then |
|---|---|---|---|
| AC-030 | a queued document is a text-based PDF | parsing runs | the text of each page is extracted together with its page number |
| AC-031 | a queued document is a .docx file | parsing runs | its body text is extracted in reading order with headings retained as text and a sequential location marker recorded for each extracted block |
| AC-032 | a queued document is a .txt or Markdown file | parsing runs | its text is extracted, Markdown structure is preserved as readable text, and location markers are recorded so passages can be cited |
| AC-033 | a document parses successfully but yields no extractable text at all | parsing completes | the document is marked failed with a reason stating no readable text was found, and no empty chunks are indexed |

### US-011: Read scanned and image-only PDFs with OCR

**Priority:** high

> As a user, I want scanned or photographed PDFs to be put through text recognition, so that I can ask questions about documents that contain no embedded text layer.

| ID | Given | When | Then |
|---|---|---|---|
| AC-034 | a queued PDF has no usable embedded text layer, or pages whose extracted text is empty | parsing runs | those pages are routed to Amazon Textract for text recognition instead of being discarded |
| AC-035 | Textract returns recognised text for a scanned page | the result is captured | the recognised text is associated with the correct page number so citations can point at that page |
| AC-036 | a PDF contains a mix of text pages and scanned pages | parsing runs | embedded text is used where available and OCR is used only for the pages that need it, and the combined text is passed on to chunking |
| AC-037 | the OCR service returns an error or times out after the configured retries | the job cannot complete | the document is marked failed with a reason indicating text recognition failed, and I am offered the option to retry processing |

### US-012: Chunk extracted text while preserving provenance

**Priority:** critical

> As a user, I want document text split into retrievable passages that each remember where they came from, so that every answer can cite a precise source location.

| ID | Given | When | Then |
|---|---|---|---|
| AC-038 | extracted text for a document | chunking runs | the text is split into passages of a configured target size with overlap, such that no passage is empty and the passages together cover the whole document |
| AC-039 | a passage is created | it is persisted | it stores the owning user ID, document ID, document title, page number or location marker, its ordinal position in the document, and its text |
| AC-040 | a passage has been indexed | it is later returned by retrieval | its stored document title and page/location are available to render a citation without re-parsing the original file |

### US-013: Embed and index passages for vector search

**Priority:** critical

> As a user, I want my passages embedded and stored in a searchable vector index, so that questions can find the relevant parts of my documents.

| ID | Given | When | Then |
|---|---|---|---|
| AC-041 | passages have been created for a document | embedding runs | each passage is embedded using gemini-embedding-001 and the resulting vector is written to the pgvector-backed table in PostgreSQL alongside the passage's provenance metadata and owning user ID |
| AC-042 | the embedding API returns a rate-limit or transient error | the job retries with backoff up to the configured limit | embedding resumes and only passages still missing vectors are re-embedded |
| AC-043 | all passages for a document have been embedded and indexed | indexing completes | the document status becomes ready and it immediately becomes eligible for retrieval in my chats |
| AC-044 | a document is still queued, processing or failed | I ask a question | none of its passages are retrieved or cited |

## EPIC-004: Document Library Management

Covers the assumption confirmed in discovery: users can list their uploaded documents with live processing status, see a clear failure state with a reason, and delete a document so that its chunks and embeddings are removed and it stops appearing in answers and citations.

### US-014: See my documents with their processing status

**Priority:** high

> As a signed-in user, I want a list of every document I have uploaded with its current status, so that I know what is ready to be asked about and what is still being prepared.

| ID | Given | When | Then |
|---|---|---|---|
| AC-045 | I have uploaded documents | I open my library | I see each document's title, file type, upload date, and a status of queued, processing, ready or failed |
| AC-046 | a document is processing while I am viewing the library | its background job completes or fails | the displayed status updates to the new state without me having to sign out or navigate away and back |
| AC-047 | I have uploaded no documents yet | I open my library | I see an empty-state message prompting me to upload my first document with the supported formats and size cap shown |

### US-015: Understand and retry a failed document

**Priority:** high

> As a user, I want a failed document to tell me why it failed and let me retry, so that I can fix or re-upload the file rather than silently losing content.

| ID | Given | When | Then |
|---|---|---|---|
| AC-048 | a document has status failed | I view it in my library | it is visibly marked as failed and shows a plain-language reason such as unsupported or corrupt file, no readable text found, text recognition failed, or processing error |
| AC-049 | a document failed for a retryable reason | I choose retry processing | a new ingestion job is enqueued for the same stored file, the status returns to queued, and any partial passages from the previous attempt are discarded |
| AC-050 | a document is in the failed state | I ask any question in any chat | it is excluded from retrieval and never appears as a citation |

### US-016: Delete a document and purge it from the index

**Priority:** high

> As a user, I want to delete a document and have everything derived from it removed, so that it can no longer influence or be cited in any answer.

| ID | Given | When | Then |
|---|---|---|---|
| AC-051 | I am viewing a document in my library | I choose delete and confirm in the confirmation prompt | the document record, all of its passages and embeddings in pgvector, and its stored original object in S3 are removed |
| AC-052 | a document has just been deleted | I ask a question whose answer previously came from that document | no passages from it are retrieved and it never appears as a citation |
| AC-053 | a deleted document was selected in a chat's document filter | I next open that chat | it is silently removed from the filter selection and the chat continues to work against my remaining documents |
| AC-054 | a document is still queued or processing | I delete it | its pending ingestion job is cancelled or its results are discarded on completion, so no orphaned passages remain |
| AC-055 | I attempt to delete a document ID that is not mine | the request is processed | it is refused and nothing is deleted |

## EPIC-005: Chat Sessions and Conversation History

Multiple distinct, resumable chat sessions per account, each identified by a session ID and retaining its own saved message history. Confirms the discovery assumption that history is used conversationally, not merely displayed: the last few turns are sent with each question so follow-ups that reference earlier turns are resolved before retrieval.

### US-017: Start a new chat session

**Priority:** critical

> As a signed-in user, I want to start a fresh conversation, so that I can keep separate lines of enquiry about my documents apart from each other.

| ID | Given | When | Then |
|---|---|---|---|
| AC-056 | I am signed in | I choose new chat | a chat session is created with its own session ID, owned by me, with an empty message history and a default title |
| AC-057 | a new chat session has been created | I send my first message | the session is given a readable title derived from that first question and appears in my chat list |
| AC-058 | I already have existing chat sessions | I start a new one | the new session has no messages from and no effect on any other session |

### US-018: Resume a previous chat session

**Priority:** high

> As a returning user, I want to reopen an earlier conversation exactly where I left it, so that I can carry on without repeating context.

| ID | Given | When | Then |
|---|---|---|---|
| AC-059 | I have previous chat sessions | I open my chat list | I see each session with its title and the time of its most recent message, ordered most recent first |
| AC-060 | I have signed out and signed back in since using it | I select a past session | its full message history, in order, with the citations shown on each previous answer, is restored and I can send a new message that continues the same conversation |
| AC-061 | I am viewing a chat session | a question is asked and answered | both my message and the assistant's answer with its citations are persisted to that session and survive page reloads and later sign-ins |

### US-019: Rename and delete chat sessions

**Priority:** low

> As a user, I want to rename or delete my conversations, so that I can keep my chat list meaningful and discard ones I no longer need.

| ID | Given | When | Then |
|---|---|---|---|
| AC-062 | I am viewing a chat session I own | I rename it | the new title is saved and shown in the chat list |
| AC-063 | I am viewing a chat session I own | I delete it and confirm | the session and all of its messages are removed from my chat list and can no longer be opened, while my documents remain untouched |
| AC-064 | I request to open, rename or delete a session ID belonging to another account | the request is processed | it is refused and no session data is returned or changed |

### US-020: Resolve follow-up questions against recent turns

**Priority:** high

> As a user, I want follow-up questions that refer back to earlier messages to be understood, so that I can ask naturally without restating the full question each time.

| ID | Given | When | Then |
|---|---|---|---|
| AC-065 | a chat where my previous question and the assistant's answer named several items | I ask a referential follow-up such as 'what about the second one?' | the question is rewritten into a self-contained query using the recent turns before retrieval runs, and the retrieved passages relate to the referenced item |
| AC-066 | any question in a chat session | the answer is generated | the last few turns of that session's history are included in the prompt sent to gemini-2.5-flash alongside the retrieved passages |
| AC-067 | a chat session with a long history | a new question is sent | only the most recent configured number of turns is included so the prompt stays within model limits, and older turns remain stored and visible in the transcript |
| AC-068 | a follow-up that is rewritten using history | retrieval returns no adequate passages for the rewritten question | the assistant still gives the explicit 'cannot answer from your documents' reply rather than answering from conversation memory or general knowledge |

## EPIC-006: Retrieval Scope and Grounded Answering

Questions search the user's whole ready library by default, with an optional per-chat filter narrowing to selected documents. Retrieval uses gemini-embedding-001 query embeddings against pgvector; answers are generated by gemini-2.5-flash, streamed into a standard chat interface, strictly grounded in retrieved passages, and always carry source citations. When retrieval surfaces nothing adequate, the assistant says so explicitly instead of using general knowledge. No voice input, summarisation or export features are in scope.

### US-021: Ask a question across my whole library

**Priority:** critical

> As a signed-in user, I want my question to search all of my ready documents by default, so that I get an answer without first having to pick which file it is in.

| ID | Given | When | Then |
|---|---|---|---|
| AC-069 | I have documents with status ready and no filter applied in this chat | I send a question | the question is embedded with gemini-embedding-001 and the top matching passages are retrieved from pgvector across all of my ready documents |
| AC-070 | retrieval runs for my question | candidate passages are selected | only passages owned by my account are considered, and passages from queued, processing, failed or deleted documents are excluded |
| AC-071 | my library is empty or has no ready documents | I send a question | I receive a message telling me there is nothing indexed yet to answer from, with a prompt to upload a document |

### US-022: Narrow a chat to selected documents

**Priority:** medium

> As a user with a large library, I want to restrict a conversation to the documents I choose, so that irrelevant material does not pollute the answers.

| ID | Given | When | Then |
|---|---|---|---|
| AC-072 | I am in a chat session | I open the document filter | I see my ready documents and can select one or more to scope this chat to |
| AC-073 | I have selected documents in a chat's filter | I send a question | retrieval considers passages only from the selected documents and citations can only reference those documents |
| AC-074 | a filter is active in a chat | I view the chat | the active filter and the number of selected documents are visible, and I can clear it to return to searching my whole library |
| AC-075 | I set a filter in one chat session | I open or start a different chat session | that other session is unaffected and defaults to searching my whole library, because the filter is scoped per chat |
| AC-076 | a filter is active and the selected documents contain nothing relevant | I send a question | the assistant replies that it cannot answer from the selected documents rather than silently widening the search |

### US-023: Receive a streaming answer grounded only in my documents

**Priority:** critical

> As a user, I want the answer to be written only from the retrieved passages and to appear progressively as it is generated, so that I can trust what it says and start reading straight away.

| ID | Given | When | Then |
|---|---|---|---|
| AC-077 | relevant passages have been retrieved for my question | the answer is generated | gemini-2.5-flash is prompted with those passages plus recent chat turns and instructed to answer solely from the supplied passages |
| AC-078 | an answer is being generated | tokens are produced | the answer text streams into the chat view incrementally and a visible indicator shows while retrieval and generation are still in progress |
| AC-079 | the retrieved passages do not contain a fact | the answer is generated | the answer does not assert that fact from the model's general knowledge and instead states what the documents do and do not cover |
| AC-080 | an answer has finished streaming | the turn completes | the full answer text and its citation set are persisted to the chat session |
| AC-081 | the generation or retrieval call fails or times out | the turn cannot complete | I see an error message in the chat explaining the answer could not be produced and offering to retry, and no partial answer is saved as if it were complete |

### US-024: See source citations on every answer

**Priority:** critical

> As a user, I want each answer to show which of my documents it drew on and where, so that I can verify the claim against the original.

| ID | Given | When | Then |
|---|---|---|---|
| AC-082 | an answer has been generated from retrieved passages | it is displayed | it lists the source documents it drew on, each citation showing the document title and the page number or location of the supporting passage |
| AC-083 | an answer draws on several passages from several documents | citations are rendered | every distinct source document used is represented and duplicate citations to the same passage are collapsed |
| AC-084 | I reopen an old chat session | I view a past answer | its citations are shown exactly as they were when the answer was produced |
| AC-085 | an answer that states it cannot answer from my documents | it is displayed | no citations are attached to it |

### US-025: Preview the passage behind a citation

**Priority:** high

> As a user, I want to click a citation and read the exact passage that supported the answer, so that I can check the answer without hunting through the original file.

| ID | Given | When | Then |
|---|---|---|---|
| AC-086 | an answer with citations | I click a citation | a panel opens showing the full text of the supporting passage together with its document title and page or location |
| AC-087 | a citation passage preview is open | I close it | I return to the conversation with my scroll position and the answer intact |
| AC-088 | the source document has since been deleted | I click a citation on an older answer | I am told the source document is no longer in my library instead of seeing an error |
| AC-089 | the open question of whether citations link into a full in-app document viewer showing the original page | this version is delivered | only the passage preview is provided and no original-page viewer is included, since that decision is undecided and deferred |

### US-026: Be told explicitly when my documents cannot answer

**Priority:** critical

> As a user, I want the assistant to say plainly when it has nothing relevant in my documents, so that I am never misled by an answer invented from general knowledge.

| ID | Given | When | Then |
|---|---|---|---|
| AC-090 | I ask a question whose subject appears nowhere in my library | retrieval returns no passages above the configured relevance threshold | the assistant replies with an explicit statement that it cannot answer from my documents, and suggests uploading relevant material or clearing the document filter |
| AC-091 | retrieval returns passages but none actually support an answer | generation runs | the model is instructed to return the explicit cannot-answer response rather than speculating, and that response is what is shown and saved |
| AC-092 | a question is about well-known general knowledge not present in my documents | I ask it | the assistant does not answer it from the model's own knowledge and instead gives the cannot-answer response |

## EPIC-007: Dashboard, Onboarding and Delivery

The entry point after sign-in and the guidance that makes the product usable. Delivered as a desktop-browser-first web application, not a mobile app, API-only service or embeddable widget.

### US-027: See my library and recent chats on a dashboard

**Priority:** medium

> As a signed-in user, I want a landing view showing my documents and my recent conversations, so that I can pick up where I left off or add new material in one place.

| ID | Given | When | Then |
|---|---|---|---|
| AC-093 | I have signed in | I land on the dashboard | I see my recent chat sessions with their titles and last activity, and a summary of my library including documents currently processing and any failures |
| AC-094 | I am on the dashboard | I choose an action | I can start a new chat, open an existing chat, upload a document, or go to the full library list from that single view |
| AC-095 | I am brand new with no documents and no chats | I land on the dashboard | I see a first-run state guiding me to upload a document before asking a question |

### US-028: Follow a setup and usage guide

**Priority:** medium

> As a new user or operator, I want documentation covering how to set the system up and how to use it, so that I can run it and get value from it without asking the team.

| ID | Given | When | Then |
|---|---|---|---|
| AC-096 | the delivered product | I read the setup guide | it lists the required configuration — Gemini API credentials for gemini-2.5-flash and gemini-embedding-001, the S3 bucket, the Redis queue, the PostgreSQL database with the pgvector extension, and Amazon Textract access — and the steps to run the web application and the background workers |
| AC-097 | I am a new end user | I read the usage guide | it explains signing up, uploading documents, what the processing statuses mean, how to ask questions, how the per-chat document filter works, and how to read citations and the cannot-answer reply |
| AC-098 | the usage guide | I look for the product's boundaries | it states the supported file formats, the file size cap, and that external source connectors and spreadsheet/CSV tabular ingestion are not supported |

### US-029: Use the application comfortably in a desktop browser

**Priority:** medium

> As a user, I want a layout designed for a desktop browser window, so that the chat, citations and library are all usable at the screen size I work on.

| ID | Given | When | Then |
|---|---|---|---|
| AC-099 | I open the application in a current desktop browser | I use the chat, library and citation preview | the layout renders without horizontal scrolling or overlapping controls at typical desktop widths |
| AC-100 | a long conversation or a long document list | I scroll | content loads and scrolls smoothly and the message input and primary navigation remain reachable |
| AC-101 | this scope targets desktop browsers first | the product is delivered | no native mobile application, public API product, or embeddable chat widget is included |
<!-- sdlc:auto:end epics -->
