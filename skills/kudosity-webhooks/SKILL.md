---
name: kudosity-webhooks
description: "Create and manage webhooks on the Kudosity platform. Use when receiving notifications for delivery status, inbound messages and replies, tapped RCS suggestions, MMS events, link hits, or opt-outs."
metadata:
  author: Kudosity
  version: 0.3.1
  category: Platform
  tags: webhooks, callbacks, delivery-receipts, inbound, replies, rcs-suggestions, link-hits, opt-out
  related:
    - kudosity-setup
    - kudosity-sms
    - kudosity-rcs
    - kudosity-whatsapp
---

# Create & Manage Webhooks

Webhooks allow users to receive HTTP POST notifications when events occur, such as SMS delivery status changes, inbound messages, link clicks, and opt-outs.

## Authentication

- Header: `x-api-key: {KUDOSITY_API_KEY}`
- Credentials come from the Kudosity dashboard under **Developers → API Settings**

## API Details

- **Base URL**: `https://api.transmitmessage.com`
- **Content-Type**: `application/json`
- All requests use `curl` commands

## Create a Webhook

**Endpoint**: `POST /v2/webhook`

Required parameters:
- `name` (string): Webhook name, 2-100 characters
- `url` (string): HTTPS URL that accepts JSON-encoded POST requests

Optional parameters:
- `filter` (object): Filter which events trigger the webhook
  - `event_type` (array): Types of events to subscribe to
  - `sender` (array): Filter by sender number
  - `status` (array): Filter by message status (for status events only)
  - `message_ref` (array): Filter by message reference
  - `campaign_id` (array): Filter by campaign ID
- `rate_limit` (integer): Max requests per second to your URL (max 10,000, 0 = system default)

### Available Event Types

Fourteen event types. Ten report on messages, covering every channel — not just SMS:

| Event Type | Description |
|-----------|-------------|
| `SMS_STATUS` | SMS delivery status changes |
| `SMS_INBOUND` | Inbound SMS received from a recipient |
| `MMS_STATUS` | MMS status changes (currently internal statuses only — SENT, FAILED) |
| `MMS_INBOUND` | Inbound MMS received from a recipient |
| `WHATSAPP_STATUS` | WhatsApp message status changes |
| `WHATSAPP_INBOUND` | Inbound WhatsApp message received from a recipient |
| `RCS_STATUS` | RCS status changes — including `READ` |
| `RCS_INBOUND` | Inbound RCS message received from a recipient — a text reply, a tapped suggestion, or rich content |
| `LINK_HIT` | Recipient clicked a tracked link |
| `OPT_OUT` | Recipient opted out via link or STOP message |

If you are sending WhatsApp or RCS, these are the event types you want — `SMS_STATUS` will not report on them.

Four more report on the account, not on messages:

| Event Type | Description |
|-----------|-------------|
| `PAYMENT_SUCCEEDED` | A payment charged to the account succeeded, such as a credit top-up |
| `PAYMENT_FAILED` | A payment charged to the account was declined or otherwise failed |
| `SENDER_REGISTRATION_STATUS` | A sender registration was created, or its `status` changed. Carries only `registration_id`, `type`, `sender` and the new `status` — call `GET /v2/senders/registrations` for the rest |
| `SENDER_COMPLIANCE_STATUS` | A sender's `compliance.status` changed, or the sender got or lost a compliance verdict. Carries only `sender_id`, `sender`, `registry` and the new `status` — call `GET /v2/senders/{id}` for the rest |

**These four are opt-in.** They are only delivered to a webhook whose `filter.event_type` names them. A webhook with no `event_type` filter receives all ten message event types and none of these.

For the two sender events, the `sender` filter applies to the event's `sender`. They have no message status, `message_ref` or `campaign_id`, so a webhook that sets any of those filters receives no sender events — give them a webhook of their own.

### Status Values

For any status event, the nested `status.status` field carries one of:

