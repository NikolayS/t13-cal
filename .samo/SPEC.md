# t13 — FOSS Calendly — SPEC v0.2

## Goal & why it's needed

Build an open-source, self-hostable scheduling tool that lets an individual share a personal booking link, have invitees pick a slot that respects the host's real-time Google Calendar availability, and have the resulting meeting appear on the host's Google Calendar with a Zoom or Google Meet link automatically attached.

Why this exists: Calendly is the de facto standard for 1:1 booking, but it is closed-source SaaS, charges per host, and forces users to trust a third party with calendar OAuth tokens and meeting metadata in perpetuity. Solopreneurs and individual consumers want the same frictionless booking experience without the subscription, without vendor lock-in, and with the option to host the service themselves so their calendar data never leaves infrastructure they control. Existing FOSS alternatives (e.g. Cal.com) are powerful but heavy, multi-tenant, and complex to deploy; v0.1 of t13 is deliberately minimal — single-host, Google-only, two meeting providers — so it can be `docker run`-ed in under five minutes by a non-expert and produce a working booking page.

The core promise of v0.1: "a public link where someone can book a 30-minute call with me, and the call shows up on my Google Calendar with a Zoom or Meet link, and double-bookings are impossible." Nothing more, nothing less.

## Non-goals (v0.1)

Explicitly OUT of scope, called out so reviewers do not re-introduce them:

