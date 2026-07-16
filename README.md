# yandex-sdk-integration — an AI skill for integrating Yandex Games

A portable skill that helps an AI assistant integrate and publish games for
**Yandex Games (Я.Игры)**: Yandex Games SDK integration
(ads, leaderboards, cloud saves, purchases) for **browser/HTML5, Three.js with
TypeScript/Vite, Godot, and Unity**, plus the publishing and moderation rules.

Its first rule: **always fetch Yandex's current rules and SDK docs before
building** — they change, so the skill points the AI at the live docs instead of
trusting frozen copies.

## Use it with any AI

- **Claude Code:** copy this folder to `~/.claude/skills/yandex-sdk-integration/`.
  It auto-activates from the `SKILL.md` description.
- **Cursor / other rule-based tools:** point a rule at `SKILL.md` (and the
  relevant `references/*.md`).
- **Any chat (ChatGPT, Gemini, …):** attach or paste the relevant file —
  `SKILL.md` for the workflow, a `references/*.md` for depth.

## What's inside

- `SKILL.md` — the workflow and a router to the references.
- `references/` — live-rules hub, SDK reference, Three.js/Vite and other
  per-engine guides, publishing.
- `assets/ya-sdk.js` — a copy-paste Yandex SDK wrapper for HTML5 games.
- `assets/serve-yandex-local.mjs` — a loopback-only local server for a Yandex
  build and the official development SDK adapter.

## License

MIT

## Support

If this project is useful to you, you can support the original developer with a crypto tip — thank you!

**USDT — Ethereum (ERC-20):**

`0xad39bdf2df0b8dd6991150fcea0a156150ed19b8`

[View / verify on Etherscan](https://etherscan.io/address/0xad39bdf2df0b8dd6991150fcea0a156150ed19b8)

> Send only on the **Ethereum (ERC-20)** network.
