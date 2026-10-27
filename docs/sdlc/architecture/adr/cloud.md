<!-- sdlc:auto:start adr -->
# ADR-0004: Use AWS for Cloud

Status: Proposed  
Date: 2026-10-27

## Context and Problem Statement

Which Cloud technology does the project use? Decided in architecture v1.

## Considered Options

- AWS
- GCP
- Azure

## Decision Outcome

Chosen option: AWS, because This is the approved stack. The design uses ECS Fargate, RDS, S3, CloudFront and Textract.

## Consequences

Gemini is an external SaaS call outside AWS.
<!-- sdlc:auto:end adr -->