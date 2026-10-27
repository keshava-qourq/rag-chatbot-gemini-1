<!-- sdlc:auto:start adr -->
# ADR-0002: Use FastAPI 0.115 for Backend

Status: Proposed  
Date: 2026-10-27

## Context and Problem Statement

Which Backend technology does the project use? Decided in architecture v1.

## Considered Options

- FastAPI 0.115
- Django
- Express

## Decision Outcome

Chosen option: FastAPI 0.115, because This is the approved stack. Async StreamingResponse fits NDJSON chat, and the worker shares the same codebase.

## Consequences

A separate worker process is needed for long ingestion jobs.
<!-- sdlc:auto:end adr -->