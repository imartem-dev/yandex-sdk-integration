---
name: yandex-sdk-integration
description: Use when integrating, testing, packaging, or publishing a game for Yandex Games (Я.Игры), including the Yandex Games SDK, ads, cloud saves, purchases, platform lifecycle, HTML5/Three.js/Vite builds, Godot or Unity exports, local SDK mocks, drafts, and moderation. ALWAYS fetch the current Yandex rules and SDK docs first.
---

# Integrating games with Yandex Games (Я.Игры)

## Step 0 — Fetch the CURRENT rules first (do not skip)

Yandex changes its requirements, moderation rules, and SDK. The notes in this
skill are a starting checklist, **not** the source of truth. Before writing game
code, open and read the live docs:

- Docs hub (RU): https://yandex.ru/dev/games/doc/dg/
- Docs hub (EN): https://yandex.com/dev/games/doc/en/

Read the current **requirements**, **moderation rules**, and **SDK reference**.
`references/rules-and-requirements.md` lists exactly what to check and the deep links.

## Universal workflow

1. **Fetch current rules** (Step 0).
2. **Pick the engine** → open the matching integration reference below.
3. **Keep platform SDK calls behind a focused service or adapter.**
4. **Integrate the SDK** — at minimum `YaGames.init()` + `LoadingAPI.ready()`;
   then cloud saves / leaderboards / ads / purchases as needed. For HTML5 you can
   drop in `assets/ya-sdk.js`.
5. **Test locally with SDK mocks**, then test as a draft in the developer console.
6. **Package the ZIP** (`index.html` in the root) and **upload**.
7. **Pass moderation** — run the rejection-reasons checklist first.

## References

| Topic | File |
|---|---|
| Live rules, requirements, moderation | `references/rules-and-requirements.md` |
| Yandex Games SDK (ads, saves, leaderboards, purchases) | `references/yandex-sdk.md` |
| Browser / HTML5 (JS, Phaser, PixiJS, Construct) | `references/browser-html5.md` |
| Three.js + TypeScript + Vite | `references/threejs-vite.md` |
| Godot 4 (Web export) | `references/godot.md` |
| Unity (WebGL) | `references/unity.md` |
| Publishing, console, localization | `references/publishing.md` |
| Copy-paste SDK wrapper | `assets/ya-sdk.js` |
