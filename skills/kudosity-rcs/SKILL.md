---
name: kudosity-rcs
description: "Send RCS (Rich Communication Services) messages via the Kudosity platform: text, images and video, rich cards, carousels, and suggested reply, link and call buttons, with SMS fallback for non-RCS devices. Use when sending an RCS message, building a rich card or carousel, adding buttons or quick replies, handling a tapped suggestion, checking whether a number can receive RCS, or setting up and tracking RCS-to-SMS fallback. RCS sends through an RCS agent ID, not a phone number."
metadata:
  author: Kudosity
  version: 0.3.0
  category: Messaging
  tags: rcs, rich-messaging, rich-card, carousel, suggestions, sms-fallback, capability-check, agent-id, rbm
  related:
    - kudosity-setup
    - kudosity-webhooks
---

# Send RCS Messages

RCS is rich, branded messaging delivered in the recipient's native messaging app. Kudosity sends RCS over the V2 API: plain text, a single image or video, a rich card, or a carousel of cards, each with optional tappable suggestions.

## Three things that trip people up

**1. RCS does not send from a phone number.** Unlike SMS, MMS and WhatsApp, the `sender` field is the **Agent ID of an RCS agent on your account**, e.g. `DemoSender`. Pass a phone number and the send fails validation.

- Find the Agent ID in the Kudosity app: **RCS → RCS Agents**, open the agent, **Agent details** on the **Details** tab.
- You create, test and launch the agent yourself in the app (**RCS → RCS Agents → Create agent**). See the [RCS Agent Setup Guide](https://developers.kudosity.com/docs/rcs-onboarding). If **RCS** isn't in the menu, it isn't enabled on the account yet — the account manager turns it on.
- **Until the agent is live, it can message its accepted test devices only.** A send to any other number from an unlaunched agent will not deliver.

**2. The recipient must be in E.164 international format.** `61438333061` or `+61438333061`. Local format such as `0438 333 061` is rejected — convert it first (`0438 333 061` → `61438333061`, UK `07123 456 789` → `447123456789`).

**3. An SMS fallback needs a sender.** A request with `sms_fallback.message` and no `sms_fallback.sender` is accepted, but when the RCS leg fails the SMS is never sent. Always set both. See [SMS fallback](#sms-fallback--do-this-by-default).

## Authentication

- Header: `x-api-key: {KUDOSITY_API_KEY}`
- Base URL: `https://api.transmitmessage.com`

## Send an RCS message

**Endpoint:** `POST /v2/rcs/messages`

Required:
- `sender` (string) — your RCS Agent ID
- `recipient` (string) — destination number in E.164 format
- `content_type` (string) — `text`, `media` or `rich_card`
- `content` (object) — must contain the object that matches `content_type`

Optional:
- `content.suggestions` (array) — up to 11 chips shown under the message, on any content type
- `sms_fallback` (object) — see below. **Use this on almost every send.**
- `message_ref` (string, max 500 chars) — your reference, echoed back in `RCS_STATUS` webhook events and when you get the message

### Choose a format

| Format | `content_type` | Object inside `content` | Use it for |
|---|---|---|---|
| Text | `text` | `text` | Reminders, alerts, conversations |
| Image or video | `media` | `media` | One media file, **no caption** |
| Standalone rich card | `rich_card` | `rich_card.standalone` | One item with text, optional media and buttons |
| Carousel | `rich_card` | `rich_card.carousel` | 2 to 10 cards the recipient swipes through |

`content_type` must match the key you send inside `content`, or the request fails with `400`. Standalone cards and carousels both use `rich_card`.

### Text, with suggested replies

Text carries up to 3072 characters, full UTF-8 and emoji. A message of up to 160 characters is a **Basic** RCS message; a longer one is a **Single** RCS message.

```bash
curl -s -X POST "https://api.transmitmessage.com/v2/rcs/messages" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.3.0" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": "DemoSender",
    "recipient": "61438333061",
    "content_type": "text",
    "content": {
      "text": { "message": "Can you confirm your appointment on Thursday at 10am?" },
      "suggestions": [
        { "reply": { "text": "Yes", "postback_data": "appt-789-yes" } },
        { "reply": { "text": "No",  "postback_data": "appt-789-no" } }
      ]
    },
    "sms_fallback": {
      "sender": "61481074185",
      "message": "Can you confirm your appointment on Thursday at 10am? Reply YES or NO."
    },
    "message_ref": "appt-789"
  }'
```

For plain text, leave out `content.suggestions`.

A successful send returns `200` with the message as accepted, wrapped in a `data` envelope:

```json
{ "data": { "id": "0b8f3c2e-6d4a-4f1e-9a7b-2c5d8e1f4a63", "message_ref": "appt-789",
            "sender": "DemoSender", "recipient": "61438333061", "recipient_country": "AU",
            "content_type": "text", "content": { "...": "as sent" },
            "sms_fallback": { "...": "as sent" }, "created_at": "2026-11-12T01:30:00Z" } }
```

Keep `data.id` — it's what you match webhook events against.

- **`200` means accepted, not delivered.** Track delivery with the `RCS_STATUS` webhook event or by getting the message.
- **There is no idempotency.** Sending the same request again sends a second message.

### Image or video

One file by public URL. There is no separate content type for video — point `file_url` at a video file. Media messages don't support a caption; when the image needs words, use a standalone rich card.

```json
{
  "sender": "DemoSender",
  "recipient": "61438333061",
  "content_type": "media",
  "content": {
    "media": { "file_url": "https://cdn.example.com/promo/summer-sale.jpg" },
    "suggestions": [
      { "action": { "text": "Shop now", "open_url_action": { "url": "https://example.com/sale" } } }
    ]
  },
  "message_ref": "sale-2027"
}
```

### Standalone rich card

A card combines an image or video, a title, a description and up to 4 buttons. It needs **at least one** of `title`, `description` or `media` — not necessarily all three.

```json
{
  "sender": "DemoSender",
  "recipient": "61438333061",
  "content_type": "rich_card",
  "content": {
    "rich_card": {
      "standalone": {
        "card_content": {
          "title": "Your order has shipped",
          "description": "Order #12345 is on its way and should arrive Thursday.",
          "media": { "file_url": "https://cdn.example.com/orders/parcel.jpg" },
          "suggestions": [
            { "action": { "text": "Track parcel", "open_url_action": { "url": "https://example.com/track/12345" } } }
          ]
        }
      }
    }
  },
  "sms_fallback": {
    "sender": "61481074185",
    "message": "Your order #12345 has shipped. Track it: https://example.com/track/12345"
  },
  "message_ref": "order-12345"
}
```

### Carousel

Each entry in `card_contents` is one card, following the same rules as a standalone card. A carousel needs **2 to 10 cards**. Set either `standalone` or `carousel` inside `rich_card`, never both.

```json
{
  "sender": "DemoSender",
  "recipient": "61438333061",
  "content_type": "rich_card",
  "content": {
    "rich_card": {
      "carousel": {
        "card_contents": [
          {
            "title": "Classic tee",
            "description": "$29",
            "media": { "file_url": "https://cdn.example.com/shop/tee.jpg" },
            "suggestions": [
              { "action": { "text": "Buy tee", "open_url_action": { "url": "https://example.com/shop/tee" } } },
              { "reply": { "text": "More colours", "postback_data": "tee-colours" } }
            ]
          },
          {
            "title": "Hoodie",
            "description": "$59",
            "media": { "file_url": "https://cdn.example.com/shop/hoodie.jpg" },
            "suggestions": [
              { "action": { "text": "Buy hoodie", "open_url_action": { "url": "https://example.com/shop/hoodie" } } },
              { "reply": { "text": "Size guide", "postback_data": "hoodie-sizes" } }
            ]
          }
        ]
      }
    },
    "suggestions": [
      { "action": { "text": "Call us", "dial_action": { "phone_number": "+61290000000" } } }
    ]
  },
  "sms_fallback": {
    "sender": "61481074185",
    "message": "New arrivals: the classic tee ($29) and hoodie ($59). Shop at example.com"
  }
}
```

The `cdn.example.com` URLs are placeholders and don't exist — a request that uses them fails with `400`, because every media file is checked at send time.

## Suggested replies and actions

A suggestion is either a `reply` or an `action`. An action is either an `open_url_action` or a `dial_action`. Set exactly one at each level.

| Type | What a tap does | Shape |
|---|---|---|
| Reply | Sends the label back to your agent as a reply | `{"reply": {"text": "Yes", "postback_data": "appt-789-yes"}}` |
| Open URL | Opens a web page | `{"action": {"text": "Shop now", "open_url_action": {"url": "https://example.com/sale"}}}` |
| Dial | Opens the dialler with the number filled in | `{"action": {"text": "Call us", "dial_action": {"phone_number": "+61290000000"}}}` |

Rules:
- `text` is the label the recipient sees — **1 to 25 characters**
- `open_url_action.url` — an absolute `https` URL, up to 2048 characters
- `dial_action.phone_number` — E.164, leading `+` optional; local formats such as `02 9000 0000` are rejected
- `postback_data` — optional on replies and actions, up to 2048 characters, never shown to the recipient

### Where to put them

| Placement | JSON field | Limit | Behaviour |
|---|---|---|---|
| Under the message | `content.suggestions` | 11 chips | Disappear once the recipient replies or another message arrives |
| On a card | `card_content.suggestions` (standalone) or `card_contents[n].suggestions` (carousel) | 4 per card | Stay on the card after the recipient replies |

Use card buttons for actions that should stay available, such as tracking an order. Use chips under the message for a quick answer to the latest message.

### Handle a tap

**Always set your own `postback_data`.** Leave it out and Kudosity sets a value for you, which maps to nothing in your application. Make it something you can route on — an order, a booking, a step in a conversation.

A tap arrives on your `RCS_INBOUND` webhook with `mo.content_type` set to `SUGGESTION`. The label is in `mo.content.suggestion.text` and your value in `mo.content.suggestion.postback_data`:

```json
{
  "event_type": "RCS_INBOUND",
  "timestamp": "2026-11-12T01:31:20.246802Z",
  "mo": {
    "type": "RCS",
    "id": "bc2e0a4a-7e21-49de-9912-20cb4148c21f",
    "sender": "61438333061",
    "recipient": "DemoSender",
    "content_type": "SUGGESTION",
    "content": { "suggestion": { "text": "Yes", "postback_data": "appt-789-yes" } }
  }
}
```

`message_ref` tells you which message a *status* event belongs to. `postback_data` tells you which *suggestion* was tapped. They are different jobs — don't try to use `message_ref` for taps.

## SMS fallback — do this by default

Not every handset supports RCS, and delivery can fail. With `sms_fallback`, Kudosity sends an SMS when the RCS message is rejected, can't be delivered, or expires. Without it, a failed RCS message is not sent on any other channel.

```json
"sms_fallback": {
  "sender": "61481074185",
  "message": "Your order #12345 has shipped. Track it: https://example.com/track/12345"
}
```

- `message` is **required** when `sms_fallback` is present.
- **Set `sender` every time.** The API accepts a fallback without one, but then the SMS is not sent: you get the failed `RCS_STATUS` event and no `SMS_STATUS` events.
- `sender` must be a number or alphanumeric sender ID your account can send SMS from. Otherwise the request fails with **`404`**.
- **Write it as a standalone SMS.** Images, cards and buttons are not carried over, so put any link or reply instruction in the text ("Reply YES or NO"). A message too long for an SMS fails with `400`.
- It is a real SMS: it is billed as one, it isn't sent to a recipient who has opted out of your SMS or when the account is out of credit, and links in it are not shortened or tracked.

### Which channel delivered it

The fallback SMS carries the **same `id` and `message_ref`** as the RCS message it replaces. Subscribe one webhook to both `RCS_STATUS` and `SMS_STATUS`, match events on `status.id`, then read `status.type` and `status.status` together:

| You receive | What it means |
|---|---|
| `RCS_STATUS` with `DELIVERED` or `READ` | Delivered as RCS |
| `SMS_STATUS` with `DELIVERED` | Delivered as SMS |
| A failed `RCS_STATUS` and a failed `SMS_STATUS` | Not delivered on either channel |
| A failed `RCS_STATUS`, fallback set, no final `SMS_STATUS` yet | The SMS is still on its way, or couldn't be sent (no sender, opt-out, no credit) |
| A failed `RCS_STATUS` and no `sms_fallback` | Not delivered |

A failed RCS status is `REJECTED`, `UNDELIVERABLE` or `FAILED`. A failed SMS status is `HARD_BOUNCE`, `SOFT_BOUNCE`, `FAILED` or `REJECTED`. `SENT` and `ACCEPTED` are not final. Events can arrive in any order.

## Media file requirements

These apply to `content.media.file_url` and to the `media.file_url` of every card. Each file is checked when you send, and the request fails with `400` if one doesn't pass.

- An absolute `https` URL, up to 2048 characters, on the default port, without credentials. Percent-encode spaces and non-ASCII characters.
- Publicly reachable, returning the file directly — **no redirects** — with the server responding within 10 seconds.
- The server must return a `Content-Type` for a supported type, and the file content must match it:
  - Images: `image/jpeg`, `image/png`, `image/gif`
  - Videos: `video/mp4`, `video/mpeg`, `video/mpeg4`, `video/m4v`, `video/webm`, `video/h263`
- Maximum 100 MB. Return `Content-Length` or `Content-Range` so the size can be checked.
- Files are cached by URL for up to 60 days. To change a file, host it at a new URL.

A signed URL that redirects, a file behind a login, or a link to a page that *shows* an image rather than the image itself are the usual causes of a media `400`.

## Limits at a glance

| Item | Limit |
|---|---|
| Text message | 3072 characters |
| Cards in a carousel | 2 to 10 |
| Card title | 200 characters |
| Card description | 2000 characters |
| Buttons on a card | 4 |
| Suggestions under a message | 11 |
| Suggestion label | 1 to 25 characters |
| `postback_data` | 2048 characters |
| Link and media URLs | 2048 characters |
| Media file size | 100 MB |
| `message_ref` | 500 characters |

How a message looks varies between messaging apps and devices. Test button placement, media cropping and the SMS fallback on the agent's test devices before sending to customers.

## Check RCS capability before sending

**Endpoint:** `POST /v2/rcs/capabilities`

Two required fields: `sender` — the Agent ID you intend to send from — and `phone_numbers`. Capability is **per agent**, so a check that omits the sender is meaningless; a number reachable for one agent is not guaranteed reachable for another.

```bash
curl -s -X POST "https://api.transmitmessage.com/v2/rcs/capabilities" \
  -H "x-api-key: ${KUDOSITY_API_KEY}" \
  -H "User-Agent: kudosity-skills/0.3.0" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": "DemoSender",
    "phone_numbers": ["61438333061", "61491570156"]
  }'
```

Numbers are E.164 without the leading `+`. Up to 100 per request; batches of **1–10** keep latency low enough for routing-time lookups.

Results come back one per number, in request order:

```json
{ "data": { "results": [
  { "phone_number": "61438333061", "code": "ENABLED" },
  { "phone_number": "61491570156", "code": "UNREACHABLE" }
] } }
```

`code` is not a boolean:

| Code | Meaning |
|---|---|
| `ENABLED` | Can receive RCS from this sender |
| `UNREACHABLE` | Cannot receive RCS from this sender |
| `REJECTED_NETWORK` | Not sent — network not allowed |
| `REJECTED_ROUTE_NOT_AVAILABLE` | No route available for that sender |
| `REQUEST_FAILED` | External problem during the check |
| `PROCESSING_ERROR` | Couldn't be processed — retry |
| `INVALID_DESTINATION_ADDRESS` | Number in the wrong format |
| `UNKNOWN` | Unknown upstream error, or a code this list doesn't cover yet |

**Don't use this as a hard gate on sending.** Results are best-effort and reflect capability at the moment of the lookup. Treat `UNKNOWN` as reachable, send anyway, and let `sms_fallback` carry the ones that don't land — the fallback is what gives you a hard delivery guarantee, not this check. Capability also goes stale, so re-check rather than caching indefinitely.

## Read messages back

- `GET /v2/rcs/messages` — list sent RCS messages, newest first
- `GET /v2/rcs/messages/{id}` — get one by ID, including its `sms_fallback` and an `events` array (`SENT`, `DELIVERED`, `READ`, `FAILED`, each with `created_at` and `details`)

List query parameters, all optional:

| Parameter | Values |
|---|---|
| `date_range` | `last_week`, `last_thirty`, `last_month`, `all` (default), `custom_date` |
| `start_date`, `end_date` | RFC 3339, e.g. `2026-11-01T00:00:00Z` — required with `custom_date` |
| `limit` | 1 to 100, default 25 |
| `cursor` | `next_cursor` or `prev_cursor` from the previous response |
| `direction` | `next` (older, default) or `prev` (newer) |

Messages are in `data.messages`; paging is in `meta.pagination` (`next_cursor`, `prev_cursor`, `has_next`, `has_prev`). Stop when `has_next` is `false`.

A polled message has one of five statuses: `QUEUED`, `SENT`, `DELIVERED`, `READ`, `FAILED`. **Polling is coarser than webhooks:** these endpoints collapse `REJECTED`, `UNDELIVERABLE` and expiry into `FAILED`. If you need to know *why* a message failed, read the `RCS_STATUS` webhook event, not the polled status.

## Delivery events

Register a webhook (see `kudosity-webhooks`) for `RCS_STATUS` and `RCS_INBOUND` — and `SMS_STATUS` if you send with a fallback. `message_ref` is echoed in every status event, so set it to something you can join on — an order ID, an invoice number.

`RCS_STATUS` values:

| Status | Meaning |
|---|---|
| `SENT` | Submitted to the carrier |
| `DELIVERED` | Delivered to the recipient's device |
| `READ` | Read by the recipient, when the device and carrier report it |
| `REJECTED` | Rejected by the carrier or gateway — for example, a handset that can't receive RCS |
| `UNDELIVERABLE` | Reached the carrier but couldn't be delivered — handset unreachable, or the message expired |
| `FAILED` | Delivery failed with a carrier or handset error |
| `OTHER` | A status that doesn't map to a known value, passed through from the carrier |

RCS reports a status SMS never will: **`READ`**. It tells you the recipient actually opened the message, so handle it rather than treating `DELIVERED` as the end of the line.

A failed event can carry a `reason_code` such as `NOT_RCS_CAPABLE` or `EXPIRED`, or a `description`. The `description` is text for people to read — **match on `status` and `reason_code`, never on `description`.**

`RCS_INBOUND` covers free-text replies, tapped suggestions, and rich replies such as images and shared locations. `mo.content_type` says which: `TEXT`, `SUGGESTION`, `IMAGE`, `VIDEO`, `AUDIO`, `FILE`, `LOCATION` or `CONTACT`, with the content nested under `mo.content`. Note that `mo.sender` is the person who replied and `mo.recipient` is your agent.

## Errors

Errors follow RFC 9457 Problem Details under an `error` key:

```json
{ "error": { "type": "https://developers.kudosity.com/reference/errors#input-validation",
             "title": "Invalid Request", "detail": "Request validation failed", "status": 400,
             "issues": [{ "name": "sender", "message": "sender is required" }] } }
```

Read `error.issues[]`. **Validation can stop at the first problem**, so fix what it reports and retry to find any others. Each issue has a `message` and usually, but not always, the `name` of the field.

| Status | Meaning | Usual cause |
|---|---|---|
| 400 | Input validation | `sender` isn't an RCS agent on the account (or is a phone number); recipient in local format; `content` doesn't match `content_type`; text, a card or a suggestion over a limit; a media file that fails its check; `sms_fallback.message` too long for an SMS |
| 401 | Auth failed | Missing or wrong `x-api-key` |
| 404 | Not found | On send: `sms_fallback.sender` isn't a number or sender ID the account can send SMS from. On get: no message with that ID |
| 500 | Server error | Retry with backoff — but remember a retry of an accepted send is a second message |

## Related

- `kudosity-sms` — the fallback channel, and the right choice when you don't need rich content
- `kudosity-webhooks` — delivery status, replies, tapped suggestions, link hits
- Docs: [RCS Message Guide](https://developers.kudosity.com/docs/rcs-message-guide) (each format with a preview) · [Track an RCS message that falls back to SMS](https://developers.kudosity.com/docs/rcs-sms-fallback) · [RCS Agent Setup Guide](https://developers.kudosity.com/docs/rcs-onboarding)
- Working example: [`ai-payment-reminder-rcs`](https://github.com/kudosity/ai-agent-examples/tree/main/ai-payment-reminder-rcs) and [`ai-delivery-update-rcs`](https://github.com/kudosity/ai-agent-examples/tree/main/ai-delivery-update-rcs) (both plain text)
