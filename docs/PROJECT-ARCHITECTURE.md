# Project Architecture — KM-VISA-FLIGHTS (Phase 1)

## 1. Scope of Phase 1
- Search flights (one-way, round-trip) by origin, destination, dates, passengers and cabin.
- Display and compare results: price, airlines, stops, duration, departure and arrival times (with time zones).
- Save searches and selected results for the owner's reference.
- Owner-only access (Supabase Auth).

Not in Phase 1: bookings, payments, ticketing, visa tracking, PDF generation, passenger database.

## 2. High-level architecture

```
Browser (owner)
   │  HTTPS
   ▼
Next.js App Router on Vercel
   ├── Pages & UI components (client + server components)
   ├── Route handlers / server actions  ──► Flight provider adapter ──► External flight API
   │                                              (server-only secrets)
   └── Supabase client ──► Supabase (PostgreSQL + Auth + RLS)
```

### Layers
- **UI layer:** pages and components under `app/` and `components/`. No business logic.
- **Application layer:** server actions and route handlers under `app/` that validate input and orchestrate calls.
- **Domain layer:** `lib/flights/` — normalization of offers, price and time-zone calculations, comparison sorting.
- **Provider adapter:** `lib/flights/providers/` — one interface (`searchOffers(query): Promise<Offer[]>`) with one implementation per provider. Switching provider means writing one new adapter.
- **Data layer:** Supabase access through server-side clients. Browser uses only the anon key with RLS.

### Security boundaries
- Provider API keys: server-only environment variables, set in Vercel and in `.env.local` (never committed).
- Supabase `service_role` key: server-only, only if strictly needed; default is RLS-respecting clients.
- Authentication: Supabase Auth (email + password, or magic link). A single owner account is created manually. Route protection is enforced in middleware and re-checked on the server.

## 3. Flight data source — comparison (decision pending)

The owner will choose the provider after reading this section. No provider is selected yet.

| Criterion | Amadeus (Self-Service) | Duffel | Kiwi.com (Tequila) |
|---|---|---|---|
| Type of offer | GDS-style offers (Amadeus network) | Direct airline content via API | Aggregated routes, including low-cost carriers and virtual interlining |
| Free test access | Test environment available | Test mode available | Requires an application |
| Production access | Requires account approval; check current status and terms | Requires account setup and agreement; check current status | Requires application and approval |
| Pricing model | Per-call or contract; **verify current price list** | Commission/fee model; **verify current terms** | Partner/affiliate terms; **verify current terms** |
| Eligibility for an Algerian business | **To verify** | **To verify** | **To verify** |
| Booking capability later | Yes (via GDS) | Yes (ordering API) | Limited; depends on terms |
| Fit for Phase 1 (search only) | Good | Good | Good for price breadth |

**Before deciding, the owner (or the assistant with web access) must verify, for each provider:**
1. Whether an Algerian-registered business can obtain production access.
2. Current price per search and any monthly minimum.
3. Whether search results may be stored and shown to the owner under its terms.
4. Whether test data is representative of live prices.

Recommendation: start with the free test environment of one provider to build the adapter and UI, then choose production access after verification. The adapter design keeps this decision reversible.

## 4. Non-functional requirements
- **Security:** no secret in the browser; RLS on every table; server-side validation.
- **Time zones:** every segment stores IATA codes and UTC timestamps; display converts using the airport's zone.
- **Money:** integers in minor units plus ISO currency code. No floating-point arithmetic for totals.
- **Performance:** search results are cached per query for a short period to limit paid API calls. Cache duration is configurable.
- **Reliability:** explicit loading, empty, success and error states on every search.
- **Accessibility:** responsive layout for desktop, tablet and mobile.

## 5. Deployment
- GitHub (private repository) → Vercel (automatic deployments from the default branch, preview deployments from feature branches).
- Supabase project `KMVISA FLIGHTS` (region eu-north-1). Migrations applied through versioned files.
- Environment variables set per environment in Vercel.

## 6. Risks
1. **Provider eligibility:** the chosen API may refuse Algerian business accounts. Mitigation: verify before building the production path.
2. **Cost growth:** each search may be billed. Mitigation: caching, per-day limits, and an owner-set budget.
3. **Data accuracy:** prices change between search and purchase. Mitigation: label results as "indicative, confirm before booking".
4. **Accidental secret exposure:** mitigated by the rules in `CLAUDE.md` and by a pre-commit check for `.env` files (to be added in a later milestone).

## 7. Open decisions for the owner
1. Flight data provider (after verification in section 3).
2. Whether search results may be cached and for how long.
3. Login method: email + password, or magic link.
4. Default currency shown to the owner (DZD, EUR or USD).