- Offline support (host's confirmed scope-boundary answer). The booking page requires connectivity; the server requires connectivity to Google's APIs at request time.
- Multi-tenant SaaS. v0.1 is single-host: one operator, one Google account, one booking page. No team scheduling, no round-robin, no organization accounts.
- Calendar providers other than Google (no Outlook/Microsoft 365, no iCloud, no CalDAV).
- Meeting providers other than Zoom and Google Meet (no Teams, no Webex, no Jitsi).
- Payments, paid bookings, or any billing.
- Native mobile apps. The booking page must be mobile-responsive web; that is the extent of mobile support.
- AI/LLM features of any kind.
- White-labeling, custom domains beyond what a reverse proxy gives for free, or theming beyond a single accent color.
- Workflow automations, SMS reminders, email sequences. v0.1 sends exactly two transactional emails: confirmation and cancellation (plus one combined "rescheduled" email on reschedule).

These are deferred, not forbidden forever — but if a reviewer's finding pulls one of them into v0.1, the verdict is `deferred` by default.

## Target user & success metric

**Primary user (host)**: an individual consumer or solopreneur (consultant, coach, freelancer, indie founder) who already lives in Google Calendar and currently either pays for Calendly or manually negotiates times over email. They are technically comfortable enough to follow a README and run `docker compose up`, or to click "Deploy to Fly.io" — but they are not a sysadmin.

**Primary user (invitee)**: anyone with a web browser and an email address. The invitee never logs in, never installs anything, never creates an account.

**Success metric (90 days post-launch)**: the host's qualitative judgment — "scheduling calls is easy." Concretely, we will treat v0.1 as successful if, within 90 days of first deploy, the host has accepted at least 10 real bookings through the link without manually editing a calendar event to fix t13's output, and reports that they would not go back to Calendly. Quantitative proxies we will track in self-hosted analytics (opt-in, off by default — see Analytics event schema below):

- Booking funnel completion: of invitees who load the page, ≥60% pick a slot and submit.
- Zero double-bookings caused by t13 (must be zero, not "low").
- Median time from `page_view` to `booking_confirmed` server-emitted event: <60s.

### Analytics event schema (opt-in)

When `T13_ANALYTICS=1`, t13 writes one row per event to a local `analytics_event` table. No external network egress; no IDs that identify the invitee beyond a per-session opaque token rotated every 24h. Schema is the contract the success metrics are computed against:

```
analytics_event(
  id, session_token, event_type_id,
  name,            -- enum: page_view | slot_selected | submit_attempted
                   --       | booking_confirmed | booking_failed_409
                   --       | reschedule_started | reschedule_confirmed
                   --       | cancel_confirmed
  at_utc,          -- server-side timestamp
  client_at,       -- optional, browser-reported, NULL if JS disabled
  outcome,         -- success | conflict_409 | provider_error | abandoned
  meta_json        -- {duration_min, meeting_provider, invitee_tz}
)
```

A single SQL view `funnel_v` joins on `session_token` to compute completion rate, p50/p95 page-view-to-confirmed latency, and 409 rate. Without these events the success metrics are unmeasurable, so the schema is normative, not optional, when analytics are turned on.

## User stories

1. **Host first-time setup.** As a solopreneur who just `docker run`-ed t13 on a $5 VPS, I open `https://my-domain/admin`, click "Connect Google Calendar," complete OAuth, define one event type ("30-min intro call, weekdays 9am–5pm America/Los_Angeles, 15-min buffer, Zoom"), and copy my public booking link — all in under 5 minutes, without reading documentation beyond the README.

2. **Invitee books a call.** As a prospective client who received the host's link, I open it on my phone, see the next two weeks of available 30-minute slots in my own timezone (auto-detected, overrideable), tap a slot, enter my name and email, optionally add a note, and tap "Confirm." Within 10 seconds I see an in-page confirmation containing the meeting time in my timezone, a Zoom join link, and a one-click cancel/reschedule link; the email arrives via SMTP best-effort thereafter (see Email SLO clarification). The host sees the event on their Google Calendar within the same 10 seconds.

3. **Conflict avoidance under contention.** As a host whose calendar is constantly changing, I trust that if I accept a meeting on my phone at 2:00:00pm and an invitee is mid-booking for 2:00pm at the same instant, exactly one of those two outcomes occurs: either the invitee sees the slot vanish before they click confirm, or their confirmation fails cleanly with "that slot was just taken, please pick another" — never a double-booking on my calendar.

4. **Invitee reschedules without contacting the host.** As an invitee whose schedule changed, I click the "Reschedule" link in my confirmation email, pick a new slot from the same availability page (with my old slot freed), and confirm. The original Google Calendar event is updated in place (same event ID, same Zoom/Meet link), and both parties get a single "rescheduled" email — not a cancel + new-booking pair.

5. **Host revokes and migrates.** As a privacy-conscious host who wants to leave a hosted t13 instance and move to my own server, I click "Export" in admin, download a JSON dump of my event types, past bookings, and an audit log, then click "Disconnect Google." t13 deletes its stored OAuth refresh token, and I can confirm in my Google Account's third-party access page that the grant is gone. I import the JSON into a fresh t13 instance and my booking link works on the new host (with a one-time re-OAuth).

## Architecture

<!-- architecture:begin -->

```text
(architecture not yet specified)
```

<!-- architecture:end -->

### Components

The canonical diagram above is the architecture. Two background workers are first-class and run inside the same process: the **reservation sweeper** (expires `reserved` rows past `expires_at`, every 30s) and the **sync worker** (consumes Google Calendar push notifications via a webhook endpoint and falls back to a 5-minute poll per active host).

### Boundaries & key abstractions

- **Domain core** is pure: no network, no clock except an injected `Clock`, no DB. It owns `EventType`, `AvailabilityWindow`, `Booking`, and the algorithms `SlotEnumerator` and `ConflictDetector`. Unit-testable with no fixtures beyond in-memory inputs.
- **Repository** is an interface. Default impl is SQLite (file-backed, zero-config); Postgres impl is opt-in via `DATABASE_URL`. The domain never imports SQL.
- **Provider gateway** is an interface per external system: `CalendarProvider` (impl: `GoogleCalendarProvider`), `MeetingProvider` (impls: `GoogleMeetProvider`, `ZoomProvider`), `Mailer` (impl: SMTP). All four are mockable in tests.
- **Booking service** orchestrates the write path: validate → check availability → reserve slot in DB → create calendar event → create meeting link → send emails → commit. Compensating actions on failure (see Implementation details).
- **Availability engine** is the read path: given an `EventType` and a date range, it returns the set of free slots by intersecting (a) the event type's recurring rules, (b) the host's Google Calendar busy times, (c) currently-held DB reservations, (d) the host's date-overrides (blackouts/extra availability).
- **HTTP layer** is thin: parse, authenticate, rate-limit, hand to a service, render. No business logic.

### Tech stack (decided for v0.1)

- **Language**: TypeScript on Node.js 20 LTS. Chosen for hiring breadth among solo-founder users who may want to extend, and for first-class Google API client libraries.
- **Web framework**: Fastify (lighter and faster than Express, native schema validation).
- **Frontend**: Server-rendered HTML via Fastify + a tiny vanilla-TS booking widget (no React/Vue in v0.1 — page must work with JS disabled to the no-JS contract defined below, and full booking with ~10KB of JS). Admin uses Preact for a slightly richer SPA-lite (~30KB).
- **DB**: SQLite via `better-sqlite3` by default; Postgres via `pg` if `DATABASE_URL` is set. Schema migrations via `node-pg-migrate`-style numbered SQL files runnable against both.
- **Distribution**: single Docker image (multi-stage, <150MB), `docker-compose.yml` example, and a one-click Fly.io / Railway template.

### No-JS contract (booking page)

When the invitee has JavaScript disabled, the booking page must:

- Render the next 14 days of slot starts as a static HTML list grouped by day, in the host's timezone with the timezone label visible (no auto-detection without JS — auto-detection requires JS).
- Render a `<noscript>` banner: "Your browser has JavaScript disabled. Slots below show in <host_tz>. To complete a booking, please enable JavaScript or contact the host directly at <host_email>."
- The form submit button is rendered as `disabled` with `aria-disabled="true"`.
- All slot links (`<a href=...>`) point to the same SSR page anchored to that slot, so a screen-reader user can navigate but cannot submit.

Frontend tests assert the noscript banner is present in initial HTML, that the submit button has `disabled` in initial markup, and that JS removes the disabled state on hydrate.

### Data model

```
user            (id, email, google_sub, google_refresh_token_enc, tz,
                 zoom_account_id_enc, zoom_client_id_enc, zoom_client_secret_enc,
                 created_at)
event_type      (id, user_id, slug, name, duration_min, slot_interval_min,
                 buffer_before_min, buffer_after_min, min_notice_min,
                 max_horizon_days, meeting_provider, color, active, created_at)
availability_rule (id, event_type_id, weekday, start_min, end_min)  -- recurring
date_override   (id, event_type_id, date, kind, start_min, end_min) -- block/extra
booking         (id, event_type_id, predecessor_id, invitee_name, invitee_email,
                 invitee_note, start_at_utc, end_at_utc, invitee_tz, status,
                 google_event_id, meeting_url, cancel_token,
                 expires_at, created_at, updated_at)
booking_audit   (id, booking_id, action, actor, payload_json, at)
oauth_state     (state_token, pkce_verifier, expires_at)
analytics_event (see Analytics event schema)
```

`predecessor_id` is a self-FK on `booking` and is non-null on rows produced by reschedule. `slot_interval_min` defaults to `duration_min` if NULL (preserving the simple-case behavior); see Slot alignment below. The three Zoom-credential columns are encrypted with the same AES-256-GCM scheme as `google_refresh_token_enc` (see Security § Zoom credential storage).

Indexes: `booking(event_type_id, start_at_utc, status)` for conflict checks; one **per-user** unique partial index, defined exactly once and used everywhere:

```sql
CREATE UNIQUE INDEX booking_user_slot_live
  ON booking (user_id, start_at_utc)
  WHERE status IN ('reserved', 'confirmed');
```

`user_id` is denormalized onto `booking` (FK to `user`) so the index can be enforced across all event types of the same host. This index is the **last line of defense** against double-booking and intentionally also blocks a confirmed write while a sibling reservation is alive — see Conflict resolution semantics for why that's intentional. SQLite supports partial indexes; Postgres supports them; both also support including `user_id` so the cross-event-type case (Reviewer B finding) is enforced at the DB.

## Implementation details

### Host timezone source of truth

`user.tz` is a cached copy of the host's Google Calendar setting (the `summary.timeZone` from `calendars.get('primary')`), refreshed on every successful `freebusy.query` response (Google returns `timeZone` in the request echo) and on each admin login. If `user.tz` differs from Google's reported timezone, t13 logs a warning, updates `user.tz`, and **does not** re-anchor existing reserved/confirmed bookings — they keep their stored UTC instants. New availability rendering uses the new tz immediately. The host is shown a one-time admin banner: "Your Google Calendar timezone changed from X to Y; future slots will use Y. Existing bookings are unchanged." This makes Google the source of truth and pins the drift behavior.

### Slot alignment

Slot starts within an availability window are aligned to the window's `start_min` and stepped by `event_type.slot_interval_min` (default = `duration_min`). For a 45-min duration with `slot_interval_min=45` and rule 09:00–17:00, slots are `{09:00, 09:45, 10:30, 11:15, 12:00, 12:45, 13:30, 14:15, 15:00, 15:45}`; `16:30` is dropped because `16:30 + 45 > 17:00`. A `date_override` with `kind='extra'` defines its own window with the same alignment rule (anchored to the override's `start_min`, stepped by `slot_interval_min`). v0.1 does not expose `slot_interval_min` in admin — it is set equal to `duration_min` on event-type creation — but the column exists so the algorithm and tests can be written against a precise rule and v0.2 can expose it.

