# Project Aninag — Documentation Suite

Project Aninag is a cloud-native **Local Government Operations Platform** for the Philippines. It launches with a single module — **Citizen Incident Reporting** — built on an architecture designed from day one to scale into a full multi-tenant operating system for cities, municipalities, and eventually provincial and national government agencies.

This folder contains the enterprise documentation set for the platform. It covers the Core MVP scope: everything needed to take the Incident Reporting module from concept to a build-ready engineering plan, on a foundation that does not need to be re-architected as new modules, cities, or government tiers are added.

## Reading order

| #   | Document                                                         | Purpose                                                                                |
| --- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 1   | [Executive Summary](01-executive-summary.md)                     | The 5-minute pitch — problem, solution, market, ask                                    |
| 2   | [Vision & Product Strategy](02-vision-and-strategy.md)           | Why this exists, who it's for, how the platform grows over time                        |
| 3   | [Product Requirements Document (PRD)](03-prd.md)                 | MVP scope, personas, end-to-end workflow, success metrics                              |
| 4   | [Functional Requirements](04-functional-requirements.md)         | Numbered, testable system behaviors by actor and module                                |
| 5   | [Non-Functional Requirements](05-non-functional-requirements.md) | Performance, scalability, availability, security, compliance targets                   |
| 6   | [System Architecture](06-architecture.md)                        | Modular monolith design, multi-tenancy, workflow engine, tech stack rationale          |
| 7   | [Database Design & ER Model](07-database-design.md)              | Core schema, multi-tenant data model, PostGIS usage, indexing strategy                 |
| 8   | [API Specification](08-api-specification.md)                     | REST conventions, auth model, MVP endpoint catalog                                     |
| 9   | [Security Architecture](09-security.md)                          | AuthN/AuthZ, tenant isolation, data protection, Philippine Data Privacy Act compliance |
| 10  | [AI Design](10-ai-design.md)                                     | Azure OpenAI / AI Vision pipeline: validation, dedup, classification, prioritization   |
| 11  | [Roadmap](11-roadmap.md)                                         | Phased delivery plan from single-city MVP to nationwide platform                       |

## Explicitly out of scope for this pass

Per the source brief's [Documentation Strategy](../initial-project-aninag.md), the following were deferred until UI/UX and codebase decisions exist to document against — request them individually when ready:

- **User Stories** (derivable once wireframes exist — Functional Requirements cover the same ground at system level for now)
- **Business Rules** (partially covered inline in Functional Requirements and the Workflow Engine section of Architecture; a standalone rules catalog makes more sense once the Workflow Engine's rule DSL is designed)
- **Wireframes & Design System** (needs product design work, not just written spec)
- **Infrastructure & Deployment** (needs cloud subscription/environment decisions — Architecture doc covers target infra at a decision level)
- **Testing Strategy & Coding Standards** (most useful once the codebase exists to standardize against)

## Source

All documents trace back to the original founding brief: [`initial-project-aninag.md`](../initial-project-aninag.md).
