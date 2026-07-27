# Executive Summary

**Project Aninag** (working codename — *aninag* is Filipino for "translucence," the quality of being able to see through something clearly) is a cloud-native **Local Government Operations Platform** for the Philippines.

## The problem

Philippine LGUs — cities, municipalities, and their constituent barangays — run citizen-facing processes (incident reports, permits, clearances, inspections, appointments) on a patchwork of paper forms, Facebook pages, Excel trackers, and, at best, single-purpose apps built for one city that cannot be resold or extended. Citizens have no visibility into whether a report was received, who owns it, or when it will be resolved. LGUs have no structured data to manage SLAs, staff workload, or budget justification. There is no product in the Philippine market today that treats "LGU operations" as a single platform problem rather than a series of one-off procurement contracts.

## The solution

Project Aninag is built as a **multi-tenant operating system for LGUs**, not a single-purpose reporting app. The first shipped module — **Citizen Incident Reporting** — lets residents report issues (broken roads, flooding, garbage, fallen trees, broken streetlights, water leaks, fire, crime, and more) with photo and GPS evidence, have them AI-validated and de-duplicated, automatically routed to the correct barangay and department, and tracked through resolution with full audit history.

The differentiator is not the reporting form — competitors can build that in a sprint. It is the **underlying platform**: a configurable workflow engine, a multi-level government data model (Organization → City → Barangay → Department → Worker), and a multi-tenant architecture that lets a single deployment serve one city today and hundreds tomorrow, without a rewrite. Every future module (permits, business licensing, disaster response, inspections, payments, executive dashboards) is "just another workflow" running on the same engine.

## Why now

1. **Local Government Code decentralization** in the Philippines pushes service delivery to the barangay/city level, but tooling has not kept pace — most LGUs still run on manual processes.
2. **Smartphone and GPS penetration** among citizens is high enough that photo+GPS incident reporting is viable as a default channel, not a pilot.
3. **Azure OpenAI / Vision availability** in the region makes AI-assisted triage (spam detection, classification, priority scoring, duplicate detection) affordable at LGU budget scale, which was not true even three years ago.
4. **No incumbent owns the category.** Existing e-government vendors in the Philippines sell narrow, per-LGU custom systems, not a platform. A well-executed multi-tenant SaaS can win on total cost of ownership and speed of onboarding new cities.

## What "done" looks like for the MVP

A single city can be onboarded, its barangays and departments configured (no code changes), and citizens can report an incident that flows through AI validation → duplicate detection → GPS-based barangay routing → verification → worker assignment → before/after photo documentation → citizen confirmation → closure, with every step logged and visible to the citizen and to city administrators on a dashboard.

## Target customer & go-to-market shape

- **Primary buyer:** City/Municipal LGU (IT office or Mayor's office), procured as a SaaS subscription or GovTech pilot grant.
- **Primary user:** Citizens (guest or verified), Barangay staff, Department staff/heads, City Hall administrators.
- **Expansion path:** Land with one city → prove the incident reporting module → sell additional modules (permits, business licensing) into the same tenant → replicate to additional cities using the same deployment, each fully isolated and independently configurable.
- **Long-term ceiling:** Provincial and National government tiers layered on top of the same Organization → City → Barangay structure, without an architectural rewrite.

## What this document set establishes

The remaining documents in this suite ([Vision & Strategy](02-vision-and-strategy.md), [PRD](03-prd.md), [Functional](04-functional-requirements.md) and [Non-Functional Requirements](05-non-functional-requirements.md), [Architecture](06-architecture.md), [Database Design](07-database-design.md), [API Specification](08-api-specification.md), [Security](09-security.md), [AI Design](10-ai-design.md), and [Roadmap](11-roadmap.md)) define the MVP in enough detail to begin engineering, while explicitly designing every foundational layer (data model, workflow engine, tenancy, auth) to support the platform's full long-term scope without re-architecture.
