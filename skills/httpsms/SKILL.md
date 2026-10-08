# httpSMS Project Engineering Skill

## Purpose

This skill gives an AI coding agent the architectural and behavioral context needed to work safely and effectively in the httpSMS monorepo.

Repository: `lani772/httpsms`
Default branch: `main`
Primary product: httpSMS — an Android-phone SMS gateway exposed through an HTTP API.

Read this skill before modifying httpSMS code. Prefer repository source and generated API documentation over assumptions.

## Product model

httpSMS turns an Android phone/SIM into a programmable SMS/MMS gateway:

1. A user authenticates and registers a phone.
2. The Android app obtains/refreshes an FCM token and associates it with the phone/SIM.
3. An API client submits an SMS through the Go API.
4. The API persists the message and asynchronously triggers a push/event path.
5. The Android phone receives the command, fetches/handles the message, and uses Android telephony APIs to send it.
6. The Android app reports send/delivery events back to the API.
7. Incoming SMS/MMS and selected call events are captured by Android receivers/workers and forwarded to the API.
8. The web SPA exposes messages, threads, phone management, API keys, webhooks, schedules, billing, integrations and operational settings.

Core product guarantees/features include:
- programmable HTTP SMS sending and receiving
- Android phone/SIM gateway
- FCM-driven asynchronous delivery
- message/thread history
- webhooks
- optional end-to-end AES-256 message encryption
- configurable back pressure/rate limiting
- message expiration
- MMS attachments
- dual-SIM support
- phone heartbeats
- bulk SMS
- scheduled sending
- API/phone API keys
- Discord and 3CX integrations
- subscription/billing flows
- message search protected by Cloudflare Turnstile

## Repository architecture

### `android/`
Native Android application written in Kotlin with Jetpack Compose.

Important entry points:
- `MainActivity.kt`: main gateway lifecycle, permission requests, heartbeat scheduling, FCM token refresh, foreground listener service.
- `LoginActivity.kt`: login/API-key onboarding, QR scanning, Google Play Services validation and phone-number detection.
- `SettingsActivity.kt`: local Android gateway settings and logout.
- `ReceivedReceiver.kt`: receives Android SMS/MMS broadcasts, extracts content/attachments, applies optional encryption, queues network forwarding work.
- `services/StickyNotificationService.kt`: foreground listener service keeping the gateway active.
- Firebase Messaging service: handles push commands from the backend; locate current implementation before changing push behavior.
- UI/ViewModel packages: Compose screens and state management for main, login and settings.
- workers: background heartbeat, received-message forwarding and other resilient background work.
- `HttpSmsApiService`: Android-to-API HTTP boundary.
- `Settings`, `Constants`, encryption helpers and models form the local gateway state/configuration layer.

Android build characteristics:
- application id: `com.httpsms`
- min SDK 28
- compile/target SDK 37
- Kotlin 2.4.x
- Compose + Material 3
- Firebase Messaging
- OkHttp
- WorkManager
- ZXing QR scanning
- Android SMS/MMS APIs
- libphonenumber
- Timber logging
- Sentry integration

Critical Android permissions include SEND_SMS, RECEIVE_SMS, READ_SMS, READ_PHONE_STATE, POST_NOTIFICATIONS, RECEIVE_MMS, INTERNET, ACCESS_NETWORK_STATE, FOREGROUND_SERVICE, WAKE_LOCK and RECEIVE_BOOT_COMPLETED. Call-event features additionally require READ_CALL_LOG.

### `api/`
Go API/service.

Technology:
- Go 1.25.x
- Fiber v3
- GORM
- PostgreSQL-compatible storage (local Docker uses Postgres; production documentation references CockroachDB)
- Redis
- Firebase Admin SDK / FCM ecosystem
- Google Cloud Tasks
- OpenTelemetry
- Cloud Run deployment model
- Swagger/OpenAPI generated documentation
- SMTP/email
- Cloudflare Turnstile
- Pusher
- Lemon Squeezy billing
- Discord integration
- Excel/CSV processing

Entry point:
- `api/main.go`
- dependency composition is under `api/pkg/di`

The API is versioned under `/v1` and uses API-key authentication for programmable operations.

### `web/`
Nuxt 4 client-rendered SPA.

Technology:
- Nuxt 4
- Vue 3
- Vuetify 4
- Pinia
- TypeScript
- Firebase client SDK
- Pusher JS
- Chart.js
- libphonenumber-js
- Cloudflare Turnstile
- QR generation
- generated TypeScript API models from Swagger

