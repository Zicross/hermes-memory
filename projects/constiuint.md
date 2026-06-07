# ConstiuINT (CivicBridge)

## Purpose
Constituent intelligence + structured civic feedback infrastructure. Not a messaging app.

## Repo
`github.com/Zicross/civicbridge` (local: `/home/hermes/projects/civicbridge`)

## Stack
- Next.js + TypeScript + Postgres + Drizzle + Vitest + Playwright
- Mobile-first (phone app), NOT desktop-first

## Current Status
- MVP Plan 1: Foundation in progress
- Recent: Scaffold done, trust-core types in progress

## Key Files
- `planning/specs/2026-06-06-civicbridge-mvp.md` — MVP spec
- `planning/plans/2026-06-06-constiuint-mvp-plan1-foundation.md` — Implementation plan
- `planning/handoffs/` — Current handoffs

## Current Workflow (settled June 2026)
- User in Opus → dispatches Gemma 4 for implementation
- Opus reviews Gemma's output
- Hermes does strategic oversight only (NOT day-to-day coding)

## Product Guardrails
- No "send to representative" claims (conservative: "submit for review/triage")
- No donation/payment in MVP until compliance architecture specified
- Address matching minimizes retained sensitive data

## What Needs Attention
- Mobile-first requirement was missed initially — ensure future specs capture this upfront