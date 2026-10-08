# httpSMS API Reference

Base API: `/v1`
Authentication: API-key based for programmable operations; header documented as `x-api-Key`.

## Endpoint inventory

### Billing
- GET `/billing/usage`
- GET `/billing/usage-history`

### Bulk SMS
- GET/POST `/bulk-messages`
- POST `/messages/bulk-send`

### Discord
- GET/POST `/discord-integrations`
- PUT/DELETE `/discord-integrations/{discordID}`
- POST `/discord/event`

### Heartbeats
- GET/POST `/heartbeats`

### 3CX
- POST `/integration/3cx/messages`

### Threads
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

### Phones
- GET/PUT `/phones`
- PUT `/phones/fcm-token`
- DELETE `/phones/{phoneID}`

### Phone API keys
- GET/POST `/phone-api-keys`
- DELETE `/phone-api-keys/{phoneAPIKeyID}`
- DELETE `/phone-api-keys/{phoneAPIKeyID}/phones/{phoneID}`

### Scheduling
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

## Contract rules

- Read `api/docs/swagger.json` before modifying public API contracts.
- Preserve HTTP status semantics, especially asynchronous `202 Accepted` flows.
- Keep path/query/body names compatible with generated web API types.
- When request/response structures change, regenerate web API models using `web/package.json`'s `api:models` script.
- Preserve authentication and ownership checks.
- Treat message/event endpoints as asynchronous state transitions.
