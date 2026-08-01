# CLAUDE.md

This file provides guidance to Claude Code when working in this repository. It complements `AGENTS.md`, which contains the full project context, conventions, and constraints shared across AI coding agents — **read `AGENTS.md` first**; this file only adds Claude-Code-specific operating notes.

## Quick Orientation

KostKonek: dual-channel (Telegram primary / WhatsApp `wa.me` fallback) rent-billing automation for Indonesian `kost` (boarding houses), fully bilingual (Bahasa Indonesia / English) across the admin dashboard, tenant invoice page, and every reminder message. Full docs: `PRD.md`, `ARCHITECTURE.md`, `DATABASE.md`, `API.md`, `DESIGN.md`, `TESTING.md`, `DEPLOYMENT.md`, `ROADMAP.md`.

## How to Work in This Repo

*   **Plan against `PRD.md`'s FR-xx numbering.** When asked to implement a feature, first check whether it maps to an existing FR. If it's genuinely new scope, propose adding an FR entry rather than silently building undocumented behavior.
*   **Treat `ARCHITECTURE.md` §3 (dual-channel notification flow) as load-bearing.** This is the part of the system that exists specifically to solve a real operational risk the user identified (WhatsApp link blocking). Do not simplify it back down to a single-channel design even if it looks like it would reduce code — that would reintroduce the exact fragility the design solves for.
*   **Docs are part of the deliverable, not an afterthought.** If a task changes the schema, an endpoint, or a core flow, update the corresponding `.md` file(s) in the same turn/commit as the code change. Stale docs are treated as a bug.
*   **Ask before touching payment or notification credentials/flow in ways not covered by the docs.** Midtrans and Telegram integrations involve real money and real external accounts (see `DEPLOYMENT.md`) — if a requested change isn't clearly specified, surface the ambiguity instead of guessing.

## Suggested Command Reference (once the codebase exists)

These assume a standard Next.js + Prisma setup per `ARCHITECTURE.md`; adjust if the actual `package.json` differs.

```bash
npm run dev                 # local dev server
npx prisma migrate dev      # local schema changes (never against production — see DEPLOYMENT.md)
npx prisma studio           # inspect local DB
npm run test                # unit + integration tests (Vitest)
npm run test:e2e            # Playwright E2E
npm run build                # production build (also run before considering a task "done")
```

## When Generating Code

*   Match the folder structure proposed in `ARCHITECTURE.md` §5 unless the actual repo has already diverged from it — in that case, follow the repo's real structure and flag the doc as needing an update.
*   Follow the API envelope and endpoint shapes already documented in `API.md` exactly; don't invent a different response shape for consistency's sake unless explicitly asked to refactor the contract (which also requires an `API.md` update).
*   Currency is integer Rupiah; dates/times are UTC in storage, formatted to `Asia/Jakarta` (WIB) for any user-facing display.
*   All tenant-facing copy (invoice page, Telegram/WhatsApp message templates, bot confirmation/error messages) must exist in **both** Bahasa Indonesia and English (see `messages/id.json` / `messages/en.json` and `lib/notifications/messageTemplates.ts`), matching the tone of the example strings already present in `API.md` and `ARCHITECTURE.md`. Never add a string in only one language.

## Testing Expectations for Claude Code

Before marking a task complete, run the test suite and, for any change touching payments or notifications, verify the specific scenario table rows in `TESTING.md` §3.1 or §3.2 relevant to the change are still (or newly) covered.