### All-day event semantics

Google's all-day events use exclusive end dates (a 3-day vacation `2026-07-01` → `2026-07-04` means days 1, 2, 3 are out). t13 honors this exactly: `[start_date 00:00 host_tz, end_date 00:00 host_tz)`, i.e. the end date is **not** included. Tests cover the boundary explicitly with a fixture for a 1-day, 2-day, and 3-day all-day event.

### Availability computation

Given an `EventType` E, a date range `[from, to]` in the host's timezone, and "now":

1. Enumerate candidate slot starts by iterating each day in `[from, to]`, applying E's `availability_rule` rows for that weekday, slicing into `duration_min`-long slots stepped by `slot_interval_min` anchored to each window's start (see Slot alignment).
2. Apply `date_override` rows: `kind='block'` removes intersecting candidates; `kind='extra'` adds candidates outside recurring rules using the same alignment rule.
3. Filter by `min_notice_min` (e.g. "no slots starting in the next 4 hours") and `max_horizon_days`.
4. Subtract Google Calendar busy intervals fetched via `freebusy.query` for the host's primary calendar over `[from, to]` (single API call covering the whole window). Apply `buffer_before_min` and `buffer_after_min` to busy intervals when subtracting. All-day events use exclusive-end semantics.
5. Subtract DB-confirmed bookings for E **and any other event types of the same user** (a host's bookings on event type A also block event type B). This is enforced both at the application layer (this step) and at the DB layer (the per-user unique partial index above).
6. Subtract DB-held *reservations* (status='reserved', not yet expired) — held slots are unavailable to other invitees.
7. Convert to invitee's timezone for rendering.

The `freebusy.query` response is cached in-process for 30 seconds keyed by `(user_id, from, to)` to absorb burst loads on a popular booking page without exhausting the Google API quota.

### Booking write path (transactional, with compensation)

The write path must satisfy: **no double-booking on the host's calendar, ever, even under concurrent invitee submissions and concurrent host-side calendar edits.** The strategy is "reserve in our DB first, then write to Google, then confirm — with compensation on every failure point."

```
1. POST /book {event_type_id, start_at, invitee_*}
2. BEGIN TX (SERIALIZABLE on PG; default on SQLite is serial)
3.   Re-run availability check for the exact requested slot (cheap: single conflict query).
4.   If conflict → ROLLBACK, 409 "slot just taken."
5.   INSERT booking row with status='reserved', expires_at=now+90s.
      → If the per-user unique partial index trips here, ROLLBACK, 409.
6. COMMIT.  ── slot is now ours; concurrent invitees see it as held.
7. Final-slot freebusy double-check (uncached, single targeted Google call, ~80ms).
      → If busy, mark booking 'failed' (releasing the slot via partial-index predicate),
        return 409.
8. Call GoogleCalendar.events.insert (with conferenceData if Meet).
9. If 4xx/5xx → mark booking 'failed', return 502 with retry advice.
10. If Zoom event type: call Zoom REST /users/me/meetings using decrypted Zoom creds.
11. If Zoom call fails: DELETE the Google event (compensation), mark booking 'failed', return 502.
12. UPDATE booking SET status='confirmed', google_event_id=…, meeting_url=…, expires_at=NULL.
13. Enqueue confirmation email (best-effort; failure logged, does not fail the booking).
14. 200 OK with booking detail + cancel/reschedule link rendered in-page.
```

On step 7's failure, the row is moved to `status='failed'` so the unique partial index (which only covers `reserved`/`confirmed`) releases the slot immediately for other invitees.

A background sweeper every 30s expires `status='reserved'` rows past `expires_at` to `status='failed'`, freeing the slot.

### Reservation visibility & abandon behavior

- While a booking is `reserved`, the slot is **invisible** to other invitees on the booking page (subtracted in availability step 6) and a concurrent submission for the same `(user_id, start_at_utc)` is rejected with 409 by the unique partial index.
- On the booking page, the client uses `navigator.sendBeacon` to fire a best-effort `POST /book/release` with the reservation's cancel_token if the page is closed/navigated before confirm; the server moves the row to `failed`. This is best-effort; correctness still relies on the 90s `expires_at` + 30s sweeper. Worst-case visible hold for an abandoned invitee is ~120s, documented in admin.

### Conflict resolution semantics (the host's hardest constraint)

The host called out "integration with google calendar, conflict resolution" as the hardest constraint. v0.1 commits to these explicit rules:

- **Source of truth for host availability is Google Calendar.** Any event on the host's primary calendar with `transparency != 'transparent'` is busy. All-day events are busy with exclusive-end semantics.
- **Buffers apply only to confirmed bookings and freebusy results, not to tentative invites the host hasn't accepted.** Tentative invites are treated as busy (matching Calendly's default) but without buffer.
- **Race between host-accepts-meeting and invitee-books-same-slot**: the freebusy cache is 30s; in the worst case an invitee can confirm a slot that became busy in the last 30s. To close this: step 7 above performs a final freebusy check for the *single requested slot only* (one targeted API call, ~80ms) before calling `events.insert`. If that final check fails, we move the reservation to `failed` and return 409. This is the only mandatory uncached Google call per booking.
- **Cross-event-type same-host bookings**: the unique partial index is keyed on `(user_id, start_at_utc)`, not `(event_type_id, start_at_utc)`. A host with event types "30-min intro" and "60-min deep dive" cannot have two bookings starting at 2:00pm via concurrent submissions: one will trip the index, get a clean 409, and be rolled back.
- **If the host edits/deletes the Google event we created**, t13 detects this on next sync (push notification via Google Calendar `watch` if available, polling every 5min as fallback) and updates the local booking to `status='cancelled_by_host'`, sending a cancellation email to the invitee. Sync correctness is test-covered (see Tests plan § Sync path tests).

