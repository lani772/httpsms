# httpSMS Architecture Reference

## System components

| Component | Technology | Responsibility |
|---|---|---|
| Android gateway | Kotlin, Compose | Physical SMS/MMS transport, FCM command receiver, phone/SIM state |
| API | Go, Fiber, GORM | Authenticated API, persistence, orchestration, queues/events, integrations |
| Database | Postgres-compatible / CockroachDB production | Users, phones, messages, threads, events, billing/integrations |
| Redis | Redis | Caching/coordination/queue-adjacent infrastructure |
| Push | Firebase Cloud Messaging | Notify Android gateway of work |
| Async | Google Cloud Tasks / event mechanisms | Deferred processing and push/event orchestration |
| Web | Nuxt 4, Vue 3, Vuetify 4, Pinia | Dashboard, configuration, messaging and billing UI |
| Live updates | Pusher | Browser-side real-time updates where used |
| Billing | Lemon Squeezy | Subscription and payment operations |
| Abuse protection | Cloudflare Turnstile | Message search protection |
| Observability | OpenTelemetry, Sentry, logging | Traces, metrics, errors |

## Trust boundaries

1. Browser -> API: authenticated user/API-key boundary.
2. API -> Android: push + authenticated phone/message flow.
3. Android -> carrier: physical SMS/MMS side effect.
4. API -> webhook target: outbound third-party network boundary.
5. API -> billing/integration providers: third-party service boundary.
6. Android local storage -> device owner: encryption key and gateway credentials are device-local concerns.

## Data-flow invariants

- A submitted send request is not equivalent to carrier delivery.
- Message events can arrive asynchronously.
- Android can be offline after the API accepts a request.
- Incoming messages originate outside the API and must be authenticated/associated with a registered phone.
- Webhooks are integrations, not the primary source of message state.
- Encryption changes what the server can inspect.
- Dual-SIM means owner phone number and SIM identity are separate dimensions.

## Deployment

The README describes:
- web hosted as Firebase SPA
- API deployed serverlessly on Google Cloud Run
- FCM for Android notifications
- production database documented as CockroachDB
- Docker Compose for local development

When deployment code and README disagree, inspect the deployment files and CI workflows and prefer executable configuration.
