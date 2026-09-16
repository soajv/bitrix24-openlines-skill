# bitrix24-open-lines-bot

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) with empirically-verified knowledge for building a chatbot that answers inside an existing Bitrix24 Open Line (Открытая линия) — registration via `imbot.v2`, the incoming event contract, replying, and handing off to a human operator.

This isn't a bot framework — it's a skill file (`SKILL.md`) that an AI coding agent loads into context before writing or debugging a Bitrix24 Open Lines integration, capturing the parts of this feature that are either undocumented, easy to misconfigure, or actively misleading if you go by official tutorials alone.

## Why

Bitrix24's chatbot/Open Lines stack has more than one way to look correctly configured while silently not working (see "the `WELCOME_BOT_ID` trap" below). Most of what's in this skill was confirmed against a real portal — sending real messages through a real connector and observing what actually arrives — not just copied from docs. Findings are dated so you can judge whether Bitrix24's behavior may have since changed.

## What's covered

- The two integration paths and which one you actually want: a bot inside an *existing* Open Line (`imbot.v2.Bot.register`) vs. registering a brand-new channel/connector type (`imconnector.*`) — and why the latter flatly rejects webhook auth (`WRONG_AUTH_TYPE`, OAuth-app-only)
- Why `imopenlines.config.update` with `WELCOME_BOT_ID`/`WELCOME_BOT_ENABLE` looks like it attaches your bot to the line but does not — that's a separate, older "queue greeting bot" feature that never triggers `ONIMBOTV2MESSAGEADD`
- The bot registration call, its real (nested, not top-level) response shape, and how to make it idempotent across restarts
- Read-only, zero-side-effect ways to sanity-check a webhook's actual permissions before writing anything
- The incoming event contract (`ONIMBOTV2MESSAGEADD`), including the only authenticity check available (`auth.application_token`) and how to filter to Open Lines chats specifically
- Replying (`imopenlines.bot.session.message.send`) and handing off/closing a session (`.operator`, `.transfer`, `.finish`)
- A generic pattern for wiring this into an agent/LLM-driven conversation loop

See [`skills/bitrix24-open-lines-bot/SKILL.md`](./skills/bitrix24-open-lines-bot/SKILL.md) for the full content.

## Related skill

This skill assumes [`bitrix24-rest-api`](https://github.com/soajv/bitrix24-api-skill) for general auth/retry/error-shape conventions — load that one first.

## Using this skill

The skill itself lives in [`skills/bitrix24-open-lines-bot/`](./skills/bitrix24-open-lines-bot/SKILL.md).

**Claude Code:** copy or symlink that directory into your skills path (e.g. `.claude/skills/bitrix24-open-lines-bot/` for a project-local skill, or the equivalent global skills directory), or clone this repo and point at it directly:

```sh
git clone https://github.com/soajv/bitrix24-openlines-skill.git
cp -r bitrix24-openlines-skill/skills/bitrix24-open-lines-bot .claude/skills/bitrix24-open-lines-bot
```

**Other agents/tools:** `skills/bitrix24-open-lines-bot/SKILL.md` is plain Markdown with a YAML frontmatter header (`name`, `description`) — feed it into any system prompt or context-loading mechanism that accepts freeform reference material.

## Contributing

Corrections and additional confirmed gotchas are welcome. Please:

- Verify claims against a real portal where possible (ideally with a real connector, not just the API), and say what you tested and when.
- Cite the specific official docs page you're relying on, if any.
- Keep the tone of existing entries: state the behavior, the evidence for it, and how to work around it — not just a link.

## Disclaimer

This project is unaffiliated with, and not endorsed by, Bitrix24 or 1C-Bitrix. "Bitrix24" is a trademark of its respective owner. Information here reflects behavior observed as of the dates noted in the text and may not be current — always cross-check against [official docs](https://apidocs.bitrix24.com/) for anything security- or billing-sensitive.

## License

[MIT](./LICENSE)