### State transitions for `booking`

```
        ┌───────────┐  expires_at hit / explicit release / step 7 fails
        │ reserved  │ ──────────────────────────────────────────────► failed
        └─────┬─────┘
              │ google+meeting succeed (step 12)
              ▼
        ┌───────────┐
        │ confirmed │
        └─────┬─────┘
     ┌────────┼────────┬──────────────┐
     │        │        │              │
     │ invitee│ host   │ time passes  │ invitee uses
     │ cancels│ cancels│              │ reschedule link
     ▼        ▼        ▼              ▼
┌─────────┐ ┌────────────────┐ ┌─────────┐ ┌───────────────┐
│cancelled│ │cancelled_by_   │ │completed│ │ rescheduled   │
│_by_     │ │host            │ │         │ │ (new row in   │
│invitee  │ └────────────────┘ └─────────┘ │  'confirmed'  │
└─────────┘                                │  with         │
                                           │ predecessor_id│
                                           │  pointing to  │
                                           │  this row;    │
                                           │  this row →   │
                                           │ 'rescheduled')│
                                           └───────────────┘
```

On reschedule the original row transitions to `rescheduled` and the Google event is updated in place (same `google_event_id`, same `meeting_url`); a single "rescheduled" email is sent.

### Google OAuth & token storage

- Authorization Code flow with PKCE. Scopes: `calendar.events`, `calendar.readonly` (for freebusy), `userinfo.email`. **Not** `calendar` full scope.
- Refresh tokens are encrypted at rest with AES-256-GCM using a key derived from the operator's `T13_SECRET_KEY` env var (required at startup; t13 refuses to start without it). The DB never sees the plaintext refresh token.
- Access tokens are held in process memory only.
- `/admin/disconnect` revokes the token via `oauth2.revoke` and deletes the encrypted blob.

### Zoom credential storage

