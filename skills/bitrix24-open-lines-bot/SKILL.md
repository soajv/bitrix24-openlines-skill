---
name: bitrix24-open-lines-bot
description: How to build a chatbot that answers inside an existing Bitrix24 Open Line (Открытая линия) — registration via imbot.v2, the incoming event contract, replying, and handing off to a human operator. Load this before building or debugging the Bitrix24 text-channel sales agent.
---

# Bitrix24 Open Lines chatbot

Source: official Bitrix24 REST API docs (`apidocs.bitrix24.com`, verified Sep 2026 — tutorial "Example of Creating a Chatbot for Open Channels", `imbot.v2` chatbot reference, `imopenlines` chat-bot method group). See `bitrix24-rest-api` skill first for general auth/retry/error-shape conventions this builds on.

## The two integration paths — pick the right one

- **A bot that answers *inside an already-existing* Open Line** (website widget, WhatsApp connector, etc. that the portal already has configured) — register via `imbot.v2.Bot.register` with `isSupportOpenline: true`. **This is almost certainly what "text agent in Open Lines" means** — it plugs into a channel that already exists, no new channel type to build.
- **A brand-new channel type** (e.g. integrating a messaging platform Bitrix doesn't natively support) — requires `imconnector.register` + `imconnector.activate` + `imconnector.connector.data.set`. Heavier, and not needed just to add an AI responder to an existing line. Don't reach for this unless the goal is literally "make Bitrix support a new external messenger."

**Confirmed empirically**: the entire `imconnector.*` family (including read-only `imconnector.status`/`imconnector.list`) rejects inbound-webhook auth outright with `WRONG_AUTH_TYPE: Current authorization type is denied for this method. Application context required` — this isn't a missing-scope problem, no webhook permission checkbox fixes it. Connectors (which channel/widget/messenger is wired into a line) can only be managed through a full OAuth app, or through the Bitrix24 UI itself (which runs in a privileged internal context). If you only have a webhook, you cannot add, inspect, or reuse a connector programmatically — that one piece has to be done by a portal admin in the UI (*Открытые линии → &lt;line&gt; → Каналы → добавить*). Everything else in this skill (bot registration, session control, line config incl. `WELCOME_BOT_ID`) works fine over a plain webhook — only the connector/channel layer is the exception.

v1 (`imbot.register`, event `ONIMBOTMESSAGEADD`) still works but is legacy. **Use v2 (`imbot.v2.*`, event `ONIMBOTV2MESSAGEADD`) for anything new** — it's what current docs recommend, and it's the version the examples below use. A lot of tutorials/StackOverflow answers online still show v1 syntax; don't mix the two.

## Prerequisites

- An inbound webhook (see `bitrix24-rest-api` skill) with **`imbot` and `imopenlines`** scopes checked — OAuth app install is *not* required for this path.
- A publicly reachable HTTPS URL for the bot's webhook handler (events arrive here as `application/x-www-form-urlencoded` POSTs).
- **A manual step in the Bitrix24 UI is required after registration**: an admin attaches the registered bot to the specific Open Line under *Contact Center → Open Channels*. There is no pure-API way to do this — confirm with whoever administers the target portal whether this has already been done, and for which Open Line.

**Confirmed the hard way (real portal, hours of debugging): `imopenlines.config.update` with `WELCOME_BOT_ID`/`WELCOME_BOT_ENABLE` does NOT do this.** That's a completely different, older feature (the queue "greeting bot" that plays before/while routing to a human operator) — it makes the line's timeline say "routed to &lt;bot&gt;" and is visible in `imopenlines.config.get`, but it **does not connect the bot to the `imbot.v2` event pipeline**. A bot attached only via `WELCOME_BOT_ID` will never receive `ONIMBOTV2MESSAGEADD` — not via webhook, not via fetch polling, no matter how long you wait. Verified by sending real messages through a real connector into a real line with `WELCOME_BOT_ENABLE=Y` and polling `imbot.v2.Event.get` repeatedly: always `events: []`.

The actual chat-bot binding this skill's flow depends on is a **separate UI-only control**, distinct from the queue/welcome-bot settings — inside the line's edit screen, look for a block specifically about the chat-bot/REST bot (not "Каналы", not "Очередь"/queue, not the welcome-bot toggle). Official docs describe it as *Contact Center → Open Channels → &lt;line&gt; → connect the bot*, without giving a scriptable method. If your bot's messages still never arrive after this, sanity-check with `imbot.v2.Bot.list` (`botToken` param) that `isSupportOpenline` and `eventMode` are what you expect — but the missing piece in practice was this UI binding, not the registration call.

## 1. Register the bot (once, idempotent)

```
imbot.v2.Bot.register
fields:
  code: "my_openline_bot"           # unique code within this webhook/app
  botToken: "<random ≤40 char secret>"
  type: "bot"                        # or "openline"
  isSupportOpenline: true
  eventMode: "webhook"
  webhookUrl: "https://<our-host>/openlines/webhook"
  properties:
    name: "My Openline Bot"
    workPosition: "Support"
    color: "green"
```
Calling this again with the same `fields` (including the same `botToken`) is safe/idempotent — it re-registers the same bot rather than creating a duplicate; run it on every service startup.

**Confirmed real response shape** (differs from what you might guess — the bot id is nested, not the top-level `result`):
```json
{
  "result": {
    "bot": { "id": 73268, "code": "my_openline_bot_test", "isSupportOpenline": true, "eventMode": "webhook", "...": "..." },
    "users": [{ "id": 73268, "bot": true, "type": "bot", "...": "..." }]
  }
}
```
Read the bot id as `result.bot.id` (the bot's Bitrix user id — same value also appears as `result.users[0].id`).

### Read-only ways to sanity-check a portal before writing anything

Useful for a first "does this webhook actually have the rights it claims" pass, with zero side effects:
- `imopenlines.config.list.get` (note the `.get` suffix — `imopenlines.config.list` alone is `ERROR_METHOD_NOT_FOUND`) — lists every Open Line on the portal with its full config (`LINE_NAME`, `ACTIVE`, `WELCOME_BOT_ID`, `CRM_CREATE`, etc.).
- `imbot.bot.list` (the v1 method — still works and is the simplest way to enumerate every bot registered on the portal, v1 and v2 alike) — returns `{result: {"<id>": {ID, NAME, CODE, OPENLINE}, ...}}`. `OPENLINE: "Y"` marks bots with Open Lines support.
- `profile` — returns the webhook's own user (id, name, `ADMIN` flag, timezone) — the cheapest possible "is this webhook alive" check.
- The generic `scope` method is **not reliable for inbound webhooks** — it returned `{"result": [""]}` (an empty-string entry, not a real scope list) against a real portal even though the webhook demonstrably had working `crm`/`imbot`/`imopenlines` access. Don't use it to verify webhook scopes; instead, call the actual method you need (e.g. `imopenlines.config.list.get`) and read the error: `insufficient_scope` means the webhook's permission checkboxes need editing (go back to *Разработчикам → Входящие вебхуки*, open the same webhook, tick the missing scope — no need to create a new one, the URL/token stays valid).

## 2. Incoming events

Bitrix POSTs form-urlencoded (not JSON body) to `webhookUrl`. Un-flatten it, then for the message event:

```json
{
  "event": "ONIMBOTV2MESSAGEADD",
  "data": {
    "bot": {"id": 456, "code": "my_openline_bot"},
    "message": {"id": 790, "chatId": 112, "authorId": 27, "text": "..."},
    "chat": {"id": 112, "dialogId": "chat112", "type": "lines", "entityType": "LINES"},
    "user": {"id": 27, "name": "..."}
  },
  "auth": {"domain": "<portal>.bitrix24.com", "application_token": "custom<botToken>"}
}
```

**Verify every request**: `auth.application_token === "custom" + botToken`. Reject (403) anything else — this is the only authenticity check available, there's no HMAC signature like the ElevenLabs webhook has.

**Filter to Open Lines chats**: only act when `data.chat.entityType === "LINES"` — the same bot process may receive other chat events too.

The bot receives **every** client message in the line it's attached to automatically — no `@mention` needed, unlike a bot in a regular group chat.

All scalar values arrive as strings (it's form-encoded) — cast explicitly (`chat.id` to int, etc.).

## 3. Reply to the client

```
imopenlines.bot.session.message.send
CHAT_ID: <chat.id from the event>
MESSAGE: "<reply text>"
NAME: "DEFAULT"    # or "WELCOME" for the first automated greeting
```

## 4. Hand off / end the session

- `imopenlines.bot.session.operator` — `CHAT_ID` only; hands off to whichever operator on the line is free.
- `imopenlines.bot.session.transfer` — `CHAT_ID`, plus either `USER_ID` (specific person) or `QUEUE_ID`, `LEAVE: "Y"|"N"` (whether the bot itself leaves the dialog), `CLIENT_ID: botToken` (required).
- `imopenlines.bot.session.finish` — `CHAT_ID`, `CLIENT_ID: botToken`; closes the dialog (e.g. after the client says thanks / the deal is wrapped up).

All three require `imopenlines` scope on the webhook and return `{"result": true}` on success.

## Putting it together

- A typical bot lives entirely inside the `ONIMBOTV2MESSAGEADD` handler: fetch/create a conversation session keyed by `chat.id` → run one turn of your agent/LLM loop with the client's `text` → any tool calls your agent makes are just regular API/service calls, unrelated to Bitrix → reply via `imopenlines.bot.session.message.send`.
- If a request needs a human decision that isn't urgent for the client (e.g. an approval a manager has to make), it doesn't have to mean handing off the live chat — you can track that as a separate async task (e.g. a `tasks.task.add`, see the `bitrix24-rest-api` skill) while the bot keeps the conversation going with the client. Reach for `imopenlines.bot.session.operator`/`transfer` only when the *client* needs a human in the conversation right now.
- Before building against a specific portal, confirm two things with whoever administers it: whether a live Open Line is already configured (and via which connector — website widget, WhatsApp, etc.), and whether a bot has ever been attached to it before. This skill covers the mechanism; it doesn't tell you the current state of any particular portal.
