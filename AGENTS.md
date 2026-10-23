# AGENTS.md

## Repo shape
One self-contained `index.html`: HTML, exactly one `<style>` block, two inline `<script>`s (script 1 = shell/lobby/audio/wallet/XP/lives persistence; script 2 = game logic). No build step, no `package.json`, no tests/lint. Never add tooling unless asked.
- `prompt/promptForSpeed.md` is the spec (rules, arenas, payouts, palette, fonts).
- Audio files live in `music/` and are referenced as `new Audio('music/...')`; nothing copies them.
- Scratch copies (`index.html.bak`, `index.html.test*`) in `test/` are not part of the game — ignore them.
- Reference PNGs (`pointinghand_*.png`, UUID pngs) in `other/` are not loaded by the game.

## Two scripts, shared global scope
Script 1 loads before script 2 and they share `window`. Script 1 defines `G`, `sfx`, `refreshIcons`, `setXp`, `levelOf`, `getLives`, `setLives`, etc. that script 2 uses — do not reorder or split the `<script>` tags. Top-level `function`/`let` are cross-script accessible.
- `let muted = false;` must be declared **before** any `muted` usage or a ReferenceError halts all JS.

## Rules
- No code comments anywhere in `index.html` (HTML, CSS, JS).
- Exactly one `<style>` open/close pair. A stray `</style>` renders CSS as visible text.
- Never name anything `top` — collides with `window.top`, throws SyntaxError, silently kills the script. Pile accessor is `topCard()`.
- Deck is **52 cards** (4 suits × ranks 1–13) even though `promptForSpeed.md` says 40 — the code's math requires 52.

## Life system (speed_lives_v1)
- `LIVES_KEY = 'speed_lives_v1'`, `MAX_LIVES = 5`, `fallbackLives = 5`.
- `getLives()` reads `localStorage`, returns saved value or 5. `setLives(v)` clamps to [0, 5], saves, and calls `updateLivesDisplay()`.
- `updateLivesDisplay()` calls `renderTiers()`, so `tierTrack`/`tierPrev`/`tierNext` **must be defined before** `updateLivesDisplay()` is invoked (ordering bug: calling it too early throws `TypeError` and kills all UI setup).
- Session reset: `livesSessionReset()` uses `sp_lives_seen` marker — lives reset to 5 on browser close only, preserved on refresh. Follows the same pattern as `tokenSessionReset`/`xpSessionReset`.
- `startMatch()` and the rematch handler both check `getLives() <= 0` and block entry (`sfx.bad()`). Tier buttons show "NO LIVES" and are disabled at 0 lives.
- "Buy 1 Life" shop pack (`data-life="1"`, class `.pack-life`) increments +1, auto-disables at 5.

## XP & Combo visual systems
Visual-only additions — gameplay rules, XP progression (`floor(xp/100)+1`), and combo mechanics are unchanged.
- Helper functions in script 2: `xpTier(a)` → `'small'|'normal'|'large'`; `comboTier(n)` → `'l2'|'l3'|'l5'`; `xpBurst(amount)`; `comboBurst(n)`; `spawnPixels(x,y,count,color,spread,cls)` (hard-edged square particles, auto-removed after 550ms); `flashXp()`; `flashCombo()`; `shakeHard()`; `levelUpCelebrate()`.
- `popXpPanel(amount)` now accepts and displays an XP amount.
- Level-up is detected by comparing `levelOf(progressXp)` before and after every `setXp()` call; call `levelUpCelebrate()` on increase. Call sites: `playFromHand` (+1), `drawOne` (+5), `doFlip` (+5), `registerCombo` (+5), `finishMatch` (+10).
- XP pop classes: `xp-small`, `xp-normal`, `xp-large`, `xp-levelup`. Combo pop classes: `combo-l2` (x2), `combo-l3` (x3–4), `combo-l5` (x5+). Combo announce word gets `combo-big` at x5+.
- Keep animations short and responsive; never block card gameplay.

## Verification
Parse-check both inline `<script>` blocks (silent output = OK):
`node -e "const h=require('fs').readFileSync('index.html','utf8');[...h.matchAll(/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/g)].forEach((m,i)=>{try{new Function(m[1])}catch(e){console.log('script',i+1,e.message)}})"`
Keep CSS braces balanced: `{` minus `}` across the single `<style>` block = 0.

Headless Chrome check (no server; Chrome won't exit — poll `out.html`, then `pkill -f "user-data-dir=..."`):
`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --user-data-dir=/tmp/chrome-profile --window-size=1920,1080 --virtual-time-budget=15000 --dump-dom file:///full/path/index.html > out.html 2>/dev/null &`
Timer/interval gameplay is unreliable headless (arena animations eat `--virtual-time-budget`) — force state directly instead of playing.

## Persistence / session reset
`localStorage`: `speed_tokens_v1`, `speed_xp_v1` (level = `floor(xp/100)+1`), `speed_bg_v1`, `speed_cb_v1`, `speed_name_v1`, `speed_lives_v1` (added for lives). Changing a key resets it for everyone.
`sessionStorage` markers (`sp_xp_seen`, `sp_tokens_seen`, `sp_name_seen`, `sp_cb_seen`, `sp_lives_seen`) gate `localStorage` clearing on **browser close only**; refresh preserves data, closing tab clears it.

## Interaction model (do not "improve")
- Clicking a card only selects it; both center piles pulse gold (`pulseGold`) regardless of validity; pile click plays. No validity hints, no auto-play.
- Drawing is **manual for the human**: `playFromHand` does not refill; player presses **Space** (`drawOne`, refills up to 5). Only the CPU auto-refills (`refill('a')` in `aiAct`). Do not make the player auto-draw.

## Audio
- Sound files live in `music/` and are loaded by relative path (`new Audio('music/Arcade Groove.mp3')`); nothing copies them. All `new Audio(...)` calls now use the `music/` prefix.
- Per-letter SPEED sounds: `S.mp3`, `P.mp3`, `E.mp3`, `D.mp3` are played in `appendSpeedLetter(letter)` via `new Audio('music/' + letter + '.mp3')`, guarded by `if(!muted)`.
- Music lifecycle: menu music (`music/Card Rush.mp3`) starts only on first user interaction via `tryStartMenuMusic` (`{once:true}` document listeners). Tournament music (`music/Arcade Groove.mp3`) starts in `startMatch()`; `backToLobby()` stops tournament music and restarts menu music. `tryStartMenuMusic` guards with `if(inTournament) return;` so a first click that enters an arena never also starts menu music (event bubbling would otherwise layer both).