Zoom Server-to-Server OAuth (account-level credentials) is the supported v0.1 mode — the host enters Zoom **Account ID, Client ID, Client Secret** in admin once. All three are encrypted at rest with the same AES-256-GCM scheme keyed off `T13_SECRET_KEY` and stored in the columns `zoom_account_id_enc`, `zoom_client_id_enc`, `zoom_client_secret_enc` on `user`. The plaintext is held in process memory only after decryption, only for the duration of a Zoom API call. Rotation: admin shows the masked Account ID and last 4 of Client ID; "Rotate Zoom credentials" replaces all three values atomically (single transaction). Revocation: "Disconnect Zoom" zeroes all three columns and switches all event types whose `meeting_provider='zoom'` to a banner "Zoom disconnected — bookings disabled for this event type" until reconnection or provider switch. User-level Zoom OAuth is deferred (it complicates self-hosting because each redirect URI must be registered in the Zoom Marketplace).

### Emails & .ics

SMTP-only in v0.1. Operator provides SMTP creds via env. Three templates: `confirmation.html`, `cancellation.html`, `rescheduled.html`. The `.ics` attachment is generated locally (no library that pulls in 50 deps — small hand-written generator), VEVENT with ORGANIZER, ATTENDEE, UID = `t13-{booking_id}@{host}`, METHOD:REQUEST for confirm/reschedule, METHOD:CANCEL for cancellations. The generator is RFC 5545 compliant on:

- Line folding at ≤75 octets (UTF-8-aware).
- TEXT-property escaping: `,` → `\,`, `;` → `\;`, `\` → `\\`, `\n` → `\n` literal.
- Non-ASCII characters in invitee_name / invitee_note encoded as UTF-8 with proper folding on octet boundaries.
- METHOD:CANCEL emits matching UID + SEQUENCE incremented.

Golden fixtures cover all four corners (UTC, host-tz, METHOD:REQUEST, METHOD:CANCEL) plus a non-ASCII invitee name and a comma/semicolon-laden note.

### Email SLO clarification

The write path returns 200 with the booking detail and the meeting URL **rendered in-page** within the 10-second target of user story 2. SMTP delivery is best-effort and outside t13's control beyond enqueue latency; the spec promises only that the confirmation email is enqueued before the 200 is returned. The user story copy reflects this: in-page confirmation is guaranteed within 10s, email arrival is best-effort.

### Security

- CSRF tokens on all POST endpoints under `/admin`. The public `/book` endpoint is CSRF-exempt by necessity (cross-origin links must work) but is rate-limited to 10 booking attempts per IP per hour and 60 availability queries per IP per minute.
- Cancel/reschedule tokens are 32-byte URL-safe random, single-purpose, scoped to one booking. Not guessable, not reusable.
- Admin auth: passwordless email-magic-link in v0.1 (the host's own Google email is the only allowed account). No multi-user, no roles.
- All secrets via env, never in DB plaintext. `.env.example` documents every var. All host-supplied third-party credentials (Google refresh token, Zoom triplet) are encrypted with AES-256-GCM keyed off `T13_SECRET_KEY`.
- HTTPS is the operator's responsibility (reverse proxy); t13 sets `Strict-Transport-Security` and `Secure` cookies when `T13_PUBLIC_URL` starts with `https://`.

### Observability

Structured JSON logs to stdout. One Prometheus-format `/metrics` endpoint (off by default, enable with `T13_METRICS=1`) exposing booking counts, conflict-409 rate, Google API latency, freebusy cache hit ratio, and sync-worker lag. No third-party telemetry, no phone-home.

## Tests plan

### Test pyramid

- **Unit tests (TDD, red→green required)** — these pieces are built test-first, no exceptions:
  - `SlotEnumerator`: given rules + overrides + a date range + a clock, produces the expected slot set. Edge cases: DST spring-forward (a 9am–5pm Sunday in America/Los_Angeles on the spring-forward day yields one fewer slot), DST fall-back (one extra), invitee in a half-hour-offset tz (Asia/Kolkata), event type spanning midnight host-tz, leap year Feb 29, non-default `slot_interval_min` alignment.
  - `ConflictDetector`: given a candidate slot and a list of busy intervals (open/closed boundary semantics — `[09:00, 09:30)` and `[09:30, 10:00)` do **not** conflict), returns conflict yes/no and reason. Buffer math is part of this module. **Property-based tests** (fast-check) generate arbitrary rules + busy-interval sets and assert invariants: (i) no returned free-slot overlaps any busy interval after buffer, (ii) idempotence of the function, (iii) monotonicity (adding a busy interval never adds a free slot).
  - `BookingService.book` happy path and every compensation path (Google fails, Zoom fails, both succeed, double-submit, step-7 final-check fails). Use mocked providers. **Branch coverage required ≥95% on this module specifically** — see Coverage gate.
  - `.ics` generator: byte-for-byte golden file comparison against six fixtures: UTC, host-tz, all-day-not-applicable, METHOD:CANCEL, non-ASCII invitee name (Cyrillic + emoji), and a note containing comma/semicolon/backslash/newline.
  - OAuth token encryption round-trip; Zoom-credential triplet encryption round-trip with rotation atomicity.
  - All-day-event boundary (1/2/3-day vacations).

- **Integration tests (CI, against real SQLite + recorded HTTP fixtures)**:
  - Full booking flow: HTTP POST → DB row → mocked Google call → mocked Zoom call → email captured by in-memory SMTP.
  - Reservation expiry sweeper.
  - Reschedule path preserves Google event ID; predecessor_id linked correctly.
  - Postgres parity: same suite runs against a Postgres container (`testcontainers`).
  - **DB unique-index proof, single event type**: deliberately bypass the application conflict check and assert the second insert raises a constraint error.
  - **DB unique-index proof, cross event type**: with two event types A and B owned by the same user, deliberately bypass the application subtraction step and assert that two concurrent confirm-writes for the same `(user_id, start_at_utc)` produce exactly one success and one constraint-violation 409. This is the regression test for the Reviewer B finding on the per-user index.

