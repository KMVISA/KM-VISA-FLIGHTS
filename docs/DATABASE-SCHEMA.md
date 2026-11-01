# Database Schema — KM-VISA-FLIGHTS (Phase 1)

Database: Supabase PostgreSQL, project `KMVISA FLIGHTS` (eu-north-1).
All tables live in schema `public`. **Row Level Security (RLS) is enabled on every table.**

Conventions:
- Primary keys: `id uuid default gen_random_uuid() primary key`.
- Timestamps: `created_at timestamptz default now()`, `updated_at timestamptz`.
- Money: `amount_minor integer` (smallest unit, e.g. cents) + `currency char(3)` (ISO 4217).
- Times: `timestamptz`. Airports carry their IANA time zone name.

## 1. Tables

### profiles
Links each Supabase Auth user to an application role. Phase 1 has only the owner.
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK, FK → `auth.users(id)` on delete cascade |
| full_name | text | |
| role | text | not null, default `'owner'`, check in (`'owner'`, `'admin'`, `'agent'`, `'client'`) |

Only `'owner'` is used in Phase 1. Other roles are reserved for later and have no UI.

### airports
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| iata_code | char(3) | not null, unique, uppercase check |
| icao_code | char(4) | nullable, unique |
| name | text | not null |
| city | text | not null |
| country_code | char(2) | not null (ISO 3166-1 alpha-2) |
| time_zone | text | not null (IANA, e.g. `Africa/Algiers`) |

Index: `(iata_code)` (unique), and a trigram or lower-case index on `name` and `city` for search.

### airlines
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| iata_code | char(2) | unique, nullable |
| icao_code | char(3) | unique, nullable |
| name | text | not null |
| country_code | char(2) | nullable |

### flight_searches
One row per search the owner runs.
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| owner_id | uuid | not null, FK → `profiles(id)` on delete cascade |
| trip_type | text | not null, check in (`'one_way'`, `'round_trip'`) |
| origin_airport_id | uuid | not null, FK → `airports(id)` |
| destination_airport_id | uuid | not null, FK → `airports(id)` |
| departure_date | date | not null |
| return_date | date | nullable; required when `trip_type = 'round_trip'` (check constraint) |
| adults | smallint | not null, check between 1 and 9 |
| cabin | text | not null, check in (`'economy'`, `'premium_economy'`, `'business'`, `'first'`) |
| provider | text | not null (e.g. `'amadeus'`, `'duffel'`, `'kiwi'`) |
| created_at | timestamptz | default now() |

Index: `(owner_id, created_at desc)`.

### flight_offers
Normalized offers returned by a provider for a search. Cached data, not a booking.
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| search_id | uuid | not null, FK → `flight_searches(id)` on delete cascade |
| provider_offer_id | text | not null |
| total_amount_minor | integer | not null, check ≥ 0 |
| currency | char(3) | not null |
| stops_total | smallint | not null, check ≥ 0 |
| fetched_at | timestamptz | not null, default now() |
| expires_at | timestamptz | not null (cache lifetime) |

Unique: `(search_id, provider_offer_id)`. Index: `(search_id, total_amount_minor)` for sorting.

### flight_segments
Each leg of an offer, in order. Supports connections and round trips.
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| offer_id | uuid | not null, FK → `flight_offers(id)` on delete cascade |
| direction | text | not null, check in (`'outbound'`, `'return'`) |
| sequence | smallint | not null, check ≥ 1 |
| departure_airport_id | uuid | not null, FK → `airports(id)` |
| arrival_airport_id | uuid | not null, FK → `airports(id)` |
| departure_at | timestamptz | not null (UTC) |
| arrival_at | timestamptz | not null (UTC) |
| marketing_airline_id | uuid | not null, FK → `airlines(id)` |
| operating_airline_id | uuid | nullable, FK → `airlines(id)` |
| flight_number | text | not null |
| cabin | text | not null |
| duration_minutes | integer | not null, check ≥ 0 |

Unique: `(offer_id, direction, sequence)`. Check: `arrival_at > departure_at`.

### saved_offers
Results the owner marks for later comparison.
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| owner_id | uuid | not null, FK → `profiles(id)` on delete cascade |
| offer_id | uuid | not null, FK → `flight_offers(id)` on delete cascade |
| note | text | nullable |
| created_at | timestamptz | default now() |

Unique: `(owner_id, offer_id)`.

## 2. Relationships
- `profiles` 1—N `flight_searches`
- `flight_searches` 1—N `flight_offers`
- `flight_offers` 1—N `flight_segments`
- `airports` 1—N `flight_searches` (origin/destination) and `flight_segments` (departure/arrival)
- `airlines` 1—N `flight_segments` (marketing and operating)
- `profiles` N—N `flight_offers` through `saved_offers`

## 3. Cascading behavior
- Deleting a search deletes its offers and their segments.
- Deleting a profile deletes their searches and saved offers.
- Airports and airlines are referenced with `on delete restrict`: reference data cannot be removed while in use.

## 4. Row Level Security policies

Owner-only access. The owner is the only authenticated user in Phase 1.

| Table | Policy | Rule |
|---|---|---|
| profiles | select/update own row | `auth.uid() = id` |
| flight_searches | all operations | `auth.uid() = owner_id` |
| saved_offers | all operations | `auth.uid() = owner_id` |
| flight_offers | select/insert/delete | exists a `flight_searches` row with `owner_id = auth.uid()` for `search_id` |
| flight_segments | select/insert/delete | same check through `flight_offers` → `flight_searches` |
| airports | select | `auth.role() = 'authenticated'` |
| airports | insert/update | `auth.uid()` has role `'owner'` in `profiles` |
| airlines | select | `auth.role() = 'authenticated'` |
| airlines | insert/update | same as airports |

Anonymous (unauthenticated) access is denied on every table. Policies are written in migration files and tested with an anon-key client and an authenticated client before each release.

## 5. Data validation
- Database checks enforce ranges, enums and date logic.
- The server validates the same rules again (Zod schemas) before writing.
- IATA codes are uppercase, three letters; ICAO codes are four letters.

## 6. Indexes (summary)
- `airports(iata_code)` unique; `airports` text search index on `name`, `city`.
- `flight_searches(owner_id, created_at desc)`.
- `flight_offers(search_id, total_amount_minor)`.
- `flight_segments(offer_id, direction, sequence)` unique.
- `saved_offers(owner_id, offer_id)` unique.

## 7. Migration plan
1. `001_profiles_and_reference.sql` — profiles, airports, airlines, RLS.
2. `002_searches_and_offers.sql` — flight_searches, flight_offers, flight_segments, saved_offers, RLS.
3. Seed data for airports and airlines from a reviewed open dataset (source to be chosen; license must allow use).

Every migration is additive in Phase 1. Destructive changes require owner approval.
