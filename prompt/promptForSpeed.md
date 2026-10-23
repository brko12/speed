# AI Development Prompt: Speed (Card Game) - Interactive Single-File Prototype

## Objective
Act as an expert game developer and front-end engineer. Create a playable, single-file HTML5/JavaScript prototype for **Speed**, a real-time, 2D retro pixel-art card game. The prototype will use virtual **Game Tokens** instead of real money, featuring an in-game shop, tournament selection, and a single-player mode against an AI opponent.

---

## 1. Prototype Features & Architecture

### A. Tech Stack Constraints
* **Single File Mandate:** Write all HTML, CSS, and JavaScript inside a single self-contained `.html` file.
* **Styling:** Use Tailwind CSS (via CDN) combined with CSS pixelation rules (`image-rendering: pixelated;`) for a clean, retro 8-bit / 16-bit arcade look.
* **Icons & Font:** Use Google Fonts (`Press Start 2P` or `VT323`) and Lucide Icons (via CDN) for authentic pixel gaming vibes.

### B. Game Economy (Tokens & Tournaments)
1. **Token Wallet:**
   * Player starts with a default balance of **50 Tokens**.
   * Balance is persistent in `localStorage`.
2. **Main Menu / Lobby Screen:**
   * **Wallet Section:** Displays current token balance. Includes a "Free Tokens" button (+25 tokens) and a "Shop" modal to simulate adding tokens.
   * **Tournament Selector Cards:**
     * **Bronze Arena:** Entry Fee = 5 Tokens | Prize Pool = 10 Tokens
     * **Silver Arena:** Entry Fee = 15 Tokens | Prize Pool = 30 Tokens
     * **Gold Arena:** Entry Fee = 50 Tokens | Prize Pool = 100 Tokens
   * Entry fee is deducted upon joining a tournament match. If the player wins, they earn the full prize pool. If lost, entry fee is lost.

---

## 2. Speed Game Mechanics & Rules

### A. Setup
* **Deck Setup:** Total deck size = 40 cards (Ranks 1 to 13, repeated across suits, simplified for pixel clarity).
* **Player Decks:** Player 1 (Human) gets 20 cards total; Player 2 (AI) gets 20 cards total.
* **Hands:** Both human and AI draw 5 cards into their hand. The remaining 15 cards stay in their personal draw pile.
* **Center Table:**
  * 2 Center Active Piles (starts face-down, flipped when match begins).
  * 2 Center Backup Piles (5 cards each) to resolve deadlocks.

### B. Real-Time Gameplay Logic (Simultaneous Action)
* **No Turn System:** Both Human and AI play cards continuously in real-time.
* **Card Play Logic:**
  * A card from a player's hand can be placed on either center active pile if its rank is **+1 or -1** of the pile's top card.
  * Ace (1) wraps around to King (13) and 2 ($13 \leftrightarrow 1 \leftrightarrow 2$).
* **Auto-Refill Hand:** Playing a card automatically draws a replacement from the player's personal 15-card draw pile into their 5-card hand until the draw pile is empty.
* **Human Controls:** Click a card in hand, then click target center pile (or click-to-play if only one valid spot exists).
* **AI Logic (Player 2):**
  * AI checks its hand every 1.2 to 2.5 seconds (adjustable speed based on selected tournament tier).
  * If a valid move exists, AI plays the card with a slight simulated reaction delay and draws a new card.

### C. Deadlock ("Stuck") State
* If neither player can make a move for 3 seconds, a "STUCK!" indicator highlights.
* Clicking the "Flip Backup" button (or auto-triggering after 4 seconds) flips 1 card from each backup pile onto the center active piles to resume gameplay.

### D. Win / Lose Condition
* The first player to clear all 5 cards from their hand AND all 15 cards from their draw pile wins immediately.
* Display a pixel-art banner: **"VICTORY! +X TOKENS"** or **"DEFEAT! BETTER LUCK NEXT TIME"**.

---

## 3. UI/UX & Retro Aesthetics

* **Color Palette:** Neon arcade colors (dark navy background `#0f172a`, arcade gold `#f59e0b`, vibrant crimson `#ef4444`, cyber green `#10b981`).
* **Visual Elements:**
  * Retro-style pixelated card borders with clear rank indicators.
  * Clear display of human hand (bottom), center play table (middle), and AI hand/deck counters (top).
  * Real-time match statistics overlay (Cards remaining, AI speed).

---

## Instructions for AI Generation
Generate the complete, runnable HTML file containing all HTML structure, embedded CSS, and fully functional JavaScript game logic. Ensure smooth gameplay, accurate rule enforcement, zero external missing dependencies, and a clean retro UI.