<!-- sdlc:auto:start adr -->
# ADR-0003: Use PostgreSQL 16 for Database

Status: Proposed  
Date: 2026-10-27

## Context and Problem Statement

Which Database technology does the project use? Decided in architecture v1.

## Considered Options

- PostgreSQL 16
- MySQL

## Decision Outcome

Chosen option: PostgreSQL 16, because This is the approved stack. pgvector provides vector search inside the same database, so no separate vector service is needed.

## Consequences

Filtered HNSW queries need iterative scan tuning.
<!-- sdlc:auto:end adr -->