Important configuration:
- `web/nuxt.config.ts`
- SSR is disabled (`ssr: false`)
- authenticated application routes are excluded from robots/sitemap
- dark Vuetify theme is the default
- runtime API base URL comes from `API_BASE_URL`
- Firebase, Pusher, Turnstile and checkout configuration are environment-driven
- `web/package.json` contains API model generation and lint/build commands

Do not assume a conventional Nuxt directory name for a feature. Inspect the current repository before editing pages/stores/composables because this repository may organize them differently than a stock Nuxt app.

## API surface

The current Swagger document exposes 37 path entries and 63 definitions. Main resource areas:

### Billing
- GET `/billing/usage`
- GET `/billing/usage-history`

### Bulk SMS
- GET/POST `/bulk-messages`
- POST `/messages/bulk-send`

Bulk uploads accept CSV/XLSX documents and are processed asynchronously.

### Discord
- GET/POST `/discord-integrations`
- PUT/DELETE `/discord-integrations/{discordID}`
- POST `/discord/event`

### Heartbeats
- GET/POST `/heartbeats`

Heartbeats represent Android gateway liveness/availability and are used to determine phone health.

### 3CX
- POST `/integration/3cx/messages`

### Message threads
- GET `/message-threads`
- PUT/DELETE `/message-threads/{messageThreadID}`

### Messages
- GET `/messages`
- POST `/messages/send`
- GET `/messages/outstanding`
- POST `/messages/receive`
- GET `/messages/search`
- POST `/messages/bulk-send`
- GET/DELETE `/messages/{messageID}`
- POST `/messages/{messageID}/events`
- POST `/messages/calls/missed`

### Phone API keys
- GET/POST `/phone-api-keys`
- DELETE `/phone-api-keys/{phoneAPIKeyID}`
- DELETE `/phone-api-keys/{phoneAPIKeyID}/phones/{phoneID}`

### Phones
- GET/PUT `/phones`
- PUT `/phones/fcm-token`
- DELETE `/phones/{phoneID}`

### Send schedules
- GET/POST `/send-schedules`
- PUT/DELETE `/send-schedules/{scheduleID}`

### Users
- GET/PUT/DELETE `/users/me`
- DELETE `/users/subscription`
- GET `/users/subscription-update-url`
- POST `/users/subscription/invoices/{subscriptionInvoiceID}`
- GET `/users/subscription/payments`
- DELETE `/users/{userID}/api-keys`
- PUT `/users/{userID}/notifications`

### Webhooks
- GET/POST `/webhooks`
- PUT/DELETE `/webhooks/{webhookID}`

### Attachments
- GET `/v1/attachments/{userID}/{messageID}/{attachmentIndex}/{filename}`

When implementing an API client change, update the source handler/service/model first and regenerate web API types where applicable rather than hand-editing generated API types.

## Core message lifecycle

### Outgoing SMS

Expected conceptual flow:

```
API client
  -> POST /v1/messages/send
  -> persist message
  -> enqueue async notification/event
  -> push/notification infrastructure
  -> Android FCM service
  -> Android fetches/handles outstanding message
  -> Android SmsManager/native SMS API
  -> Android posts message event/result
  -> API updates message/event state
```

The API intentionally returns acceptance asynchronously for send operations. Do not convert the flow into a synchronous 'wait until carrier delivery' request without understanding queue, expiration and retry semantics.

### Incoming SMS/MMS

```
Android Telephony broadcast
  -> ReceivedReceiver
  -> parse SMS or MMS
  -> identify SIM + owner
  -> check gateway active/incoming setting
  -> optionally encrypt content
  -> WorkManager network job
  -> POST /messages/receive
  -> persist/update thread
  -> webhook/integration/event consumers
  -> web UI/live updates
```

MMS attachments are temporarily stored on-device, encoded for forwarding, then local temporary files are cleaned up after forwarding.

## Encryption model

The repository supports optional end-to-end encryption for message content.

Important rule: encryption is performed on the phone, with the encryption key stored on the phone. The architectural goal is that the server cannot read encrypted SMS content.

Never:
- move the encryption key into web/frontend configuration
- log plaintext encrypted-message content
- persist the encryption key server-side
- silently change encryption defaults
- replace the existing cryptographic primitive without a migration/security review

When changing encryption code, inspect both outgoing and incoming paths and all event payloads.

## Back pressure and expiration

Back pressure exists to avoid abusing Android/carrier SMS sending capacity. A client may submit many messages while the gateway sends them at a controlled rate.

Expiration exists because a push/device may be offline. An outstanding message can become invalid after its timeout.