- **Sync path tests (new, for cancelled_by_host correctness)**:
  - Unit: Google Calendar `watch` webhook payload parser — given a representative push payload, emits the right `(user_id, channel_id, resource_id)` event.
  - Unit: poll-fallback diff — given a previous and current state of the host's events, emits `cancelled_by_host` for deleted events that t13 had created and `host_edited` for time/duration changes.
  - Integration: a fixture mutates a Google event t13 created (delete, time-change, duration-change, all-day flip), runs the sync worker, and asserts (i) the booking transitions to `cancelled_by_host`, (ii) a cancellation email is queued, (iii) booking_audit gets the action row. Both push and poll paths run.

- **Contract tests against Google APIs**:
  - `nock`-based fixtures recorded once from a sandbox Google account; the test suite replays them on every PR.
  - A separate `npm run test:live` job hits real Google + real Zoom with dedicated test accounts to detect API drift. Runs **nightly** in CI on a schedule and is **a required check on the release-tag pipeline** — a failing live run blocks tagging v0.1.0 (and all subsequent releases). On-PR latency cost and flake risk are absorbed by the nightly cadence; the release-gate ensures no v0.x.y ships without a green live run within the prior 24h.

- **End-to-end browser tests** (Playwright, headless Chromium): host-onboards → invitee-books → cancels. Run on every PR. The Google and Zoom backends are stubbed at the network layer.

- **Accessibility tests**: `axe-core` integrated into the Playwright suite; runs against the booking page (with and without JS), the admin onboarding page, the admin event-type-list page, and the admin booking-list page. **Zero serious or critical violations** are required to pass; this is a CI required check on every PR. The no-JS contract is asserted explicitly (noscript banner present, submit button has `disabled` and `aria-disabled` in initial markup).

- **Concurrency stress test**: a script fires 50 concurrent booking requests for the same slot; assertion: exactly one succeeds with 200, the other 49 receive 409, zero Google events were created beyond the one. A second variant fires 50 concurrent requests across two different event types of the same host at the same start time; assertion: exactly one succeeds, 49 receive 409, zero double-bookings on Google. Both run in CI on every PR.

- **Timezone matrix test**: a parameterized test runs the booking flow for every pair drawn from `{America/Los_Angeles, Europe/Berlin, Asia/Kolkata, Pacific/Chatham, UTC}` × `{standard time, DST transition day, all-day events present}` and asserts the start time stored in DB and rendered to invitee match expectation.

- **Host-tz drift test**: simulate `calendars.get('primary')` returning a different `summary.timeZone`; assert `user.tz` updates, the admin banner is shown once, and existing reserved/confirmed bookings keep their stored UTC instants.

### Explicit red→green TDD callout

The following modules **must** be developed with a failing test committed before the implementation, and the commit history must show this:

1. `SlotEnumerator` (domain core, the heart of correctness).
2. `ConflictDetector` (the buffer + boundary semantics are easy to get wrong; property tests included).
3. `BookingService` compensation paths (failure modes are otherwise impossible to trigger reliably; tests are the only reasonable spec).
4. `.ics` generator (RFC 5545 compliance is bug-prone; golden files are the spec).
5. Sync worker payload parser + diff logic (the only thing that fulfills the host-edit detection promise).

Other modules (HTTP routes, the admin UI, the Postgres repository) may be developed test-after.

### CI required checks (must pass to merge)

- Lint (`eslint`, `prettier --check`).
- Typecheck (`tsc --noEmit`).
- Unit + integration suite on Node 20 LTS, both SQLite and Postgres.
- Playwright E2E suite.
- **axe-core accessibility suite (zero serious/critical)** — every PR.
- Concurrency stress test (single-event-type and cross-event-type variants).
- Sync-path integration test (push and poll).
- Docker image build + smoke boot (image starts, `/healthz` returns 200).
- `npm audit --omit=dev` with zero high/critical.
- **Coverage gate**:
  - Domain core (`SlotEnumerator`, `ConflictDetector`): ≥85% line coverage.
  - `BookingService`: ≥95% branch coverage (branch coverage, not line — every compensation path must be hit).
  - `.ics` generator: ≥95% line coverage.
  - Provider gateways (`GoogleCalendarProvider`, `ZoomProvider`, `Mailer`): ≥80% line coverage.
  - Sync worker (parser + diff): ≥90% line coverage.
  - No gate elsewhere.

### Release-tag pipeline (additional checks beyond per-PR)

A git tag matching `v*.*.*` triggers a separate pipeline that, in addition to the PR checks, requires:

- A green `test:live` run against real Google + Zoom within the prior 24h (this typically means the nightly run from the night before; a failed nightly blocks the tag until rerun-and-green).
- Manual checklist sign-off (see below).
- Security review sign-off recorded in the release notes.

### Manual test checklist (mapped to user stories)

One human pass before tagging v0.1.0, exercising each of the five user stories above end-to-end on a real Google account and real Zoom account, on desktop Chrome, desktop Safari, mobile Safari, and mobile Chrome.

## Team

Veteran experts to hire (hire for the duration of v0.1, ~6–8 weeks):

