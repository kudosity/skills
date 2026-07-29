---
name: kudosity-rcs
description: "Send RCS (Rich Communication Services) messages via the Kudosity platform, with automatic SMS fallback for non-RCS devices. Use when sending an RCS message, checking whether a number can receive RCS, or configuring RCS-to-SMS fallback. RCS sends through a registered agent ID, not a phone number."
metadata:
  author: Kudosity
  version: 0.1.0
  category: Messaging
  tags: rcs, rich-messaging, sms-fallback, capability-check, agent-id, rbm
  related:
    - kudosity-setup
    - kudosity-webhooks
---

# Send RCS Messages

RCS is rich, branded messaging delivered in the recipient's native messaging app. Kudosity sends RCS over the V2 API.

> **Beta.** The RCS endpoints are marked beta in the Kudosity API and may change. Check `https://developers.kudosity.com` before relying on a field not listed here.

## The one thing that trips people up

**RCS does not send from a phone number.** Unlike SMS, MMS and WhatsApp, the `sender` field is a **registered RCS agent ID** — alphanumeric or numeric, e.g. `DemoSender`. If you pass a phone number, the send fails validation.

An agent must be registered and launched on the destination carrier before it will deliver. Registration is not part of this API — talk to your Kudosity account contact to get an agent provisioned, then use its ID here.

## Authentication

- Header: `x-api-key: {KUDOSITY_API_KEY}`
- Base URL: `https://api.transmitmessage.com`

## Send an RCS message

**Endpoint:** `POST /v2/rcs/messages`

Required:
- `sender` (string) — your registered RCS agent ID
- `recipient` (string) — destination number, local or [E.164](https://en.wikipedia.org/wiki/E.164) format (`0438 333 061` → `61438333061`)
- `content_type` (string) — currently only `text`
- `content` (object) — `{ "text": { "message": "..." } }`

Optional:
- `sms_fallback` (object) — see below. **Use this on almost every send.**
- `message_ref` (string, max 500 chars) — your reference, echoed back in webhook events

### Message length

- **Basic RCS** — up to 160 characters (SMS-shaped)
- **Simple RCS** — up to 3072 characters, full UTF-8 and emoji

```bash
curl -s -X POST "https://api.transmitmessage.com/v2/rcs/messages" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.1.0" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": "DemoSender",
    "recipient": "61438333061",
    "content_type": "text",
    "content": { "text": { "message": "Your order has shipped. Track it: https://example.com/t/7782" } },
    "sms_fallback": {
      "sender": "61481074185",
      "message": "Your order has shipped. Track it: https://example.com/t/7782"
    },
    "message_ref": "order-7782"
  }'
```

Response is wrapped in a `data` envelope:

```json
{ "data": { "id": "6fdae71c-dad7-4c36-9734-a69693ecf3b4", "sender": "DemoSender",
            "recipient": "61438333061", "created_at": "2026-07-29T00:00:00Z" } }
```

Keep `data.id` — it's what you match webhook events against.

## SMS fallback — do this by default

Not every handset supports RCS, and carrier delivery can fail. `sms_fallback` sends an SMS instead when the RCS leg doesn't land.

```json
"sms_fallback": {
  "sender": "61481074185",
  "message": "Shorter plain-text version."
}
```

- `message` is **required** when `sms_fallback` is present
- `sender` is optional but should be a number or alphanumeric sender ID registered to your account
- The fallback is a real SMS — it is billed as one and is subject to SMS character limits, so write a separate, shorter body rather than reusing a 3072-character RCS message

## Check RCS capability before sending

**Endpoint:** `POST /v2/rcs/capabilities`

Use this to segment an audience — send RCS to capable devices and plain SMS to the rest — rather than paying for a fallback on every send.

```bash
curl -s -X POST "https://api.transmitmessage.com/v2/rcs/capabilities" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.1.0" \
  -H "Content-Type: application/json" \
  -d '{"recipients": ["61438333061", "61491570156"]}'
```

Capability is per agent *and* per device — a number capable for one agent is not guaranteed capable for another, and results go stale. Re-check rather than caching indefinitely.

## Read messages back

- `GET /v2/rcs/messages` — list sent RCS messages
- `GET /v2/rcs/messages/{id}` — get one by ID

## Delivery events

Register a webhook (see `kudosity-webhooks`) to receive delivery status, link hits and replies. `message_ref` is echoed in every event, so set it to something you can join on — an order ID, an invoice number.

## Errors

Errors follow RFC 9457 Problem Details under an `error` key:

```json
{ "error": { "type": "https://developers.kudosity.com/reference/errors#input-validation",
             "title": "Invalid Request", "detail": "Request validation failed", "status": 400,
             "issues": [{ "name": "sender", "message": "sender is required" }] } }
```

Read `error.issues[]` — it lists every failed field at once rather than one per attempt.

| Status | Meaning | Usual cause |
|---|---|---|
| 400 | Input validation | `sender` is a phone number instead of an agent ID; missing `content_type` |
| 401 | Auth failed | Missing or wrong `x-api-key` |
| 500 | Server error | Retry with backoff |

## Related

- `kudosity-sms` — the fallback channel, and the right choice when you don't need rich content
- `kudosity-webhooks` — delivery status, replies, link hits
- Working example: [`ai-payment-reminder-rcs`](https://github.com/kudosity/ai-agent-examples/tree/main/ai-payment-reminder-rcs) and [`ai-delivery-update-rcs`](https://github.com/kudosity/ai-agent-examples/tree/main/ai-delivery-update-rcs)
