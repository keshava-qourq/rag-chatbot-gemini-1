<!-- sdlc:auto:start adr -->
# ADR-0001: Use React 19 for Frontend

Status: Proposed  
Date: 2026-10-27

## Context and Problem Statement

Which Frontend technology does the project use? Decided in architecture v1.

## Considered Options

- React 19
- Vue
- Svelte

## Decision Outcome

Chosen option: React 19, because This is the approved stack. It is built as a static SPA served from CloudFront with streaming fetch for chat.

## Consequences

SPA routing needs a CloudFront fallback to index.html.
<!-- sdlc:auto:end adr -->