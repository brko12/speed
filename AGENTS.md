# AGENTS.md

## Repo shape
- One self-contained `index.html`: HTML, a single `<style>`, two inline `<script>`s. No build step, no `package.json`, no tests/lint. Never add tooling unless asked.
- `promptForSpeed.md` is the spec (rules, arenas, payouts, palette, fonts). Loose `Gemini_*.jpg` / `pointinghand_*.png` are reference assets, not loaded by the game.
- Local audio: `Arcade Groove.mp3`, `Card Rush.mp3`, `firework.wav`, `drawCard.wav`, `flip.wav`, `click.wav`, `WOW.mp3`, `cool.mp3`, `speed.mp3`, `drop.wav`, `gameOver.wav`, `youWin.wav`.
- Scratch copies (`index.html.bak`, `index.html.test*`) are not part of the game — ignore them.

## Two scripts, shared global scope
- Script 1 = shell/lobby/audio/wallet/XP persistence. Script 2 = game logic.
- Top-level `function`/`let` are cross-script accessible (`G`, `sfx`, `refreshIcons`, `setXp`, `levelOf`, etc.).
- No code comments anywhere in `index.html` (HTML, CSS, JS). Do not add any.

## Verifying changes
- Parse-check both inline `<script>` blocks (silent output = OK):
  `node -e "const h=require('fs').readFileSync('index.html','utf8');[...h.matchAll(/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/g)].forEach((m,i)=>{try{new Function(m[1])}catch(e){console.log('script',i+1,e.message)}})"`
- Keep CSS braces balanced: `{` minus `}` across the single `<style>` block = 0.
- Headless Chrome check (no server; Chrome won't exit — poll `out.html`, then `pkill -f "user-data-dir=..."`):
  `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --user-data-dir=/tmp/chrome-profile --window-size=1920,1080 --virtual-time-budget=15000 --dump-dom file:///full/path/index.html > out.html 2>/dev/null &`
- Timer/interval gameplay is unreliable headless (arena animations eat `--virtual-time-budget`) — force state directly instead of playing.

## Hard-won gotchas (silent failures)
- Never name anything `top` — collides with `window.top`, throws SyntaxError, silently kills the script. Pile accessor is `topCard()`.
- Exactly one `<style>` open/close pair. A stray `</style>` renders game CSS as visible text.
- Deck is **52 cards** even though promptForSpeed.md says 40 — the code's math requires 52.
- `let muted = false;` must be declared **before** any `muted` usage or a ReferenceError halts all JS.
- Screen-shake specificity trap: never add `animation` to `#game` by id — id specificity (1,0,0) beats `.screen-shake` (0,1,0) and shake silently stops.

## Persistence / session reset
- `localStorage`: `speed_tokens_v1` (tokens), `speed_xp_v1` (total XP; level = `floor(xp/100)+1`), `speed_bg_v1`, `speed_cb_v1`, `speed_name_v1`. Changing a key resets it for everyone.
- Four `sessionStorage` markers (`sp_xp_seen`, `sp_tokens_seen`, `sp_name_seen`, `sp_cb_seen`) gate `localStorage` clearing on **browser close only**; refresh preserves data, closing tab clears it.

## Interaction model (do not "improve")
- Clicking a card only selects it; both center piles pulse gold (`pulseGold`) regardless of validity; pile click plays. No validity hints, no auto-play.
- Drawing is **manual for the human**: `playFromHand` does not refill; player presses **Space** (`drawOne`, refills up to 5). Only the CPU auto-refills (`refill('a')` in `aiAct`). Do not make the player auto-draw.

## Draw animation (script 2)
- `drawOne()` sets `G.drawAnim = { idx }` before `renderHand()`.
- `renderHand()` runs a FLIP pass: captures each card rect, rebuilds the hand via the existing flex layout, then applies an inverted `translate3d` that transitions out via the existing `.15s` ease-out so existing cards smoothly shift to make room.
- New card gets `.hand .card.drawing`, positioned from the draw-stack `.draw-stack .card.back` rect to its hand slot, random −18°…18° rotation, `z-index:6`. Travel ≈300ms (`drawTravel` ease-out), then ≈100ms scale overshoot land (`drawLand`). A 420ms timeout removes `.drawing` and inline transforms.
- Guarded for detached elements and successive draws; does not touch click/hover/play logic or screen-shake/`+1 XP` pop.