- Veteran backend engineer, Node.js + TypeScript + Fastify + SQL (1) — owns domain core, booking service, repositories, migrations.
- Veteran calendar/scheduling domain expert (1) — owns availability semantics, timezone correctness, Google Calendar API integration, sync worker; ideally has shipped a calendar product before.
- Veteran frontend engineer, vanilla-TS + Preact, accessibility-first (1) — owns booking page (must be fast, mobile-first, WCAG AA, no-JS contract) and admin UI.
- Veteran SRE / DevOps engineer (1) — owns Dockerfile, docker-compose, Fly.io template, CI pipeline, release engineering, observability.
- Veteran security engineer, OAuth + token storage + web app sec (1) — reviews OAuth flows, token + Zoom-credential encryption, CSRF/rate-limit posture, performs threat model + pentest before v0.1.0 tag.
- Veteran QA engineer, Playwright + property-based testing (fast-check) + axe-core (1) — owns the timezone matrix test, ConflictDetector property tests, concurrency stress tests (both variants), sync-path tests, accessibility suite, manual checklist, and live-API drift suite.
- Veteran technical writer / DX (0.5, part-time) — owns README, deploy guides, troubleshooting docs; pairs with SRE on the "5-minute setup" experience.

Total: 6.5 FTE. The scheduling domain expert and security engineer are the two roles where hiring under-experienced people would cause shippable but wrong software, so those are non-negotiable seniority hires.

## Implementation plan

Six sprints, one week each, with deliberate parallelization. Sprint 0 is shared setup; sprints 1–5 ship in increments toward v0.1.0.

### Sprint 0 — Foundation (week 1)

All hands.

- SRE: repo skeleton, TypeScript build, ESLint/Prettier, Fastify hello-world, Dockerfile, GitHub Actions CI scaffold, SQLite + Postgres test containers, axe-core wired into Playwright, nightly `test:live` job scaffolded (initially no-op), release-tag pipeline scaffolded.
- Backend: domain core skeleton (empty `EventType`, `Booking` types), repository interface, in-memory repository for tests.
- Calendar expert: Google API client wired up with a sandbox account, freebusy + events.insert smoke test green.
- Frontend: design system decisions (single accent color, typography, spacing scale), booking-page wireframe approved, no-JS contract documented.
- Security: threat model document v0, env-var inventory, `T13_SECRET_KEY` derivation scheme decided, Zoom-credential storage design.
- QA: test plan approved, Playwright scaffolded, fast-check added.
- Writer: README v0 with promise statement.

Exit criteria: `docker run` starts a server that returns 200 on `/healthz`. CI is green on an empty test suite. Release-tag pipeline runs (no-op) on a fake tag.

### Sprint 1 — Domain core, TDD (week 2)

Parallelizable across two engineers; the calendar expert leads, backend pairs.

- Calendar expert + Backend (paired): `SlotEnumerator` red→green, `ConflictDetector` red→green (including fast-check property tests), all-day-event boundary tests, the full timezone/DST test matrix. **Nothing else this sprint.**
- Frontend: static booking-page HTML with mock data, mobile-first responsive, no-JS contract implemented and asserted in axe + DOM tests.
- SRE: CI matrix (Node 20, SQLite, Postgres). axe runs on every PR.
- Security: OAuth design doc reviewed, PKCE flow specified, Zoom-credential encryption module specified.
- QA: timezone matrix test harness built, runs against the new domain core; property tests for ConflictDetector.
- Writer: "How availability is computed" doc (will become the most-linked page).

Exit criteria: domain core is feature-complete and 100%-covered for the spec'd timezone matrix. Frontend renders against fake JSON and passes axe with no JS.

### Sprint 2 — Persistence, OAuth, booking happy path (week 3)

- Backend: SQLite + Postgres repositories, migrations including the per-user unique partial index, predecessor_id column, Zoom-credential columns; `BookingService.book` happy path test-first, `/book` HTTP endpoint behind it.
- Calendar expert: Google OAuth (PKCE), refresh-token storage hooked to Security's encryption module, freebusy integration, `events.insert` integration including `conferenceData` for Meet, host-tz cache-and-update logic.
- Security: AES-GCM encryption module shipped with round-trip tests for both Google refresh token and Zoom triplet; CSRF middleware; rate limiter.
- Frontend: live booking page wired to `/availability` and `/book` against a real backend.
- SRE: deploy preview environment per PR (Fly.io); axe runs against live deploy preview.
- QA: integration test suite green, single-event-type concurrency stress test red (will go green next sprint), cross-event-type stress test red (will go green next sprint).
- Writer: deploy-to-Fly guide draft.

Exit criteria: a developer can OAuth in, define an event type via DB seed, share the link, and a real invitee can book a Google Meet that appears on the calendar.

### Sprint 3 — Conflict correctness + Zoom + reschedule (week 4)

- Backend: reservation/compensation flow, the per-user unique partial index enforced end-to-end, reservation sweeper, all `BookingService` failure-path tests red→green, beacon-based release endpoint. Both concurrency stress tests go green here.
- Calendar expert: final-slot freebusy double-check, Zoom Server-to-Server provider with encrypted triplet storage and rotation, reschedule path (in-place Google event update with predecessor_id linking, single combined email), sync worker (push webhook + 5-min poll) with red→green parser + diff tests.
- Frontend: "slot just taken" 409 UX, reschedule page, cancel page, beacon release on navigate-away.
- Security: pen-test pass #1 against deploy-preview; rate-limit tuning; Zoom-credential rotation flow validated.
- QA: full E2E (host onboards → invitee books → invitee reschedules → invitee cancels) on Playwright, all four browsers; sync-path integration tests green; cross-event-type DB-unique-index proof green.
- Writer: troubleshooting guide.

