# AGENTS.md

## Repo shape
One self-contained `index.html`: HTML, exactly one `<style>` block, two inline `<script>`s (script 1 = shell/lobby/audio/wallet/XP persistence; script 2 = game logic). No build step, no `package.json`, no tests/lint. Never add tooling unless asked.
- `promptForSpeed.md` is the spec (rules, arenas, payouts, palette, fonts).
- Audio files live in `music/` and are referenced by relative path (`new Audio('music/Arcade Groove.mp3')`); nothing copies them.
- Scratch copies (`index.html.bak`, `index.html.test*`) are not part of the game — ignore them.
- Reference PNGs (`pointinghand_*.png`, UUID pngs) are not loaded by the game.

## Two scripts, shared global scope
Script 1 loads before script 2 and they share `window`. Script 1 defines `G`, `sfx`, `refreshIcons`, `setXp`, `levelOf`, etc. that script 2 uses — do not reorder or split the `<script>` tags. Top-level `function`/`let` are cross-script accessible.

## Rules
- No code comments anywhere in `index.html` (HTML, CSS, JS).
- Exactly one `<style>` open/close pair. A stray `</style>` renders CSS as visible text.
- Never name anything `top` — collides with `window.top`, throws SyntaxError, silently kills the script. Pile accessor is `topCard()`.
- Deck is **52 cards** (4 suits × ranks 1–13) even though `promptForSpeed.md` says 40 — the code's math requires 52.
- `let muted = false;` must be declared **before** any `muted` usage or a ReferenceError halts all JS.

## Verifying changes
Parse-check both inline `<script>` blocks (silent output = OK):
`node -e "const h=require('fs').readFileSync('index.html','utf8');[...h.matchAll(/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/g)].forEach((m,i)=>{try{new Function(m[1])}catch(e){console.log('script',i+1,e.message)}})"`
Keep CSS braces balanced: `{` minus `}` across the single `<style>` block = 0.

Headless Chrome check (no server; Chrome won't exit — poll `out.html`, then `pkill -f "user-data-dir=..."`):
`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --user-data-dir=/tmp/chrome-profile --window-size=1920,1080 --virtual-time-budget=15000 --dump-dom file:///full/path/index.html > out.html 2>/dev/null &`
Timer/interval gameplay is unreliable headless (arena animations eat `--virtual-time-budget`) — force state directly instead of playing.

## Persistence / session reset
`localStorage`: `speed_tokens_v1`, `speed_xp_v1` (level = `floor(xp/100)+1`), `speed_bg_v1`, `speed_cb_v1`, `speed_name_v1`. Changing a key resets it for everyone.
Four `sessionStorage` markers (`sp_xp_seen`, `sp_tokens_seen`, `sp_name_seen`, `sp_cb_seen`) gate `localStorage` clearing on **browser close only**; refresh preserves data, closing tab clears it.

## Interaction model (do not "improve")
- Clicking a card only selects it; both center piles pulse gold (`pulseGold`) regardless of validity; pile click plays. No validity hints, no auto-play.
- Drawing is **manual for the human**: `playFromHand` does not refill; player presses **Space** (`drawOne`, refills up to 5). Only the CPU auto-refills (`refill('a')` in `aiAct`). Do not make the player auto-draw.

## Audio
- Sound files live in `music/` and are loaded by relative path (`new Audio('music/Arcade Groove.mp3')`); nothing copies them.
- Per-letter SPEED sounds: `S.mp3`, `P.mp3`, `E.mp3`, `D.mp3` are played in `appendSpeedLetter(letter)` via `new Audio('music/' + letter + '.mp3')`, guarded by `if(!muted)`.
- All `new Audio(...)` calls respect `muted`; `let muted = false;` is declared in script 1 before any `muted` usage.
- Music lifecycle: menu music (`music/Card Rush.mp3`) starts only on first user interaction via `tryStartMenuMusic` (`{once:true}` document listeners). Tournament music (`music/Arcade Groove.mp3`) starts in `startMatch()`; `backToLobby()` stops tournament music and restarts menu music. `tryStartMenuMusic` guards with `if(inTournament) return;` so a first click that enters an arena never also starts menu music (event bubbling would otherwise layer both).

## Draw animation (script 2)
- `drawOne()` sets `G.drawAnim = { idx }` before `renderHand()`.
- `renderHand()` runs a FLIP pass: captures each card rect, rebuilds the hand via the existing flex layout, then applies an inverted `translate3d` that transitions out via the existing `.15s` ease-out so existing cards smoothly shift to make room.
- New card gets `.hand .card.drawing`, positioned from the draw-stack `.draw-stack .card.back` rect to its hand slot, random −18°…18° rotation, `z-index:6`. Travel ≈300ms (`drawTravel` ease-out), then ≈100ms scale overshoot land (`drawLand`). A 420ms timeout removes `.drawing` and inline transforms.
- Guarded for detached elements and successive draws; does not touch click/hover/play logic or screen-shake/`+1 XP` pop.

## SPEED letter sequence (script 2)
- Triggered only in `playFromHand()` — the player's successful card-play event. Never in render/draw/hover/frame loops.
- State lives on `G`: `speedActive`, `speedPlayed`, `speedLetters`. Reset each round in `createState()` (`startRound()`).
- `const SPEED_SEQ = ['S','P','E','E','D'];` — one letter appended per play while `speedPlayed < 5`; only the new letter animates (`#speedLetters .speed-letter.speed-new`).
- Container: `#speedLetters` fixed beside `#roundScore` (middle-left). `appendSpeedLetter(letter)` creates one `div`; existing letters stay static. Each call also plays `'music/' + letter + '.mp3'` (S/P/E/E/D), muted-aware.
- `pDraw` (the deck) decreases **only** in `drawOne()` via `G.pDraw.pop()`. `refill('p')` is never called; `refill('a')` is CPU-only. So deck reaches 0 only through `drawOne`.
- Player total remaining = `G.pHand.length + G.pDraw.length` (shown in `youRemain`).

## Inactivity countdown (script 2)
- After **5 seconds** of player inactivity (no card played), a countdown bar appears above the player hand (`#countdownWrap`) showing `5 → 4 → 3 → 2 → 1` (`#countdownNum`) with a draining red bar (`#countdownFill`).
- Timer starts in `startRound()` once `G.started = true`; cleared in `clearTimers()`; guarded by `G.over` so it never fires after round end.
- Every successful `playFromHand` calls `resetInactivityTimer()` — stops/hides the countdown and restarts the 5s inactivity window. Drawing a card does **not** reset it.
- Numbers 2 and below turn crimson and pulse (`warn`); number 1 adds a faster pulse + shake (`urgent`).
- If the countdown reaches `1`, `endGame('ai')` is called (player loses the round) and further plays are blocked by `ready()`.
- DOM: `#countdownWrap`, `#countdownBar`, `#countdownFill`, `#countdownNum`.

