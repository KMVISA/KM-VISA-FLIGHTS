# CLAUDE.md — KM-VISA-FLIGHTS

Read this file before any work in this repository.

## Project purpose
Private, single-owner flight search and comparison tool for KM VISA (Annaba, Algeria).
Phase 1 scope is **flight search and comparison only**.

## Out of scope for Phase 1 (do not build, do not scaffold)
- Booking or ticketing of any kind
- Payments
- Visa file tracking
- PDF itinerary generation, passenger database, ticket archive
- Multi-tenant / SaaS features

Any feature outside Phase 1 requires explicit owner approval first.

## Language rules
- The owner explains and reviews in **Arabic**. All summaries, warnings and questions to the owner are in Arabic.
- Everything executable or technical is in **English** (or French if English is impossible): terminal commands, code, file names, `CLAUDE.md`, schema, variable names, commit messages.

## Technology stack
- Next.js (App Router) with TypeScript (strict mode)
- Tailwind CSS
- Supabase (PostgreSQL, Auth, Row Level Security)
- GitHub (source control, private repository)
- Vercel (hosting)

Use current, stable versions. Consult official documentation before using any API.

## Architecture rules
- Server-side code (route handlers, server actions) calls external flight APIs. Browser code never calls flight APIs directly.
- Business logic lives in `lib/` (for example `lib/flights/`), not inside UI components.
- Data validation runs on both client and server (use a schema library such as Zod).
- Flight-data provider access sits behind one adapter interface, so the provider can change without touching the UI.

## Security rules (mandatory)
- Secrets are stored in **Bitwarden** (owner's password manager). Never write secret values in chat, in code, or in documentation.
- Local secrets go in `.env.local`. This file is **never committed**. It must be listed in `.gitignore`.
- `.env.example` is committed. It lists variable names only, with empty values.
- Production secrets are set only in the **Vercel dashboard** (Environment Variables), never in the repository.
- The Supabase `service_role` key is **server-only**. It must never appear in client code, in any `NEXT_PUBLIC_` variable, or in browser bundles.
- Only the Supabase anon/publishable key may be used in browser code, and only together with Row Level Security enabled on every table.
- Every table in `public` has RLS enabled. Policies are part of each migration.
- Do not rely on frontend checks alone for access control.

## Database rules
- Schema is defined in `docs/DATABASE-SCHEMA.md`. Changes to the schema update that document first.
- All schema changes go through versioned migration files in `supabase/migrations/`. No manual edits in the dashboard.
- Use explicit primary keys (UUID), foreign keys, unique constraints, and indexes on columns used in searches.
- Store all monetary amounts as integers in the smallest currency unit, with an explicit ISO 4217 currency code.
- Store times with time zone (`timestamptz`). Never assume departure and arrival share a time zone; store the IATA airport code and resolve the zone from the airport record.

## Coding standards
- TypeScript strict mode; no `any` without a written justification in a comment.
- Small, reusable components. No duplicated business logic.
- Every async call handles loading, empty, success and error states.
- User-facing messages are clear and free of technical jargon.

## Naming conventions
- Files and folders: `kebab-case` (e.g. `flight-search-form.tsx`).
- React components: `PascalCase`.
- Database tables and columns: `snake_case`, plural table names.
- Environment variables: `SCREAMING_SNAKE_CASE`.

## Git workflow
- Work on feature branches. Never commit directly to the default branch without owner approval.
- Commit messages are in English, short and in the imperative mood (e.g. `Add flight search schema`).
- Do not commit or push unless the owner asks.

## Testing requirements
- Unit tests for flight-data normalization and price/time-zone logic.
- Manual test checklist for each milestone, written in `docs/TESTING-STRATEGY.md` (to be created).

## Documentation map
- `docs/PROJECT-ARCHITECTURE.md` — architecture and provider comparison
- `docs/DATABASE-SCHEMA.md` — tables, relations, RLS policies

## Approval gates
Ask the owner before any of the following: spending money (paid API tiers, paid Supabase or Vercel plans), deleting data or files, changing the database in a non-additive way, or adding features outside Phase 1.