Changes affecting message queueing must preserve:
- ordering where required
- retries
- expiration semantics
- idempotency
- per-phone/per-user limits
- observable message states
- delivery reporting

Do not treat HTTP 202 as successful carrier delivery. It means the platform accepted the request for asynchronous processing.

## Authentication and credentials

There are several distinct credential concepts:
- web/Firebase authentication
- user API key for programmable HTTP API access
- phone API keys used to associate/control phones
- FCM registration token
- local Android login/session state

Never conflate these.

API keys must not be printed in logs, committed to source, or exposed in browser telemetry.

Environment/configuration includes sensitive Firebase service credentials, SMTP credentials, Turnstile secret, checkout/billing configuration and API/event queue credentials. Keep secrets out of Git.

## Android reliability rules

The Android app is the physical gateway, so reliability is a first-class requirement.

Before changing lifecycle/background behavior, consider:
- Android background execution limits
- foreground service requirements
- battery optimization
- boot restart
- WorkManager constraints/retries
- dual-SIM slot selection
- missing Google Play Services
- notification permissions
- SMS/MMS runtime permissions
- device offline state
- carrier/network failures
- duplicate broadcasts
- process death

Heartbeat scheduling currently uses WorkManager with a network-connected periodic job. FCM tokens are refreshed periodically rather than blindly on every UI lifecycle callback.

## Web UI rules

The web application is a client-rendered SPA. Keep authenticated screens out of search indexing.

When adding a route:
- determine whether it is public/marketing or authenticated
- update robots/sitemap exclusions for authenticated routes
- follow existing Vuetify theme/component conventions
- use Pinia/composables patterns already present in the repository
- use generated API types where available
- preserve Firebase auth behavior
- preserve Pusher/live-update behavior where relevant
- keep runtime configuration environment-driven

## Development

Root Docker composition provides:
- Postgres
- Redis
- API
- web

Typical documented local startup:

```bash
docker compose up --build
```

Documented local endpoints:
- web: `http://localhost:3000`
- API: `http://localhost:8000`

Web commands:
```bash
pnpm build
pnpm dev
pnpm generate
pnpm preview
pnpm api:models
pnpm lint
pnpm lintfix
```

API:
```bash
cd api
go test ./...
```

Use the repository's actual current CI/workflow files and Make/Task scripts if present before inventing commands.

## Change workflow for an AI agent

1. Identify the subsystem: Android, API, web, infrastructure, or cross-cutting.
2. Read the relevant source files and tests.
3. Trace the feature end-to-end across API ↔ Android ↔ web when applicable.
4. Check Swagger/API contracts before changing request/response shapes.
5. Preserve asynchronous semantics and idempotency.
6. Make the smallest coherent change.
7. Add/update tests.
8. Run targeted tests/lint first, then broader checks.
9. Regenerate derived API types/docs when their source contract changes.
10. Review secrets, logs, permissions and auth boundaries.
11. Summarize behavior changes and any deployment/configuration requirements.

## Cross-stack feature checklist

For a new SMS feature, inspect all of:
- API request/response model
- API handler/service/repository
- message persistence
- queue/event dispatch
- Android push handling
- Android SMS sending/receiving
- message event reporting
- message/thread state
- webhook/integration propagation
- web UI/API client
- billing/usage if billable
- encryption behavior
- expiration/retry behavior
- dual-SIM semantics
- tests and Swagger

For a new phone feature, inspect:
- phone registration/upsert
- FCM token lifecycle
- SIM identity
- heartbeat
- active/inactive state
- Android permissions/background service
- phone API keys
- web management UI

## Safety and correctness principles

- Prefer source-of-truth code over README claims when they differ.
- Never invent an endpoint, model field, queue state or Android service.
- Never expose secrets.
- Treat SMS sending as an external side effect; use idempotency and explicit state transitions.
- Treat webhook delivery as potentially unreliable; preserve retry/error semantics.
- Validate phone numbers using the repository's existing phone-number library.
- Avoid breaking older Android versions supported by minSdk.
- Preserve API backward compatibility unless a versioned migration is intentional.
- Do not assume PostgreSQL-only behavior when production storage may be CockroachDB-compatible.
- Keep observability useful without logging message bodies or credentials.

## Reference files

See:
- `skills/httpsms/references/architecture.md`
- `skills/httpsms/references/api.md`
- `skills/httpsms/references/android.md`
- `skills/httpsms/references/web.md`

These references are intentionally concise and should be expanded from repository source when a task touches a subsystem.