| Status | Meaning |
|---|---|
| `SENT` | Submitted to the carrier |
| `ACCEPTED` | Accepted by the carrier; delivery may have been attempted, but is not confirmed |
| `DELIVERED` | Delivered to the handset |
| `FAILED` | Error from the carrier or handset |
| `SOFT_BOUNCE` | Temporarily undeliverable — handset switched off, out of range |
| `HARD_BOUNCE` | Handset disconnected |
| `REJECTED` | Rejected before delivery — by Kudosity (for example, compliance validation) or, for RCS, by the carrier or gateway (for example, a handset that can't receive RCS) |
| `UNDELIVERABLE` | **RCS** — reached the carrier but couldn't be delivered: handset unreachable, or the message expired |
| `READ` | Read by the recipient — **RCS and WhatsApp**; SMS and MMS never report it |
| `OTHER` | Any other status from the carrier |

`ACCEPTED` is not `DELIVERED`. Treating it as a successful delivery is a common source of over-reported success rates.

For RCS, the failed statuses are `REJECTED`, `UNDELIVERABLE` and `FAILED`. The webhook is the only place you see the difference: `GET /v2/rcs/messages` collapses all three into `FAILED`.

### Filter Logic

- **Within a filter array** (e.g., multiple senders): conditions are combined with **OR**
- **Between different filters** (e.g., sender AND status): conditions are combined with **AND**

The same filter name reads a different part of the payload depending on the event type:

| Event type | `sender` / `message_ref` are matched against |
|---|---|
| Status events (`SMS_STATUS`, `RCS_STATUS`, …) | the `status` payload |
| `LINK_HIT` | `link_hit.source_message` |
| `OPT_OUT` | `opt_out.source_message` |
| Inbound events (`SMS_INBOUND`, …) | `sender` matches the **`recipient`** field — the address that received the inbound. `message_ref` and `campaign_id` match `last_message`, when present. |

The inbound row is the one that surprises people: filtering inbound events by `sender` filters by *your* number, not the customer's.

### Example: Create a webhook for SMS delivery status and inbound messages

```bash
curl -s -X POST "https://api.transmitmessage.com/v2/webhook" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.3.1" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "SMS Events",
    "url": "https://myapp.com/webhooks/sms",
    "filter": {"event_type": ["SMS_STATUS", "SMS_INBOUND"]}
  }'
```

### Example: Create a webhook filtered to specific statuses

```bash
curl -s -X POST "https://api.transmitmessage.com/v2/webhook" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.3.1" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Failed SMS Alerts",
    "url": "https://myapp.com/webhooks/failures",
    "filter": {"event_type": ["SMS_STATUS"], "status": ["FAILED", "HARD_BOUNCE"]}
  }'
```

## List All Webhooks

**Endpoint**: `GET /v2/webhook`

```bash
curl -s "https://api.transmitmessage.com/v2/webhook" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.3.1"
```

## Webhook Payload Examples

### SMS Status Event
```json
{
  "event_type": "SMS_STATUS",
  "timestamp": "2021-05-06T05:16:07Z",
  "status": {
    "type": "SMS",
    "id": "a51ebe4e-a412-440e-a8d9-464e68a521cc",
    "message_ref": "ncc5009d",
    "recipient": "447507222200",
    "routed_via": "447507333300",
    "sender": "61481074190",
    "status": "SENT"
  }
}
```

### SMS Inbound Event

```json
{
  "event_type": "SMS_INBOUND",
  "timestamp": "2021-05-06T05:16:33Z",
  "mo": {
    "type": "SMS",
    "id": "alss-2way-605b31c7-d2c49104",
    "message": "Yes, I'm interested",
    "recipient": "61481074190",
    "routed_via": "447507333300",
    "sender": "447507222200",
    "last_message": {
      "type": "SMS",
      "id": "a51ebe4e-a412-440e-a8d9-464e68a521cc",
      "message": "Hey, check this out!",
      "message_ref": "ncc5009d",
      "recipient": "447507222200",
      "routed_via": "447507333300",
      "sender": "61481074190"
    }
  }
}
```

**`last_message` is the field that makes two-way messaging work.** On receipt of an inbound message, Kudosity looks for a message you sent to that recipient from that sender and attaches it here. Its `message_ref` is your join key back to whatever the original send belonged to — an order, a booking, a conversation thread.

Route replies on `mo.last_message.message_ref`, not on the phone number. Number matching breaks the first time one contact is in two flows at once, and it breaks again when a shared number is involved.

It is best-effort — if no recent outbound matches, `last_message` is absent. Handle that case rather than assuming it's always there.

### Shared numbers and `routed_via`

`routed_via` appears when a **shared local number** was used to deliver your message. When it's present, the number the recipient actually replied to is not your sender.

Any logic that pairs an inbound message to an outbound one by comparing `sender`/`recipient` needs to account for this — which is the other reason to prefer `last_message.message_ref`.

### RCS Status Event

```json
{
  "event_type": "RCS_STATUS",
  "timestamp": "2026-11-17T01:05:04Z",
  "webhook_id": "67e3e4af-62e8-4515-8b67-6f20d0c51ffb",
  "webhook_name": "RCS and SMS status",
  "status": {
    "type": "RCS",
    "id": "0b8f3c2e-6d4a-4f1e-9a7b-2c5d8e1f4a63",
    "message_ref": "order-12345",
    "sender": "DemoSender",
    "recipient": "61438333061",
    "status": "DELIVERED",
    "reason_code": "DELIVERED_TO_HANDSET"
  }
}
```

`status.sender` is the RCS Agent ID, not a phone number. A failed event can carry a `reason_code` such as `NOT_RCS_CAPABLE` or `EXPIRED`, or a `description`. The `description` is text for people to read — match on `status` and `reason_code`, never on `description`.

### RCS with an SMS fallback: one message, two event types

An RCS message sent with `sms_fallback` reports on two channels. When the RCS leg fails, the fallback SMS reports through `SMS_STATUS` — with the **same `id` and `message_ref`** as the RCS message, and the SMS sender in `status.sender`.

Subscribe one webhook to both, or you will see the RCS failure and never learn that the SMS arrived:

```bash
curl -s -X POST "https://api.transmitmessage.com/v2/webhook" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.3.1" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "RCS and SMS status",
    "url": "https://myapp.com/webhooks/kudosity",
    "filter": {"event_type": ["RCS_STATUS", "SMS_STATUS"]}
  }'
```

Match on `status.id`, then read `status.type` and `status.status` together: `RCS` + `DELIVERED`/`READ` means it arrived as RCS; `SMS` + `DELIVERED` means it arrived as the fallback. A failed `RCS_STATUS` with no `SMS_STATUS` to follow means the fallback wasn't sent — most often because the send had no `sms_fallback.sender`. See `kudosity-rcs` for the full table.

### RCS Inbound Event

`RCS_INBOUND` fires when a recipient replies in an RCS conversation: free text, a tap on a suggested reply or action, or rich content. `mo.content_type` says which — `TEXT`, `SUGGESTION`, `IMAGE`, `VIDEO`, `AUDIO`, `FILE`, `LOCATION` or `CONTACT` — and the matching content is nested under `mo.content`. Unlike `SMS_INBOUND`, there is no flat `mo.message` field.

A free-text reply:

```json
{
  "event_type": "RCS_INBOUND",
  "timestamp": "2026-11-12T01:34:51.246802Z",
  "webhook_id": "fd0e6485-b905-44c1-bd55-fee1d0d6d864",
  "webhook_name": "RCS Inbound Webhook",
  "mo": {
    "type": "RCS",
    "id": "bc2e0a4a-7e21-49de-9912-20cb4148c21f",
    "sender": "61438333061",
    "recipient": "DemoSender",
    "content_type": "TEXT",
    "content": { "text": { "message": "Yes, that time works for me." } }
  }
}
```

A tapped suggestion:

```json
{
  "event_type": "RCS_INBOUND",
  "timestamp": "2026-11-12T01:35:20.246802Z",
  "webhook_id": "fd0e6485-b905-44c1-bd55-fee1d0d6d864",
  "webhook_name": "RCS Inbound Webhook",
  "mo": {
    "type": "RCS",
    "id": "5d1c7a90-3f6e-4b0a-8f27-91c4e2a6b7d3",
    "sender": "61438333061",
    "recipient": "DemoSender",
    "content_type": "SUGGESTION",
    "content": { "suggestion": { "text": "Yes", "postback_data": "appt-789-yes" } }
  }
}
```

`mo.sender` is the person who replied; `mo.recipient` is your RCS agent. `suggestion.text` is the label they tapped, and `suggestion.postback_data` is the value you set on that suggestion when you sent the message.

**Route taps on `postback_data`, not on the label.** Labels get reworded and repeat across messages ("Yes" on every confirmation you send); `postback_data` is yours, so make it carry the record and the choice — `appt-789-yes`. If the send left it out, Kudosity fills in a value that means nothing to your application. Branch on `mo.content_type` first: a handler that reads `content.text.message` on every `RCS_INBOUND` breaks on the first tap.

### Link Hit Event

The payload nests under `link_hit`, with a running total of `hits` and the message the link was sent in:

```json
{
  "event_type": "LINK_HIT",
  "timestamp": "2021-07-20T23:14:04Z",
  "link_hit": {
    "hits": 1,
    "url": "https://www.example.com/abc",
    "source_message": {
      "type": "SMS",
      "id": "faf68308-16cd-4cf9-aef7-47342bd405be",
      "message": "Hey, Check this out! http://clckme.info/KYhSsuIH",
      "message_ref": "D301",
      "recipient": "61435795809",
      "sender": "61481074185"
    }
  }
}
```

For an MMS the `source_message` additionally carries `subject` and `content_urls`.

`url` is the original destination, not the shortened link. `hits` is cumulative for that tracked link — so it counts repeat clicks, and is not a unique-recipient count.

### Opt-Out Event
```json
{
  "event_type": "OPT_OUT",
  "timestamp": "2021-05-06T05:16:20Z",
  "opt_out": {
    "source": "SMS_INBOUND",
    "source_message": {
      "type": "SMS",
      "id": "a51ebe4e-a412-440e-a8d9-464e68a521cc",
      "message": "Your promo message with opt-out link",
      "message_ref": "ncc5009d",
      "recipient": "61435790000",
      "sender": "61481074190"
    }
  }
}
```

`opt_out.source` tells you how they opted out — `SMS_INBOUND` for a STOP reply, `LINK_HIT` for the opt-out link. Both are binding; don't treat the link as weaker consent withdrawal than the reply.

## Important Notes

- Webhook URLs **must use HTTPS**
- A successful create returns **`201`**, not `200`
- The `event_type` field at the top level of the request body is **deprecated** — always use `filter.event_type` instead
- Rate limit defaults to the system limit if set to 0 or not specified; max is 10,000/sec
- **Multiple status events can be triggered for a single message**, and they are not guaranteed to arrive in order. Key on `status.id` and treat handling as idempotent — a message can go `SENT` → `DELIVERED`, and a late `SENT` must not overwrite a recorded `DELIVERED`.
- Validation errors on this endpoint return a plain `{"error": "..."}` string, not the RFC 9457 Problem Details object the messaging endpoints use
- Always confirm with the user which event types they need before creating the webhook