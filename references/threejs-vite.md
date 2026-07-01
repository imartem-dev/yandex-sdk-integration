# Three.js + TypeScript + Vite

> Read the current Yandex requirements and SDK reference before using these
> patterns. This guide covers the Three.js/Vite integration boundary, not the
> complete SDK. Use `yandex-sdk.md` for saves, leaderboards, purchases, and ads.

## Contents

1. SDK bootstrap
2. Integration boundary
3. Loading and gameplay lifecycle
4. Ads
5. Vite build and assets
6. Canvas, resize, and mobile behavior
7. Local and draft testing
8. Three.js release checklist

## 1. SDK bootstrap

For a ZIP uploaded to Yandex, load the platform-provided SDK before the Vite
entry module:

```html
<head>
  <script src="/sdk.js"></script>
</head>
<body>
  <div id="game"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
```

`/sdk.js` is the intentional exception to relative game asset paths. Do not
download it into the project. Use the official development proxy locally.

Initialize the SDK before calling any SDK method. Load the game assets, create
the renderer, and render the first usable frame before hiding the Yandex loader:

```ts
const ysdk = await window.YaGames.init();

await loadGameAssets();
startGame();
renderer.render(scene, camera);

ysdk.features.LoadingAPI?.ready();
```

Call `LoadingAPI.ready()` once, when the player can actually interact. Do not
silently ignore an SDK initialization failure in a Yandex production build.

## 2. Integration boundary

Keep the SDK outside the Three.js scene graph and simulation. Create one small
application service that owns the initialized `ysdk`, ad state, player data,
and platform lifecycle. Expose game-oriented operations such as:

```ts
export interface PlatformService {
  readonly language: string;
  markPlayable(): void;
  gameplayStart(): void;
  gameplayStop(): void;
  showFullscreenAd(): Promise<boolean>;
  showRewardedAd(): Promise<boolean>;
}
```

Define only the Yandex TypeScript types used by the project, or use the current
official TypeScript declarations. Do not spread `window.YaGames` checks across
rendering, UI, and gameplay modules. If standalone development needs a mock,
inject a separate local implementation instead of weakening production errors.

## 3. Loading and gameplay lifecycle

Track pause causes independently so an ad closing cannot resume a game whose
tab is still hidden. Typical causes are `ad`, `visibility`, and an in-game menu.
When any cause is active:

- stop simulation and input processing;
- suspend or mute every game audio source;
- call `ysdk.features.GameplayAPI?.stop()` when active gameplay stops;
- optionally keep rendering a static paused frame.

When the last cause is removed:

- reset `THREE.Clock` or the previous RAF timestamp before the next simulation
  step, and clamp the first delta as a second guard;
- restore audio only when the user has not muted it and browser policy permits;
- call `ysdk.features.GameplayAPI?.start()` when active gameplay resumes.

```ts
const pauseReasons = new Set<string>();
let lifecyclePaused = false;

function setPaused(reason: string, paused: boolean): void {
  if (paused) pauseReasons.add(reason);
  else pauseReasons.delete(reason);

  const shouldPause = pauseReasons.size > 0;
  simulationPaused = shouldPause;
  if (shouldPause === lifecyclePaused) return;
  lifecyclePaused = shouldPause;

  if (shouldPause) {
    clock.stop();
    void audioContext.suspend();
    ysdk.features.GameplayAPI?.stop();
  } else {
    clock.start();
    void audioContext.resume().catch(() => {});
    ysdk.features.GameplayAPI?.start();
  }
}

document.addEventListener("visibilitychange", () => {
  setPaused("visibility", document.hidden);
});
window.addEventListener("blur", () => setPaused("visibility", true));
window.addEventListener("focus", () => {
  setPaused("visibility", document.hidden);
});
```

Make pause/resume transitions idempotent in the real implementation so repeated
focus events do not repeatedly suspend audio or emit gameplay events. Keep the
RAF delta bounded even after device sleep:

```ts
const deltaSeconds = Math.min(clock.getDelta(), 0.05);
if (!simulationPaused) updateSimulation(deltaSeconds);
renderer.render(scene, camera);
```

## 4. Ads

Allow only one ad request at a time. Pause before requesting an ad, not after an
`onOpen` callback that may arrive after the ad becomes visible. Clear the ad
pause in both `onClose` and `onError`; the pause-reason set prevents an incorrect
resume while the document remains hidden.

For rewarded ads, grant the reward exactly once and only from `onRewarded`.
Closing a video is not proof that the reward was earned.

