# Kudosity Agent Skills

Agent Skills for the [Kudosity](https://kudosity.com) messaging platform — send SMS, MMS, WhatsApp and RCS, and manage contacts and webhooks, from inside your AI coding agent.

## What this is

A skill is a folder with a `SKILL.md` inside it: YAML frontmatter plus instructions the agent loads on demand. The format is an open standard, originally Anthropic's, now read by Claude Code, Cursor, Copilot, Windsurf, Zed, Cline, OpenCode and OpenClaw. One file, every agent.

Skills are not an MCP server. The MCP server ([`kudosity/mcp`](https://github.com/kudosity/mcp)) gives an agent *tools it can call*; skills give it *knowledge of how the API actually behaves* — which of the two APIs to use, why RCS sends through an agent ID instead of a phone number, why a WhatsApp message that returns `200` can still never arrive. They compose: an agent with both calls the right tool with the right arguments the first time.

## Install

Once published:

```bash
npx skills add kudosity/skills
```

Or clone into your agent's skills directory — `~/.claude/skills/`, `.cursor/skills/`, or `.agents/skills/` depending on the host.

## Skills

### Shipped

| Skill | Category | What it covers |
|---|---|---|
| `kudosity-setup` | Getting started | Account, credentials, which API needs which, getting a sender, verifying it works |
| `kudosity-sms` | Messaging | Single-recipient (V2) and list/bulk (V1) sends, scheduling, link tracking |
| `kudosity-mms` | Messaging | Image, GIF, video and audio attachments |
| `kudosity-rcs` | Messaging | RCS via agent ID, SMS fallback, capability check |
| `kudosity-whatsapp` | Messaging | Templates, the 24-hour service window, opt-in, SMS fallback |
| `kudosity-whatsapp-templates` | Messaging | Naming, positional parameters, locales, media/buttons via `custom` |
| `kudosity-contacts-lists` | Audience | Lists, members, opt-outs, bulk CSV import |
| `kudosity-webhooks` | Platform | Delivery status, inbound replies, link hits, opt-outs |

### Planned

`kudosity-api-overview`, `kudosity-reporting`, `kudosity-numbers-keywords`, `kudosity-troubleshooting`, `kudosity-ci-alerts`.

Contributions and issue reports are welcome — if a skill sends you down the wrong path, that's a bug worth filing.

`kudosity-setup` is where someone with no account or credentials starts, and it's the one to reach for on a `401`. It is **not** a prerequisite of the others — every skill above carries its own auth block and works installed on its own.

## Credentials

| Variable | Used by |
|---|---|
| `KUDOSITY_API_KEY` | Everything on the V2 API (`x-api-key` header) |
| `KUDOSITY_API_SECRET` | V1 API only (HTTP Basic, paired with the key) — lists, contacts, reporting, balance |

Both are in your Kudosity dashboard under **Settings → API Settings**.

Kudosity runs two APIs and they authenticate differently. That split is the single biggest source of confusion for a new integrator, which is why `kudosity-setup` exists as its own skill rather than a paragraph repeated in each one.

## Conventions

- **Every skill stands alone.** Install any one of these on its own and the task it names works end to end — endpoint, auth header, payload shape, error handling. No skill requires another to be present. This is deliberate: skills are loaded on demand by relevance, so a skill that only works when a sibling happens to be installed is a skill that silently half-works.
- **Frontmatter carries `metadata.version`, `category`, `tags` and `related`.** `related` is a **pointer, not a dependency** — it names skills a reader may also want. Nothing loads it automatically. (Sinch call this field `uses`, but no runtime acts on it, so naming it `uses` overstates what it does.)
- **Shared knowledge is repeated, not referenced.** The V2 auth header is one line; duplicating it in seven files costs nothing and keeps each one independent. Cross-references are for *depth* — "templates get complicated, here's the skill for that" — never for the basics a skill needs to function.
- **`User-Agent: kudosity-skills/<version>`** on every documented `curl`. This is deliberate — it lets [plugin attribution](https://github.com/kudosity/kudosity-ai-plugin-attribution) separate skill-driven traffic from the Claude Code plugin's.
- **The API is the source of truth.** Endpoint detail here was pulled from the live OpenAPI specs, not from memory. If a skill disagrees with the spec, the spec wins — re-pull rather than editing around it.

## Related

- [`kudosity/mcp`](https://github.com/kudosity/mcp) — MCP server, 19 tools
- [`kudosity/ai-agent-examples`](https://github.com/kudosity/ai-agent-examples) — runnable Node.js examples per channel
- [`kudosity/kudosity-claude-sms-plugin`](https://github.com/kudosity/kudosity-claude-sms-plugin) — Claude Code plugin (four of these skills originated there)
- [`kudosity/kudosity-sms-action`](https://github.com/kudosity/kudosity-sms-action) — GitHub Action for CI notifications