Exit criteria: zero double-bookings under both stress-test variants. Both Meet and Zoom event types work end-to-end. Reschedule preserves event ID, meeting URL, and links predecessor_id. Sync worker transitions a host-deleted event to `cancelled_by_host` in tests.

### Sprint 4 — Admin UI + sync-in-prod + polish (week 5)

- Frontend: admin SPA-lite (Preact) — connect Google, define event types, list bookings, disconnect/export, Zoom-credential entry/rotation/disconnect, host-tz-changed banner. Magic-link login.
- Backend: export JSON endpoint, analytics_event table + funnel_v view (gated on `T13_ANALYTICS=1`).
- Calendar expert: edge cases on host-side edits in production (event moved, event deleted, event duration changed, all-day flip); poll-fallback validated against rate limits.
- Security: pen-test pass #2 (focused on admin surface and Zoom credentials); revocation flow validated against Google's third-party access page; Zoom disconnect zeroes all three columns.
- SRE: observability — `/metrics`, structured logs, healthchecks, sync-worker lag metric.
- QA: manual checklist dry-run on staging; bug bash with full team; axe sweep on admin pages.
- Writer: README finalized, screenshots, the "5-minute setup" tested by someone outside the team.

Exit criteria: a non-engineer can deploy and onboard in <5 minutes following only the README. All bugs filed in the bug bash are triaged. Admin axe is clean.

### Sprint 5 — Hardening + release (week 6)

- All hands: bug-fix only. No new features. Code freeze on day 3.
- Security: final review + threat-model sign-off.
- QA: manual checklist executed on real accounts across all four browsers; nightly `test:live` green for at least 3 consecutive nights before tag.
- SRE: build + sign + publish Docker image to GHCR; Fly.io and Railway templates published; release-tag pipeline gates on green `test:live` within 24h + manual + security sign-off.
- Writer: announcement post, CHANGELOG, GitHub release notes.
- Backend + Calendar expert: post-launch monitoring rota set up.

Exit criteria: v0.1.0 tagged, image published, announcement live. Success metric counters are live for 90-day evaluation.

### Parallelization map (who blocks whom)

```
Sprint:    0       1       2       3       4       5
Backend:   skel ─► core ─► repo ─► race ─► sync ─► fix
CalExp:    smoke─► core ─► oauth─► zoom ─► edge ─► fix
Frontend:  design─► fake─► live ─► UX  ─► admin─► fix
SRE:       CI  ─►  CI+  ─► PR-env ─► —  ─► obsv ─► release
Security:  TM  ─►  oauth─► enc  ─► pen1 ─► pen2 ─► signoff
QA:        plan─► tz mx─► integ─► E2E  ─► bash ─► manual
Writer:    READ─► avail─► deploy─► trouble─► full ─► launch
```

The critical path runs through Backend + Calendar expert. Frontend, SRE, Security, QA, and Writer can each absorb a slipped day from the critical path without blocking the release; one slipped day on the critical path slips the release one day. Sprint 3 is the highest-risk sprint (correctness under concurrency + sync correctness); we deliberately schedule it before admin/polish so a slip there doesn't cascade into surface-area work that's easier to truncate.

## Embedded Changelog

- v0.1 (2026-05-05) — Initial spec. Established goal/non-goals, five user stories, architecture (TS/Node/Fastify/SQLite-or-PG), data model, availability + booking algorithms with explicit conflict-resolution semantics, reserve→write→confirm flow with DB-level uniqueness as last-line defense, OAuth/security posture, full test plan with TDD callouts on domain core, 6.5-FTE team, six-sprint plan with parallelization map.
- v0.2 (2026-05-05) — Reviewer B pass. Reconciled unique-index predicate to a single per-user `(user_id, start_at_utc) WHERE status IN ('reserved','confirmed')` definition that covers cross-event-type same-host concurrency. Moved the architecture diagram inside the canonical `<!-- architecture:* -->` markers. Added `predecessor_id` to booking schema and `slot_interval_min` to event_type. Specified Zoom credential storage (three encrypted columns, rotation atomicity, disconnect zeroes). Defined the no-JS booking-page contract. Pinned host-tz source-of-truth to Google with admin-banner-on-drift behavior. Specified all-day exclusive-end semantics. Defined slot phase/offset alignment. Clarified email SLO (in-page confirmation guaranteed, SMTP best-effort). Documented reservation visibility + beacon-based release. Defined analytics_event schema for success metrics. Test plan: added cross-event-type DB-unique-index proof, sync-path tests (parser + diff + integration), property-based tests for ConflictDetector, axe on every PR, expanded .ics fixtures (line-folding, escaping, non-ASCII, METHOD:CANCEL, host-tz drift). Strengthened coverage gate (BookingService ≥95% branch, .ics ≥95% line, sync worker ≥90% line, providers ≥80% line). Made `test:live` nightly and a release-tag gate. Added sprint-level work for sync worker, Zoom-credential UI, and analytics schema.
