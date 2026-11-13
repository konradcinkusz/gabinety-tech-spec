# Gabinety Tech — Contracts

> Normative. The data models, HTTP contracts, state machines and formulas every service and the web app are built against.
> **Contracts first** (SUPERVISOR §4): this file is owned by the supervisor. An implementer never edits it; a task that finds it
> wrong stops and says so in the PR ("Blocked: question for supervisor"). A change lands in a specification PR first, then
> in code. Narrative and rules are in [`SPEC.md`](SPEC.md); the LLM gateway is in [`COPILOT-SDK.md`](COPILOT-SDK.md).
>
> Notation: `string(n)` is text of at most n characters; `ts` is an instant in UTC; `money` is a whole number of grosz
> (`long`, one PLN is 100 grosz) with the currency `PLN` implied; `uuid` is a version 7 UUID (`Guid.CreateVersion7()`).

## 0. Conventions

| Topic | Rule |
|---|---|
| Base path | `/api/v1/...` for callers acting as a user, `/internal/v1/...` for service calls (ADR-0007), `/webhooks/...` for providers |
| JSON | `camelCase` properties, enums as strings, instants as ISO 8601 with `Z` (`2026-12-01T09:30:00Z`), no `null` in a request where "absent" is meant (omit the property) |
| Money | In JSON as an object: `amountMinor` (integer) and `currency` (`"PLN"`). Never a decimal number. C# type `long` in minor units, wrapped in the `Money` record of `Gabinety.Billing` |
| Ids | `uuid`. A `slug` is `[a-z0-9]+(-[a-z0-9]+)*`, 3 to 60 characters |
| Errors | RFC 9457 `application/problem+json`: `type` = `urn:gabinety:error:<code>`, `title`, `status`, `detail`, extension `code` (the `<code>` of §10) and, for validation, `errors`: a map of property to messages. `detail` never contains user free text, a secret or a stack trace |
| Pagination | Cursor: `?limit=` (default 12, maximum 24; 50 for admin lists) and `?cursor=`; response `{ "items": [...], "nextCursor": "..." }` with `nextCursor` omitted on the last page. The cursor is opaque; ordering is total (it ends with the id) |
| Idempotency | `POST /api/v1/bookings` requires the header `Idempotency-Key` (a uuid chosen by the client). Same key and same caller within 24 h returns the original response, status code included. Same key with a different body: `409 idempotency_conflict` |
| Time | Stored and transferred in UTC. Availability rules are local wall-clock times in `Europe/Warsaw` and are converted by `Gabinety.Scheduling` (§9.4). Nothing else knows about the zone except the web app, which formats for display |
| Auth | Public: no header. User: `Authorization: Bearer <access token>` from authservice (the web app forwards the user's token). Internal: `X-Internal-Key`. Admin: user token with the authservice role `Admin` or `SuperAdmin` |
| Versioning | A breaking change is a new `/api/v2` path, never an edit of v1. Additive fields are allowed |
| OpenAPI | Every service publishes `/openapi/v1.json` in development and in CI; `web/` generates TypeScript types from them (SPEC §5 rule 9) |
| Rate limits | Table in §10.2 |

## 1. Identity

Token claims used (names are an assumption until spike T-01.01 reads them from authservice, A-01; the contract is the
abstraction, not the claim string):

| Abstraction | Source | Used for |
|---|---|---|
| `UserId` | token subject (`sub`) | the owner of a practitioner profile, a booking, a review |
| `IsAdmin` | the authservice platform role `Admin` or `SuperAdmin` | every `/api/v1/admin/...` endpoint |
| `Email`, `EmailVerified` | token claims, if present; otherwise authservice `GET /api/v1/auth/me` | verification badge, notifications |

Roles of the marketplace are facts in Directory (§3): a user **is a practitioner** when a `practitioners` row has their
`UserId`; **is a moderator** when a `moderators` row has it; every signed-in user is a client. No token claim names them (ADR-0002).

## 2. Taxonomy

Two kinds of category, both rows of `categories` in Directory, seeded by a versioned script (P4, never `HasData`):

| Kind | Meaning | Seed set (slug: Polish name) |
|---|---|---|
| `problem` | What the client wants done | `audyt-kodu`: Audyt kodu · `migracja-legacy-dotnet`: Migracja legacy .NET · `integracja-ai`: Integracja AI · `automatyzacja-procesow`: Automatyzacja procesów · `architektura-systemu`: Architektura systemu · `code-review`: Code review · `wycena-projektu`: Wycena i plan projektu · `rozwiazanie-awarii`: Rozwiązanie awarii |
| `tech` | A technology | `dotnet`: .NET · `csharp`: C# · `aspnet-core`: ASP.NET Core · `entity-framework`: Entity Framework · `azure`: Azure · `sql-server`: SQL Server · `postgresql`: PostgreSQL · `python`: Python · `llm`: LLM i agenci · `kubernetes`: Kubernetes · `react`: React · `typescript`: TypeScript |

Page structure for SEO (SPEC §6.7): `/programisci` (all), `/programisci/{problem}` and `/programisci/{problem}/{tech}`.
A taxonomy version string (`2026-11-v1`) is stored in a `taxonomy_meta` row; a seed script raises it when it changes the set.

## 3. Directory (`directorydb`)

### 3.1 Data model

| Table | Columns (type) |
|---|---|
| `practitioners` | `id` uuid PK · `user_id` string(64) unique · `slug` string(60) unique · `display_name` string(80) · `headline` string(120) · `bio` string(2000), plain text · `github_login` string(39)? · `github_linked` bool · `nip` string(10)? · `nip_valid` bool · `portfolio_url` string(300)? https only · `meeting_url_default` string(300)? https only · `status` enum `Draft\|Submitted\|Approved\|Rejected\|Suspended` · `status_reason` string(500)? · `approved_at` ts? · `created_at` ts · `updated_at` ts · `rating_count` int · `rating_avg` decimal(3,2) · `reputation_version` long · `completed_count` int · `practitioner_cancel_count` int · `no_show_upheld_count` int · `open_slots_14d` int · `stats_version` long · `profile_completeness` decimal(3,2) · `ranking_score` decimal(5,2) · `ranking_version` string(20) |
| `offerings` | `id` uuid PK · `practitioner_id` FK · `title` string(80) · `description` string(600) · `duration_minutes` smallint (30 or 60) · `price_minor` long (5000 to 150000) · `currency` string(3) = `PLN` · `problem_slug` string(60) FK · `tech_slugs` array of string · `meeting_url` string(300)? https only · `is_active` bool · `created_at` ts |
| `categories` | `slug` PK · `kind` `problem\|tech` · `name_pl` string(80) · `sort` int · `is_active` bool |
| `taxonomy_meta` | `version` string(20), one row |
| `moderators` | `user_id` PK |
| `verification_events` | `id` PK · `practitioner_id` FK · `kind` `Submitted\|Approved\|Rejected\|Suspended\|Reinstated\|GithubLinked\|NipChanged` · `actor_user_id` string(64) · `note` string(500)? · `at` ts. Append-only |

Indexes: `practitioners(status, ranking_score desc, approved_at, id)` for the listing, `offerings(problem_slug, is_active)`,
GIN on `offerings(tech_slugs)` (PostgreSQL). A query that filters then projects orders **before** it projects (CLAUDE lesson
#78 of the chess project: an `OrderBy` over a constructed record does not translate on Npgsql); every list query has a
`tier=integration` test on a real PostgreSQL.

### 3.2 Public API (no authentication)

| Method, path | Query / body | Response |
|---|---|---|
| `GET /api/v1/taxonomy` | — | `{ version, problems: [{slug, namePl}], techs: [{slug, namePl}] }` |
| `GET /api/v1/practitioners` | `problem`, `tech`, `q` (≤60, matches name and headline, case-insensitive), `duration` (30\|60), `maxPriceMinor`, `cursor`, `limit` | `{ items: [PractitionerCard], nextCursor? }`; only `Approved`; order = ranking (§9.3): score desc, `approved_at` asc, id asc |
| `GET /api/v1/practitioners/{slug}` | — | `PractitionerProfile` or `404 not_found` (also for any status other than `Approved`) |
| `GET /api/v1/ranking-rules` | — | `{ version, effectiveFrom, weights: { rating, reliability, availability, completeness }, constants: { m, mu, minReviewsToShowRating, slotsForFullAvailability } }`, produced from `Gabinety.Ranking` (the single source of the public page `/zasady-rankingu`) |
| `GET /api/v1/sitemap` | — | `{ practitioners: [{slug, lastModified}], categories: [{path, practitionerCount, indexable}] }`; `indexable` is true from 3 approved practitioners |

`PractitionerCard`: `id`, `slug`, `displayName`, `headline`, `githubLogin?`, `badges` (array of `email`, `github`, `nip`, `approved`),
`rating?` (`{ average, count }`, present only when `count` ≥ 3), `isNew` (true when `count` < 3), `fromPrice` (money, the lowest active offering),
`problemSlugs`, `durations`.
`PractitionerProfile`: the card plus `bio`, `portfolioUrl?`, `offerings: [Offering]` (active only; `meetingUrl` never included), `completedCount`, `memberSince`.
`Offering`: `id`, `title`, `description`, `durationMinutes`, `price` (money), `problemSlug`, `techSlugs`.

### 3.3 User API (Bearer)

| Method, path | Who | Request | Response / rules |
|---|---|---|---|
| `GET /api/v1/me` | any user | — | `{ userId, practitionerId?, practitionerStatus?, isAdmin, isModerator }` |
| `POST /api/v1/me/practitioner` | any user without one | `{ displayName, slug? }` | `201` profile in `Draft`; slug generated from the name when absent, made unique with a numeric suffix; `409 practitioner_exists` |
| `GET /api/v1/me/practitioner` | practitioner | — | full private profile including `status`, `statusReason`, `meetingUrlDefault`, `nip`, verification flags |
| `PUT /api/v1/me/practitioner` | practitioner | `{ displayName, headline, bio, portfolioUrl?, meetingUrlDefault?, nip? }` | replaces the editable fields; validation §3.5; a change of `nip` sets status `Submitted` if it was `Approved` |
| `POST /api/v1/me/practitioner/submit` | practitioner in `Draft` or `Rejected` | — | `204`; requires the submit checklist (§3.5), else `422 submit_incomplete` with the unmet items |
| `GET /api/v1/me/practitioner/offerings` | practitioner | — | `{ items: [OfferingPrivate] }` including inactive and `meetingUrl` |
| `POST /api/v1/me/practitioner/offerings` | practitioner | `{ title, description, durationMinutes, priceMinor, problemSlug, techSlugs[], meetingUrl? }` | `201`; at most 5 active offerings per practitioner (`422 too_many_offerings`) |
| `PUT /api/v1/me/practitioner/offerings/{id}` | owner | same | price and duration changes affect only bookings made afterwards |
| `POST /api/v1/me/practitioner/offerings/{id}/deactivate` | owner | — | `204`; existing bookings unaffected |
| `POST /api/v1/me/practitioner/github-link` | practitioner | — | `204`; Directory reads the GitHub login from the user's authservice account (`GET /api/v1/auth/me` or the claim, A-01) and sets `github_linked` and `github_login`; `422 github_not_linked` when the account has no GitHub sign-in |

### 3.4 Admin API (`IsAdmin`)

| Method, path | Request | Rules |
|---|---|---|
| `GET /api/v1/admin/practitioners` | `status` (default `Submitted`), `cursor`, `limit` | oldest first |
| `POST /api/v1/admin/practitioners/{id}/approve` | — | from `Submitted` only; sets `Approved`, `approved_at`; computes ranking; writes a `verification_events` row |
| `POST /api/v1/admin/practitioners/{id}/reject` | `{ reason }` (10 to 500) | from `Submitted`; sets `Rejected` and `status_reason` |
| `POST /api/v1/admin/practitioners/{id}/suspend` | `{ reason }` | from `Approved`; hides the profile; Booking refuses new bookings (it reads status through the offering lookup) |
| `POST /api/v1/admin/practitioners/{id}/reinstate` | — | from `Suspended` to `Approved` |
| `PUT /api/v1/admin/moderators/{userId}` / `DELETE` | — | `SuperAdmin` only |

### 3.5 Validation

- `displayName`: 2 to 80 characters after trimming, no control characters. `headline`: 10 to 120. `bio`: 0 to 2000, plain text.
- URLs: `https` scheme, a host that is not an IP address and not `localhost`, at most 300 characters. A meeting URL must be on a host of
  the allow-list `Directory:MeetingHosts` (default `meet.google.com`, `zoom.us`, `teams.microsoft.com`, `whereby.com`, `meet.jit.si`).
- `nip`: ten digits; valid when the weighted checksum holds (weights 6,5,7,2,3,4,5,6,7; sum mod 11 equals the tenth digit; a remainder of 10 is invalid).
  Test vector: `5260250995` valid, `5260250996` invalid.
- Free text (`headline`, `bio`, offering text) must not contain an e-mail address, a phone number or a messaging handle (detector in
  `Gabinety.Directory` using a regular expression set, tested with positive and negative cases): `422 contact_details_not_allowed`. This is an anti-leakage control (SPEC §6.1), not security.
- Submit checklist: `headline` filled, `bio` of at least 200 characters, at least one active offering, `github_linked`, `nip_valid`, `portfolioUrl` filled, an e-mail verified at authservice.
- `profile_completeness`: sum of headline 0.15, bio of at least 200 characters 0.20, one active offering 0.25, `github_linked` 0.10, portfolio 0.10, `nip_valid` 0.10, `open_slots_14d` > 0 0.10.

### 3.6 Internal API (`X-Internal-Key`)

| Method, path | Caller | Body → effect |
|---|---|---|
| `GET /internal/v1/offerings/{id}` | Booking | `{ offeringId, practitionerId, practitionerUserId, practitionerStatus, title, durationMinutes, price, meetingUrl (offering's or the practitioner's default), isActive }` |
| `GET /internal/v1/practitioners/by-user/{userId}` | Booking, Reputation | `{ practitionerId, status }` or `404` |
| `PUT /internal/v1/practitioners/{id}/reputation` | Reputation | `{ ratingCount, ratingAvg, version }`: applied only when `version` is greater than the stored `reputation_version`; recomputes the ranking; `204` also when ignored (idempotent) |
| `PUT /internal/v1/practitioners/{id}/booking-stats` | Booking | `{ completedCount, practitionerCancelCount, noShowUpheldCount, openSlots14d, version }`: same monotonic rule on `stats_version` |
| `DELETE /internal/v1/subjects/{userId}` | an administrator's erasure job | anonymises the practitioner row (display name `Usunięty użytkownik`, text cleared, status `Suspended`, `user_id` replaced by a random value) |

## 4. Booking (`bookingdb`)

### 4.1 Data model

| Table | Columns |
|---|---|
| `availability_rules` | `id` · `practitioner_id` uuid · `iso_weekday` smallint (1 Monday to 7 Sunday) · `start_local` time · `end_local` time. Start and end on :00 or :30, end after start, no overlap on a day |
| `availability_exceptions` | `id` · `practitioner_id` · `date_local` date · unique together. A blocked day |
| `bookings` | `id` uuid PK · `client_user_id` · `practitioner_id` · `practitioner_user_id` · `offering_id` · `offering_title` · `duration_minutes` · `price_minor` long · `start_utc` ts · `end_utc` ts · `status` (§8.1) · `status_reason`? · `brief_text` string(2000) · `brief_problem_slug`? · `brief_ai_assisted` bool · `meeting_url_snapshot`? (set when `Confirmed`) · `hold_expires_at` ts? · `fee_minor` long? · `payout_minor` long? · `payout_due_at` ts? · `idempotency_key` uuid · `created_at` · `confirmed_at`? · `ended_at`? · `cancelled_at`? · `dispute_opened_at`? · `dispute_resolution`? · `resolved_at`? |
| `payments` | `id` · `booking_id` unique · `provider` string(20) · `provider_payment_id` string(100)? · `amount_minor` · `status` `Created\|Succeeded\|Failed\|Refunded\|PartiallyRefunded` · `refunded_minor` long · `created_at` · `updated_at` |
| `webhook_events` | `provider` · `event_id` · PK together · `received_at` · `outcome` `Applied\|Duplicate\|Ignored\|Rejected` |
| `ledger_entries` | `id` · `booking_id` · `kind` `ClientCharge\|Refund\|PlatformFee\|PractitionerPayout\|PayoutReversal` · `amount_minor` long (always positive; the kind gives the direction) · `pair_key` string(80)? (set on `PlatformFee`: `<client_user_id>:<practitioner_id>`) · `created_at`. Append-only. **Unique partial index** on `pair_key` where `kind = PlatformFee` makes "one fee per pair" race-safe |
| `payouts` | `id` · `booking_id` unique · `practitioner_id` · `amount_minor` · `status` `Pending\|Sent\|Failed` · `provider_ref`? · `created_at` · `sent_at`? |
| `notifications` | `id` · `booking_id` · `kind` `BookingConfirmed\|Reminder24h\|Reminder1h\|Cancelled\|ReviewRequest\|DisputeOpened\|DisputeResolved\|PayoutSent` · `to_user_id` · `due_at` ts · `sent_at`? · `attempts` int. Unique `(booking_id, kind, to_user_id)` |

**No double booking.** PostgreSQL: an exclusion constraint
`EXCLUDE USING gist (practitioner_id WITH =, tstzrange(start_utc, end_utc + interval '15 minutes') WITH &&) WHERE (status IN ('PendingPayment','Confirmed','Disputed'))`
(extension `btree_gist`), in a provider-specific migration. The application also serialises creation per practitioner (an advisory lock) and maps the constraint
violation to `409 slot_taken`. The InMemory provider cannot enforce the constraint, so the rule has a `tier=integration` test on a real PostgreSQL with two
concurrent requests for one slot: exactly one succeeds.

### 4.2 Public and user API

| Method, path | Who | Request → response |
|---|---|---|
| `GET /api/v1/practitioners/{practitionerId}/slots` | public | `offeringId`, `from`, `to` (dates, range ≤ 14 days, within 60 days) → `{ slots: [{ startUtc, endUtc }] }` from §9.4, minus active bookings and their 15 minute buffer, minus the lead time |
| `GET /api/v1/me/availability` | practitioner | `{ rules: [...], exceptions: [...] }` |
| `PUT /api/v1/me/availability` | practitioner | `{ rules: [{ isoWeekday, startLocal, endLocal }], exceptions: [{ dateLocal }] }` replaces both; at most 28 rules, 90 exceptions; recomputes `openSlots14d` and pushes booking-stats to Directory |
| `POST /api/v1/bookings` | client, header `Idempotency-Key` | `{ offeringId, startUtc, brief: { text (20 to 2000), problemSlug?, aiAssisted } }` → `201 { bookingId, status: "PendingPayment", holdExpiresAt, checkoutUrl }`. Errors: `422 slot_unavailable`, `409 slot_taken`, `422 offering_inactive`, `422 own_offering`, `422 practitioner_not_bookable`, `429 rate_limited` |
| `GET /api/v1/bookings/{id}` | client, practitioner of it, or admin | `BookingView` (§4.4) |
| `GET /api/v1/me/bookings` | any user | `role` (`client`\|`practitioner`), `status`, `cursor` → `{ items: [BookingView], nextCursor? }`, newest start first |
| `POST /api/v1/bookings/{id}/cancel` | client or practitioner of it | `{ reason? }` → `BookingView`; §8.1 and §9.2 decide status and refund |
| `POST /api/v1/bookings/{id}/dispute` | client | `{ kind: "not_as_described"\|"practitioner_no_show"\|"other", text (10 to 1000) }` → `BookingView`; allowed from `ended_at` until `ended_at + 48 h`, or for `practitioner_no_show` from `start_utc` until `start_utc + 72 h` |
| `POST /api/v1/bookings/{id}/report-client-no-show` | practitioner | — → `BookingView`; allowed from `start_utc + 15 min` until `start_utc + 72 h` |
| `POST /api/v1/payments/simulate` | any signed-in user, **only when `Payments:Provider` is `Mock`**, otherwise `403` | `{ bookingId, outcome: "succeeded"\|"failed" }` → runs the same handler as the webhook |

### 4.3 Webhook, admin, internal

| Method, path | Rules |
|---|---|
| `POST /webhooks/payments/{provider}` | Public path, provider signature over the **raw body** verified in constant time before any parsing; no secret configured: accepted in Mock mode, rejected in any other mode. Idempotent on `(provider, event_id)`. Provider events map to the internal vocabulary `PaymentSucceeded`, `PaymentFailed`, `RefundSucceeded`, `Unknown` at the boundary; an unknown event is logged and recorded as `Ignored`. A webhook for a booking row that does not exist yet is retried by returning `202` with a `Retry-After`. The amount and currency are checked against the booking: a mismatch records `Rejected` and raises an alert log |
| `GET /api/v1/admin/disputes` | `status` (`open` default, `resolved`) → list of disputed bookings with party ids, reason, text |
| `POST /api/v1/admin/bookings/{id}/resolve` | `{ resolution: "release"\|"refund_full"\|"refund_partial", refundMinor? (partial: 1 to price − 1), note (10 to 500) }`; from `Disputed` only |
| `GET /api/v1/admin/ledger.csv` | `from`, `to` (dates) → CSV `booking_id,kind,amount_minor,currency,created_at`, UTF-8, header row, no personal data |
| `GET /internal/v1/bookings/{id}/reviewability` | Reputation → `{ reviewable, clientUserId, practitionerId, completedAt }`; reviewable only for `Completed` (or `Resolved` with `release`), within 14 days after `ended_at` |
| `DELETE /internal/v1/subjects/{userId}` | anonymises `brief_text` and user ids on that user's bookings; ledger rows stay (accounting) |

### 4.4 `BookingView`

`id`, `status`, `statusReason?`, `startUtc`, `endUtc`, `durationMinutes`, `offeringTitle`, `price` (money), `practitioner` (`id`, `displayName`, `slug`),
`client` (`displayName` only, visible to the practitioner once `Confirmed`), `brief` (visible to the practitioner only after `Confirmed`, to the
client always), `meetingUrl` (present only when status is `Confirmed` and the caller is a party), `holdExpiresAt?`, `payment` (`status`, `refunded`),
`fee?`, `payout?` (money; visible to the practitioner and admin only), `canCancel`, `canDispute`, `canReportNoShow`, `canReview` (booleans computed server-side).

## 5. Reputation (`reputationdb`)

### 5.1 Data model

| Table | Columns |
|---|---|
| `reviews` | `id` · `booking_id` unique · `practitioner_id` · `client_user_id` · `rating` smallint 1..5 · `text` string(2000) · `status` `Published\|Held\|Rejected` · `hold_reasons` array of string · `triage_flag` `none\|hold\|unavailable` · `triage_model_id`? · `created_at` · `published_at`? · `moderated_by`? · `moderation_note`? · `reply_text` string(1000)? · `replied_at`? |
| `rating_summaries` | `practitioner_id` PK · `count` int · `sum` int · `version` long (increments on every change) · `pushed_version` long |
| `moderation_events` | `id` · `review_id` · `kind` `Created\|Held\|Published\|Rejected\|Replied` · `actor` (`system`\|`triage`\|user id) · `note`? · `at` |

### 5.2 API

| Method, path | Who | Rules |
|---|---|---|
| `POST /api/v1/reviews` | client | `{ bookingId, rating, text (20 to 2000) }` → `201 { reviewId, status }`. Reputation asks Booking `reviewability`; `422 not_reviewable`, `409 already_reviewed`. Then §5.3 |
| `GET /api/v1/practitioners/{practitionerId}/reviews` | public | `cursor`, `limit` (≤ 20) → `{ items: [{ id, rating, text, createdAt, reply? }], nextCursor? }`, `Published` only, newest first. The reviewer is shown as "Zweryfikowany klient" with no name |
| `POST /api/v1/reviews/{id}/reply` | the reviewed practitioner | `{ text (10 to 1000) }`, once; `409 already_replied` |
| `GET /api/v1/admin/reviews` | admin | `status` (`Held` default), `cursor` |
| `POST /api/v1/admin/reviews/{id}/publish` | admin | from `Held` |
| `POST /api/v1/admin/reviews/{id}/reject` | admin | `{ reason }`; from `Held` or `Published`; the practitioner cannot reject |
| `DELETE /internal/v1/subjects/{userId}` | internal | replaces the review text with `Opinia usunięta na wniosek autora`, keeps the rating |

### 5.3 Publication rule

1. Deterministic checks (pure, in `Gabinety.Reputation`): any of these puts the review in `Held` with that reason code — `contains_link` (more than one URL, or any URL on a host outside a short allow-list of code hosts), `contact_details`, `blocked_term` (a list in configuration), `repetition` (a character or word repeated more than 8 times), `too_short_after_trim`.
2. If it passed, Reputation calls the Assistant triage (§6, 25 s, no retry). `hold` → `Held` with reasons from the answer; `none` → `Published`; the Assistant unavailable, over budget or invalid → `triage_flag = unavailable` and the review is **published** (a model is advice and its absence never blocks a person; SPEC §5 rule 6).
3. A `Published` review updates `rating_summaries` in the same transaction; a background service pushes unpushed summaries to Directory (`PUT .../reputation`), retrying with backoff.

## 6. Assistant (`assistantdb`, usage log only)

| Method, path | Who | Request → response |
|---|---|---|
| `GET /api/v1/assistant/status` | public | `{ state: "Ready"\|"Fake"\|"NotAuthenticated"\|"Unavailable"\|"Disabled", features: ["brief", "review-triage"] }`. `Disabled` means the budget is spent or the policy guard refuses the configured model (`COPILOT-SDK.md` §7) |
| `POST /api/v1/assistant/brief` | signed-in user | `{ text (20 to 2000) }` → `200 BriefSuggestion`, or `503 assistant_unavailable` (the web app then shows the plain form), `429 rate_limited`, `422 invalid_input` |
| `POST /internal/v1/assistant/review-triage` | Reputation | `{ rating, text }` (no ids) → `{ flag: "none"\|"hold", reasons: [ "spam"\|"abuse"\|"personal_data"\|"off_topic"\|"threat" ], modelId }` or `503` |

`BriefSuggestion`: `problemSlug` (one of the taxonomy `problem` slugs), `techSlugs` (0 to 4, taxonomy `tech` slugs), `summary` (≤ 300 characters, in Polish),
`clarifyingQuestions` (0 to 3 strings of ≤ 140), `suggestedDurationMinutes` (30 or 60), `confidence` (`low`\|`medium`\|`high`), `modelId`, `promptVersion`.
The Assistant validates the model's JSON against this schema and against the live taxonomy (§2, fetched from Directory, cached 10 minutes, stale value used when
Directory is down for up to 24 h). Invalid output gets one repair attempt, then `503`; **no suggestion is ever invented** (the analogue of "no fallback move").

`assistant_usage` (assistantdb): `id` · `at` · `feature` `brief\|review-triage` · `model_id` · `provider_kind` `copilot\|custom\|fake` · `input_tokens` int? · `output_tokens` int? · `ai_credits` decimal? · `usd_estimate` decimal? · `latency_ms` int · `outcome` `ok\|invalid_output\|timeout\|unavailable\|over_budget\|policy_refused` · `user_hash` string(16)? (a salted hash, never the id). No prompt and no reply is stored.

## 7. Web app BFF (not a service contract)

The browser talks to its own origin only. Routes: `/api/config` (runtime configuration), `/api/auth/login|register|logout|refresh|github/start|github/callback` (session handling,
calls authservice), `/api/proxy/[...path]` (catch-all to the services by the candidate ladder of `FRONTEND-BFF` §5). State-changing requests (`POST`, `PUT`, `DELETE`) are refused with
`403` when `Origin` is foreign or `Sec-Fetch-Site` is `cross-site`. Cookies: `HttpOnly`, `Secure` outside development, `SameSite=Lax`, host-only.

## 8. State machines

### 8.1 Booking

| From | Trigger (who) | To | Side effects |
|---|---|---|---|
| — | create (client) | `PendingPayment` | slot held until `hold_expires_at` (created + 10 min); payment row `Created` |
| `PendingPayment` | `PaymentSucceeded` | `Confirmed` | set `confirmed_at`, `meeting_url_snapshot`; notifications `BookingConfirmed` (both), `Reminder24h`, `Reminder1h`, `ReviewRequest` scheduled; ledger `ClientCharge` |
| `PendingPayment` | hold expires or `PaymentFailed` | `Expired` | slot released; if a payment succeeds after expiry it is refunded in full and recorded `Refund` |
| `Confirmed` | cancel (client), start − now ≥ 24 h | `CancelledByClient` | full refund (§9.2); ledger `Refund`; stats unchanged |
| `Confirmed` | cancel (client), start − now < 24 h | `LateCancelledByClient` | no refund; `payout_due_at = start_utc`; fee as for completion; no review |
| `Confirmed` | cancel (practitioner), any time before end | `CancelledByPractitioner` | full refund; `practitioner_cancel_count` + 1 |
| `Confirmed` | time passes `end_utc + 48 h` with no dispute | `Completed` | `ended_at = end_utc`; fee and payout computed (§9.1); `payout_due_at = end_utc + 48 h`; stats `completed_count` + 1 |
| `Confirmed` | report-client-no-show (practitioner) | `ClientNoShow` | counts as completed for money; `payout_due_at` = report time; no review |
| `Confirmed` | dispute (client) within the window | `Disputed` | payout frozen; admin notified |
| `Disputed` | resolve (admin) | `Resolved` | `release`: as `Completed`; `refund_full`: refund, and for `practitioner_no_show` `no_show_upheld_count` + 1; `refund_partial`: refund of `refundMinor`, payout of the rest minus fee on the rest |

Any other combination answers `409 invalid_transition`. A transition is one database transaction that also writes its notification rows. Jobs that move
bookings (hold expiry, auto-complete, payout) are hosted services that read `TimeProvider`, run every 60 seconds, and are idempotent: running one twice changes nothing.

### 8.2 Practitioner profile

`Draft` → (submit) `Submitted` → (approve) `Approved` | (reject) `Rejected` → (edit, submit) `Submitted`; `Approved` → (suspend) `Suspended` → (reinstate) `Approved`. Only `Approved` is public.

### 8.3 Review

`Held` ⇄ `Published` by an administrator; `Rejected` is final. A review is created directly as `Published` or `Held` (§5.3).

## 9. Formulas

### 9.1 Fee and payout (`Gabinety.Billing`, pure)

`rate_bps` is `Billing:FirstConsultationFeeBps` (default 1500, D-23).
`fee = (price_minor * rate_bps + 5000) / 10000` with integer division (this rounds half up), and `fee = 0` when a `PlatformFee` ledger row already exists for the pair (the unique index decides a race: the loser's insert fails and its fee is 0).
`payout = price_minor - fee`. For a partial refund `r`: `kept = price_minor - r`, `fee = (kept * rate_bps + 5000) / 10000` (same pair rule), `payout = kept - fee`.
Test vectors (price → fee, payout): 25000 → 3750, 21250 · 33333 → 5000, 28333 · 5000 → 750, 4250 · 1 → 0, 1 · 150000 → 22500, 127500.
No floating point; the arithmetic is checked (`checked`) so an overflow throws.

### 9.2 Refund and money rules

| Event | Refund to client | Practitioner receives | Fee |
|---|---|---|---|
| Client cancels 24 h or more before start | 100 % | nothing | none |
| Client cancels less than 24 h before start | 0 % | `price - fee` | yes (pair rule) |
| Practitioner cancels | 100 % | nothing | none |
| Hold expired, payment later succeeds | 100 % | nothing | none |
| Completed, no dispute | 0 % | `price - fee` after `end + 48 h` | yes |
| Client no-show reported | 0 % | `price - fee` | yes |
| Dispute resolved `release` | 0 % | `price - fee` | yes |
| Dispute resolved `refund_full` | 100 % | nothing | none |
| Dispute resolved `refund_partial` | `refundMinor` | `kept - fee(kept)` | yes, on `kept` |

The platform never holds a balance: money moves through the provider (ADR-0005). A refund is requested through `IPaymentProvider.RefundAsync`; the ledger row is written when the provider confirms (`RefundSucceeded`).

### 9.3 Ranking (`Gabinety.Ranking`, pure) — version `2026-11-v1`

Inputs per practitioner: `n` published review count, `r` their mean, `c` completed count, `x` practitioner cancellations, `s` upheld practitioner no-shows, `p` profile completeness in [0, 1], `k` open slots in the next 14 days.

- `B = ((n * r) + (m * mu)) / (n + m) / 5` with `m = 5`, `mu = 3.8`
- `C = (c + 3) / (c + x + s + 4)`
- `A = min(1, k / 10)`
- `P = p`
- `score = 100 * (0.45 * B + 0.25 * C + 0.15 * A + 0.15 * P)`, stored with two decimals (round half away from zero)

Order: `score` desc, then `approved_at` asc, then `id` asc. **No input is a payment, a plan, a promotion, a fee, a model output or a cost.**
Test vectors (`score`, two decimals): `n=0, r=0, c=0, x=0, s=0, p=1, k=10` → 82.95 · `n=10, r=4.8, c=20, x=1, s=0, p=0.8, k=5` → 82.70 · `n=3, r=5.0, c=3, x=0, s=0, p=0.5, k=0` → 67.18. They were computed independently of the implementation; the tests assert them with a tolerance of 0.005.
Display rule (not the ranking): the page shows the plain mean to one decimal and the count only when `n` ≥ 3; below that it shows "Nowy". A tiny sample is never presented as an average (the chess project's lesson on significance).
Changing a weight, a constant or an input changes the version string, adds a row to `docs/operations/RANKING-CHANGELOG.md`, and updates the public page `/zasady-rankingu` in the same PR.

### 9.4 Slots (`Gabinety.Scheduling`, pure)

Given availability rules, exceptions, the offering duration `d` (30 or 60 minutes), existing busy intervals, `now`, and a date range:
1. For each local date in the range, for each rule on that ISO weekday, skip the date if it is an exception; generate start times every 30 minutes from `start_local` while `start + d <= end_local`.
2. Convert each local start to UTC in `Europe/Warsaw`: a local time that does not exist (spring forward) is skipped; an ambiguous one (autumn back) uses the first occurrence.
3. Drop a slot whose start is earlier than `now + 12 h` (`Booking:MinLeadHours`) or later than `now + 60 days`.
4. Drop a slot whose interval `[start, end + 15 min)` overlaps a busy interval `[start, end + 15 min)` of an active booking (a 15 minute buffer after each).
5. Output ordered by start, unique.
DST test vectors: 2027-03-28 (clocks 02:00 to 03:00): a rule 01:00–04:00 yields starts 01:00 and 01:30 (local, +01:00) and 03:00, 03:30 (local, +02:00), none in 02:xx; 2027-10-31 (03:00 back to 02:00): a rule 02:00–04:00 yields the first 02:00 and 02:30 (+02:00), then 03:00 and 03:30 (+01:00); with `d = 60` the same autumn rule 02:00–04:00 yields starts 02:00, 02:30 and 03:00 (a start is kept only while `start + d` is not later than `end_local` on the wall clock). Both dates are Sundays, ISO weekday 7. The first occurrence of an ambiguous hour is the only one offered.

## 10. Errors and limits

### 10.1 Error codes

`validation_failed` 400 · `unauthenticated` 401 · `forbidden` 403 · `not_found` 404 · `invalid_transition` 409 · `slot_taken` 409 · `idempotency_conflict` 409 · `practitioner_exists` 409 · `already_reviewed` 409 · `already_replied` 409 ·
`slot_unavailable` 422 · `offering_inactive` 422 · `own_offering` 422 · `practitioner_not_bookable` 422 · `submit_incomplete` 422 · `too_many_offerings` 422 · `contact_details_not_allowed` 422 · `github_not_linked` 422 · `not_reviewable` 422 · `invalid_input` 422 ·
`rate_limited` 429 · `assistant_unavailable` 503 · `dependency_unavailable` 503. A foreign origin is `403 forbidden` with code `cross_site_request`.

### 10.2 Rate limits (per instance, sliding window; the key is the user id when signed in, else the client address)

| Endpoint group | Limit |
|---|---|
| Public reads (`GET` practitioners, taxonomy, slots, reviews) | 120 per minute |
| `POST /api/v1/bookings` | 10 per minute and 30 per day per user |
| `POST /api/v1/assistant/brief` | 5 per minute and 20 per day per user |
| `POST /api/v1/reviews` | 5 per hour per user |
| Webhooks | none by address (signature decides); body limit 64 KB |
| Everything else | 60 per minute |

### 10.3 Configuration keys (environment variables use `__`)

| Service | Key | Default |
|---|---|---|
| all | `ConnectionStrings:<db>`, `Database:Provider` (`PostgreSQL`\|`InMemory`), `Identity:Authority` (authservice URL), `Identity:Audience`, `Internal:Key` (callee) and `Internal:Keys:<callee>` (callers) | InMemory when no connection string |
| Directory | `Directory:MeetingHosts` | list of §3.5 |
| Booking | `Billing:FirstConsultationFeeBps` 1500 · `Booking:MinLeadHours` 12 · `Booking:HoldMinutes` 10 · `Booking:BufferMinutes` 15 · `Booking:CompleteAfterHours` 48 · `Payments:Provider` (`Mock`\|`<vendor>`) `Mock` · `Payments:WebhookSecret` (secret) · `Notifications:Provider` (`Log`\|`Smtp`) `Log` |
| Reputation | `Reviews:BlockedTerms` (list) · `Reviews:WindowDays` 14 |
| Assistant | `Llm:Provider` (`Copilot`\|`Fake`) · `Copilot:*` and `Arena`-style custom model keys, see `COPILOT-SDK.md` §6 · `Assistant:RequireApiKeyModelsForTraffic` true · `Assistant:DailyBudgetUsd` 5 · `Assistant:BriefTimeoutSeconds` 30 · `Assistant:TriageTimeoutSeconds` 25 |
| Web | `Backend:Directory`, `Backend:Booking`, `Backend:Reputation`, `Backend:Assistant`, `Backend:Auth` (candidate lists), `Site:BaseUrl` |
| fake profile | `GABINETY_FAKE=1` sets `Llm:Provider=Fake`, `Database:Provider=InMemory`, `Payments:Provider=Mock`, `Notifications:Provider=Log`, `Identity:Authority` = the DevIdentity URL |

### 10.4 Endpoints that exist only outside a deployed environment

Mapped only when `GABINETY_FAKE=1` (fake profile) or `Environment:Kind` is `Development`; in staging and production they are not mapped (404), and an integration test asserts it. They sit under `/internal/` and need the internal key.

| Method, path | Service | Effect |
|---|---|---|
| `POST /internal/v1/test/clock` | Booking, Reputation | `{ offsetMinutes }` sets a signed offset added to `TimeProvider.GetUtcNow()` of that service so end-to-end journeys can pass the 24 h, 48 h and 14 day windows. The jobs run on the offset time. Reset by `{ offsetMinutes: 0 }` |
| `POST /internal/v1/demo/reset` | Directory, Booking, Reputation, Assistant | removes every row whose id, slug or user is in the namespace `demo-`; returns the counts. Never touches other rows |

