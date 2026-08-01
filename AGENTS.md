# AGENTS.md

This file gives AI coding agents (Claude Code, Cursor, Copilot Workspace, etc.) the context needed to work productively on **KostKonek**, a `kost` (Indonesian boarding house) billing automation platform.

## Project Summary

KostKonek automates rent billing for `kost` owners: a real-time, color-coded room dashboard; frictionless (no-login) tenant invoice pages with QRIS payment; and **dual-channel** payment reminders — Telegram bot (primary) with automatic fallback to WhatsApp `wa.me` links — designed specifically so the owner is never blocked from reaching a tenant if one channel has an outage.

Read these docs, in this order, before making non-trivial changes:

1. `PRD.md` — product requirements, numbered FR-xx requirements are the source of truth for *what* to build.
2. `ARCHITECTURE.md` — system design, sequence diagrams for the notification and payment flows, folder structure.
3. `DATABASE.md` — schema (ERD + Prisma model). Any schema change must be reflected here and in an actual migration.
4. `API.md` — endpoint contracts (request/response shapes). Keep this in sync with actual route handlers.
5. `DESIGN.md` — visual/UX system (colors, typography, screen behavior).
6. `TESTING.md` — required test scenarios, especially §3.1 (dual-channel logic) and §3.2 (payment reconciliation).
7. `DEPLOYMENT.md` — env vars, Telegram/Midtrans setup, CI/deploy flow.
8. `ROADMAP.md` — what's Phase 1 (current scope) vs later phases; don't build Phase 2+ features unless asked.

## Tech Stack

Next.js (App Router) · TypeScript · Prisma · Supabase (Postgres + Realtime) · Tailwind CSS · Midtrans (QRIS) · Telegram Bot API · `next-intl` (bilingual ID/EN) · Vercel.

## Non-Negotiable Constraints

*   **No official WhatsApp Business API.** WhatsApp integration is limited to generating `wa.me` links; there is no WhatsApp send automation. Do not add `whatsapp-web.js`, Twilio WhatsApp, or similar — this is an explicit PRD constraint (cost/approval overhead).
*   **Telegram is free-tier, official Bot API only.** No paid Telegram features.
*   **Secrets never reach the client bundle.** `TELEGRAM_BOT_TOKEN`, `MIDTRANS_SERVER_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `JWT_SECRET` are server-only — reference them only inside `app/api/**/route.ts`, other server components, or `lib/` modules that are never imported into a `"use client"` file.
*   **Channel selection logic lives in one place**: `lib/notifications/channelSelector.ts` (see ARCHITECTURE.md §3). Don't scatter `if (telegramLinkStatus === 'LINKED')` checks across route handlers — call the shared selector so fallback/logging behavior stays consistent.
*   **All money is in Indonesian Rupiah (IDR), integer amounts (no fractional Rupiah).** Never introduce floating-point currency math; use integers (whole Rupiah) throughout, matching `DATABASE.md`'s `Invoice.amount` type.
*   **Tenant-facing pages require no auth.** Don't add a login wall to `/invoice/[invoiceId]`; access control there is the unguessable UUID in the URL, by design.
*   **No hardcoded, single-language UI strings.** Every user-facing string must go through `next-intl` (`messages/id.json` and `messages/en.json`), never inlined directly in JSX. If you add a string to one file, add its counterpart to the other in the same change — a missing translation key is treated as a bug, not a TODO for later.
*   **Reminder message language follows the tenant, not the admin.** Notification templates in `lib/notifications/messageTemplates.ts` are selected by `Tenant.preferredLocale` at send time — never by the sending admin's session locale (`User.preferredLocale`). These are two independent fields for a reason; don't collapse them.

## Conventions

*   API routes return `{ status: "success" | "error", data?: ..., message?: ... }` — match the shapes already documented in `API.md` rather than inventing a new envelope.
*   Prisma is the only DB access layer; no raw SQL except for narrowly justified performance cases, and even then wrap it in a `lib/` helper with a comment explaining why.
*   New DB fields/tables: update `DATABASE.md` (both the ERD mermaid block and the Prisma schema block) in the same change as the actual migration — the docs and schema must never drift.
*   New/changed endpoints: update `API.md` with the same request/response example style already used there (realistic-looking sample data, explicit status codes).
*   UI components should pull colors/spacing from the tokens defined in `DESIGN.md` §2–3, not ad-hoc hex values.

## Common Tasks & Where They Live

| Task | Primary files |
|:---|:---|
| Change reminder message copy | `lib/notifications/telegram.ts`, `lib/notifications/whatsapp.ts` |
| Add a new dashboard field/badge | `components/RoomCard.tsx` (or equivalent), `app/api/properties/[propertyId]/dashboard/route.ts` |
| Change Telegram linking token expiry | `lib/notifications/channelSelector.ts` or a dedicated `lib/telegram-linking.ts`, plus `DATABASE.md` if the expiry constant needs to be config-driven |
| Add a new invoice status transition | `app/api/webhooks/midtrans-payment/route.ts`, `DATABASE.md` (`InvoiceStatus` enum) |
| Add or change a UI string | `messages/id.json` **and** `messages/en.json` — both, in the same change |
| Change reminder message language logic | `lib/notifications/messageTemplates.ts` (keyed by `Tenant.preferredLocale`) |
| Add a test | See `TESTING.md` for which layer (unit/integration/E2E) fits the scenario |

## Before Opening a PR / Finishing a Task

1. Does the change match an existing `FR-xx` in `PRD.md`, or does `PRD.md` need a new entry? Docs and code should never silently diverge.
2. If you touched the schema or an API contract, did you update `DATABASE.md` / `API.md` in the same change?
3. Did you add/update the relevant scenario in `TESTING.md` if you touched payment or notification logic?
4. Run the test suite (see `TESTING.md` §5 for the CI sequence) before considering the task done.
