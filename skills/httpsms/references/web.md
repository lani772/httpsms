# httpSMS Web Reference

## Stack

- Nuxt 4
- Vue 3
- Vuetify 4
- Pinia
- TypeScript
- Firebase
- Pusher
- Chart.js
- libphonenumber-js
- Cloudflare Turnstile

The SPA uses `ssr: false`.

## Runtime configuration

Important public configuration includes:
- API base URL
- application URL/name
- documentation/download/GitHub URLs
- checkout URLs
- Cloudflare Turnstile site key
- Pusher key/cluster
- Firebase web configuration
- app environment/version

Do not put private credentials into `runtimeConfig.public`.

## SEO/security

Authenticated routes are explicitly excluded from robots/sitemap, including:
- /messages
- /threads
- /settings
- /billing
- /bulk-messages
- /heartbeats
- /phone-api-keys
- /search-messages

Maintain these exclusions when adding authenticated pages.

## API integration

The web package can generate TypeScript API models from the backend Swagger document:

```bash
pnpm api:models
```

Prefer generated types and existing API abstractions over handwritten duplicate models.

## UI behavior

Follow existing Vuetify theme/component patterns. The default theme is dark.

When changing message UI, account for:
- asynchronous message status
- incoming/outgoing direction
- threads
- encrypted content
- MMS attachments
- phone/SIM availability
- live updates
- pagination/search
