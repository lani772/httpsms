# httpSMS Android Reference

## Lifecycle

`MainActivity`:
- initializes logging
- redirects unauthenticated users to LoginActivity
- initializes MainViewModel
- requests runtime permissions
- creates notification channel
- refreshes FCM token
- starts foreground listener when gateway is active
- schedules a periodic heartbeat WorkManager job

`LoginActivity`:
- requests SMS/phone/notification permissions
- initializes login ViewModel
- supports QR-code API-key scanning
- validates Google Play Services
- auto-detects phone numbers after permissions
- redirects authenticated users to MainActivity

`SettingsActivity`:
- Compose settings screen
- logout confirmation
- returns to login after logout

## Incoming transport

`ReceivedReceiver` handles:
- SMS_RECEIVED_ACTION
- WAP_PUSH_RECEIVED_ACTION

SMS:
- concatenates multipart SMS bodies
- identifies SIM/owner
- checks incoming-message activation
- forwards via WorkManager

MMS:
- parses PDU
- extracts text parts
- saves non-text parts temporarily
- forwards attachments as base64 with content type/name
- deletes temporary files after forwarding

Before forwarding, the receiver checks:
- logged-in state
- active gateway state
- incoming-message state
- optional message encryption

## Background reliability

Use WorkManager for network work that must survive transient connectivity failures.

Use the foreground service for the active gateway listener state.

Do not perform long network operations directly inside BroadcastReceiver callbacks.

## SMS side effects

Any code that invokes Android SMS APIs can cause real carrier messages. Test with controlled numbers and mocks/fakes where possible.

## Dual SIM

Always preserve:
- SIM identifier/slot
- owner phone number
- active state
- incoming/outgoing settings

Do not collapse SIM1 and SIM2 into a single global phone record.

## Permissions

Runtime permission behavior is part of product behavior. If a feature needs a new permission:
- declare it in AndroidManifest.xml
- request it at the correct lifecycle point
- handle denial
- provide a user-facing recovery path
- consider Android version-specific behavior