```ts
let adInFlight = false;

function showRewardedAd(grantReward: () => void): void {
  if (adInFlight) return;
  adInFlight = true;
  let rewarded = false;
  let finished = false;
  setPaused("ad", true);

  const finish = () => {
    if (finished) return;
    finished = true;
    adInFlight = false;
    setPaused("ad", false);
  };

  ysdk.adv.showRewardedVideo({ callbacks: {
    onRewarded: () => {
      if (rewarded) return;
      rewarded = true;
      grantReward();
    },
    onClose: finish,
    onError: finish,
  }});
}
```

Use the same lock and lifecycle for fullscreen ads. Show them only at logical
breaks. Save important progress before an ad can navigate the player away.

## 5. Vite build and assets

Use a dedicated Yandex build mode so other hosting targets can keep their own
base path:

```ts
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig(({ mode }) => ({
  base: mode === "yandex" ? "./" : "/",
}));
```

```json
{
  "scripts": {
    "build:yandex": "vite build --mode yandex"
  }
}
```

- Import bundled assets or resolve them with `new URL(path, import.meta.url)`.
- For `public/` assets, prefix paths with `import.meta.env.BASE_URL`.
- Do not hardcode `/assets/...` or a deployment subdirectory.
- Do not replace `/sdk.js` with an absolute Yandex S3 URL in an uploaded build.
- ZIP the contents of `dist`, not the `dist` directory itself, so `index.html`
  is at the archive root.

After building, inspect `dist/index.html` and search generated files for stale
development hosts, absolute game asset paths, and unauthorized external hosts.

## 6. Canvas, resize, and mobile behavior

Size the renderer from its actual container, not only `window.innerWidth`. The
game runs inside an iframe and the available area can change because of device
rotation, browser UI, fullscreen transitions, and platform banners.

```ts
const resizeObserver = new ResizeObserver(() => {
  const width = Math.max(1, container.clientWidth);
  const height = Math.max(1, container.clientHeight);

  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(width, height, false);
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
});
resizeObserver.observe(container);
```

Adapt the pixel-ratio cap to the project's GPU budget. Ensure the page and game
container fill the available area without browser scrolling:

```css
html, body, #game {
  width: 100%;
  height: 100%;
  margin: 0;
  overflow: hidden;
  overscroll-behavior: none;
}

canvas {
  display: block;
  touch-action: none;
  user-select: none;
  -webkit-user-select: none;
}
```

Prevent `contextmenu` on the game surface and prevent long-press selection.
Declare the supported orientation in the Yandex draft. Test the opposite
orientation and fullscreen transitions; do not stretch the canvas to the wrong
aspect ratio. Keep touch controls usable without page scrolling or browser
keyboard shortcuts.

## 7. Local and draft testing

Run Vite, then route it through the official SDK proxy:

```bash
npm run dev
npx @yandex-games/sdk-dev-proxy -h http://localhost:5173 --dev-mode=true
```

The dev environment supplies SDK mocks and does not require a registered draft.
Use mock URL parameters to test authorization and orientation, for example
`lockedOrientation: "landscape"`. Test both success and error callbacks for ads,
authorization, saves, and purchases.

Before moderation, repeat the test with a real draft and production mode. Open
the Yandex debug panel or add `&debug-mode=16`, verify SDK initialization, and
test under the iframe/CSP conditions used by the platform.

Official pages to re-check:

- Connection: https://yandex.com/dev/games/doc/en/sdk/sdk-about
- Local launch: https://yandex.com/dev/games/doc/en/concepts/local-launch
- Loading/gameplay events: https://yandex.com/dev/games/doc/en/sdk/sdk-game-events
- Requirements: https://yandex.com/dev/games/doc/en/concepts/requirements

## 8. Three.js release checklist

- `LoadingAPI.ready()` fires once after assets and the first playable frame.
- Simulation, controls, particles that affect gameplay, timers, and all audio
  stop while hidden and during fullscreen or rewarded ads.
- Resume does not produce a large physics delta or duplicate RAF loop.
- Reward callbacks cannot grant twice and simultaneous ads are rejected.
- Resize, orientation changes, and fullscreen preserve camera aspect and UI.
- No page scroll, swipe-to-refresh, long-press selection, or context menu appears.
- All GLTF, textures, audio, fonts, WASM, and worker URLs load from the built ZIP.
- DevTools shows no WebGL context, shader, CORS, asset, or unhandled promise errors.
- The production ZIP has `index.html` at its root and works through the draft.
