# AGENTS.md

## Repo shape
One self-contained `speed.html`: HTML, exactly one `<style>` block, two inline `<script>`s sharing `window`. Script 1 (shell/lobby/audio/wallet/XP persistence) loads before script 2 (game logic). No build step, no `package.json`, no tests/lint. Never add tooling unless asked.
- `prompt/promptForSpeed.md` is the spec (rules, arenas, payouts, palette, fonts).
- Audio files live in `Music/` (capital M) but code references them as `new Audio('music/...')` — macOS is case-insensitive so this works; do not rename the folder.
- Scratch copies (`index.html.bak`, `index.html.test*`) in `test/` are not part of the game — ignore them.
- Reference PNGs (`pointinghand_*.png`, UUID pngs) in `other/` are not loaded by the game.

## Two scripts, shared global scope
Script 1 loads before script 2. Top-level `function`/`let` are cross-script accessible — do not reorder or split the `<script>` tags.
- `let muted = false;` must be declared **before** any `muted` usage or a ReferenceError halts all JS.

## Rules
- No code comments anywhere in `speed.html` (HTML, CSS, JS).
- Exactly one `<style>` open/close pair. A stray `</style>` renders CSS as visible text.
- Never name anything `top` — collides with `window.top`, throws SyntaxError, silently kills the script. Pile accessor is `topCard()`.
- Deck is **52 cards** (4 suits × ranks 1–13) even though `promptForSpeed.md` says 40 — the code's math requires 52.

## Interaction model (do not "improve")
- Clicking a card only selects it; both center piles pulse gold (`pulseGold`) regardless of validity; pile click plays. No validity hints, no auto-play.
- **Auto-draw on play**: `playFromHand()` now pushes `G.pDraw.pop()` into `G.pHand` (hand < 5, draw pile non-empty) and sets `G.drawAnim` for the slide-in animation. The Space key handler was removed; `Escape` still deselects. `drawOne()` still exists but is no longer bound to a key.
- Do not re-add manual drawing for the player. CPU auto-refills via `refill('a')` in `aiAct()`.

## Persistence / session reset
`localStorage` keys: `speed_tokens_v1` (reset 800), `speed_premium_v1` (reset 0), `speed_xp_v1` (level = `floor(xp/100)+1`), `speed_daily_v1` (wins/wows/speeds/claimed), `speed_name_v1` (default `PLAYER`), `speed_bg_v1` (default `void`), `speed_cb_v1` (default `classic`), `speed_avatar_v1`.
`sessionStorage` markers (`sp_xp_seen`, `sp_tokens_seen`, `sp_premium_seen`, `sp_daily_seen`, `sp_name_seen`, `sp_cb_seen`, `sp_avatar_seen`) gate `localStorage` clearing on **browser close only**; refresh preserves data, closing tab clears it.
- Changing a key resets it for everyone.
- There is **no** lives system (`speed_lives_v1`, `getLives`, `livesSessionReset` do not exist in this codebase) — do not reference them.

## Responsive CSS
Media queries added to the single `<style>` block:
- `max-width: 640px` — mobile: column lobby, compact wallet, smaller carousel, overlays use `transform: none` + adaptive offsets, `clamp()` card sizes, combo announce hidden.
- `641px–900px` — tablets: adjusted carousel scale, moderate margins.
- `max-height: 600px` + landscape — landscape phones: tighter spacing, smaller cards/fonts, combo announce hidden.
- `min-width: 1024px` — desktop: restores 3D carousel transforms.
Keep `clamp()` and relative sizing; avoid fixed dimensions that overflow viewports.

## Verification
Parse-check both inline `<script>` blocks (silent output = OK):
`node -e "const h=require('fs').readFileSync('speed.html','utf8');[...h.matchAll(/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/g)].forEach((m,i)=>{try{new Function(m[1])}catch(e){console.log('script',i+1,e.message)}})"`
Keep CSS braces balanced: `{` minus `}` across the single `<style>` block = 0.

Headless Chrome check (no server; Chrome won't exit — poll `out.html`, then `pkill -f "user-data-dir=..."`):
`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --user-data-dir=/tmp/chrome-profile --window-size=1920,1080 --virtual-time-budget=15000 --dump-dom file:///full/path/speed.html > out.html 2>/dev/null &`
Timer/interval gameplay is unreliable headless (arena animations eat `--virtual-time-budget`) — force state directly instead of playing.
