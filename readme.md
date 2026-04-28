# 🌌 Cosmic Merge — Devlog

### PLAY: https://vekien.github.io/cosmic-merge/

> A number-merging puzzle game built entirely through a single conversation with Claude.
> **Full conversation:** [claude.ai/share/8ebf4909-387f-47dd-8d1c-1b51d493f8d9](https://claude.ai/share/8ebf4909-387f-47dd-8d1c-1b51d493f8d9)

---

## The Premise

The starting prompt was simple: *"I play this game called number merge and it's basically like Tetris but has just blocks of 1 with numbers that double — can you make me a similar game in HTML/JS that works on mobile?"*

No design doc. No wireframes. Just a screenshot of the original game and a description of the rules. Everything you see in the final version was shaped through back-and-forth decisions during the conversation.

---

## How It Works

Tiles drop into a 5×8 grid. Matching numbers that touch — horizontally or vertically — merge and double. Gravity pulls everything down after each merge. Chain reactions earn a combo multiplier. The board fills up over time, and the game ends when every cell is full with no valid moves remaining.

---

## The Build, Decision by Decision

### 1. The Foundation
The first version was purely functional — a grid, tap to place, instant merges, flat colour tiles. No animations, no theme, no special mechanics. It worked but felt like a tech demo.

### 2. Drop Animation
The first big ask: *"Can you animate it falling down into place?"*

The approach was to create a "flying tile" element absolutely positioned over the grid. When you tap a column, the tile lifts from the queue, snaps to the top of that column, then falls to its resting position with a `cubic-bezier` bounce. Fall duration scales with distance — short drops are snappy, long drops feel physical. After landing, the flying tile hides and the grid re-renders with the tile in place.

### 3. Next Tile Queue
Originally the "next" tile sat to the right of the current tile with a label. This felt backwards — the natural read direction is left-to-right, so the upcoming tile should sit behind (to the left of) the active one. 

Both tiles are now absolutely positioned, aligned to the centre of column 3 via JavaScript after layout. The next tile is smaller (44px vs 60px) and at 55% opacity — clearly a preview rather than the active piece. No label needed; the size difference communicates the hierarchy.

### 4. Glass / 3D Tiles
The tiles were flat rectangles. The ask was *"can you make them look a bit 3D, a glass/gloss look?"*

The technique uses pure CSS with no images:
- `inset 0 1px 0 rgba(255,255,255,0.45)` — bright top edge faking a light source above
- `::before` pseudo-element — a white gradient over the top 48% of the tile, shaped with a rounded bottom using `border-radius` percentages, creates the gloss streak
- `::after` pseudo-element — a dark gradient at the base fakes the underside of a raised surface
- `text-shadow` — keeps the number legible against the gloss

The same treatment is applied to the queue tiles, the flying drop tile, and the animated merge flyers so everything is visually consistent.

### 5. Space Theme
*"Could the play area get a little spruced — maybe an astro/space theme?"*

This was a full overhaul:

- **Starfield** — a `<canvas>` behind everything. Stars are generated at random positions with individual twinkle speeds and directions. Occasional shooting stars are spawned as DOM elements with a CSS animation that scales them horizontally while fading out.
- **Nebula** — fixed `radial-gradient` blobs in purple, blue, and teal that slowly pulse via a CSS animation. Purely decorative depth.
- **Grid** — deep near-black background, subtle blue border glow, inset shadow, faint repeating column lane lines via `repeating-linear-gradient`, a thin light streak across the top edge.
- **UI panels** — frosted glass effect with `backdrop-filter: blur`, dark blue tints, and subtle border glows.
- **Column highlight** — changed from green to blue to fit the palette, with an inner light bloom on hover/touch.

### 6. Smart Merge Algorithm
This was the most technically interesting problem.

The original merge logic was a simple greedy scan: left-to-right, top-to-bottom, first pair wins. This missed chain opportunities. The classic example:

```
8
4 4   ← drop a 4 here →   8 4
4                          4 4
```

A naive scan merges the horizontal `4+4` first, leaving `8` on top and `8` on bottom — which then chains. That actually works in this case. But the ordering matters in subtler situations where the wrong first merge blocks a longer chain.

The fix: a **simulation-based smart merge**. Each time a merge wave runs, every candidate merge is simulated individually. For each candidate, the engine applies that merge, runs gravity, then counts how many future merges are reachable (recursively, up to 6 levels deep). The candidate with the highest chain-depth score is chosen first. Repeat until no more merges exist in that wave.

This means the engine always finds the ordering that leads to the longest cascade.

### 7. Stepped Chain Animations
Merges originally resolved instantly — the entire cascade happened in one frame. With the smart merge engine producing long chains, this looked like magic (tiles just vanishing and appearing). The fix was to animate each wave separately:

- Merge candidates spawn "flyer" div elements at their source positions
- Flyers slide to their destination with a CSS transition
- After the transition, flyers are removed, the grid re-renders with a `pop` animation on the merged tiles
- A short pause, then the next wave begins

This makes even a 4-wave cascade legible — you can follow each step.

### 8. Combo Multiplier
With chain animations slowed down enough to follow, the scoring needed to reward them. Wave 1 scores at face value, wave 2 at ×2, wave 3 at ×3, etc. A banner animates out of the merged tile position announcing *"3× COMBO!"*

### 9. Rainbow Tile
*"Can you add a rainbow block that is super random and will multiply any number it lands on?"*

The rainbow tile (★) is stored as a sentinel value (`-1`) in the grid so it can't accidentally be treated as a number. It uses a CSS animation to cycle through hues with `background-size: 300% 300%` and a spinning `conic-gradient` shimmer ring on the `::before` pseudo-element.

When it resolves, it picks a random adjacent numeric tile and multiplies it — originally ×2, ×4, or ×8 with equal probability. After a player got `16 × 8 = 128` unexpectedly (they expected ×2), the weights were adjusted: 70% chance ×2, 30% chance ×4. The popup now always shows the before and after value so there's no confusion.

Spawn rate: 1%.

### 10. Bomb & Hammer Tiles (Added and Removed)
Two more special tiles were built:

- **💣 Bomb** — destroyed a random adjacent tile, awarded its value as points
- **🔨 Hammer (Mjölnir)** — smashed every tile in the column below it, one by one with an animation, awarding each tile's value

Both were fully implemented with CSS animations (the bomb wobbled and pulsed orange, the hammer spun purple), resolve logic, score updates, and banners.

They were ultimately removed. The bomb felt arbitrary — destroying a tile you'd worked to build was frustrating, not fun. The hammer was dramatic but disrupted the merge flow in a way that felt disconnected from the core loop. Sometimes the right call is to cut.

### 11. Tile Spawn Weights
The original tile generator was board-aware: it looked at what values existed on the board and picked randomly from those (capped at 256). The idea was to keep drops relevant. In practice, late-game boards full of 64s and 256s meant those values kept spawning, making it nearly impossible to build up from smaller tiles.

After trying a slightly-weighted board-aware version (giving more tickets to 4, 8, 16), the final approach is fixed probabilities with no board awareness:

| Tile | Chance |
|------|--------|
| 2 | 25% |
| 4 | 25% |
| 8 | 22% |
| 16 | 12% |
| 32 | 9% |
| 64 | 4% |
| 128 | 4% |
| ★ Rainbow | 1% |

Predictable, fair, independent of game state.

### 12. Undo System
One undo charge, capped at 1. It only works if the last placement caused no merges — once tiles combine, the board state is too changed to cleanly reverse. The snapshot is taken before each placement and discarded if any merge triggers.

Recharges every 1,000 points. A canvas ring around the undo button shows charge progress. When it recharges, the button flashes three times.

### 13. Auto-Save
Every move serialises the full game state to `localStorage`: grid, score, best score, current tile, next tile, and the undo snapshot. On load, it attempts to restore the saved state. A lost game (board full) clears the save so it doesn't try to resume.

### 14. Game Over Logic
Originally ended when any tile reached row 0. This was wrong — a tile in row 0 might be merge-able with the incoming tile, giving a valid move.

The correct condition: game over when every cell is filled **and** no column can accept the current tile. A column can accept the current tile if it has any empty cell, or if its top tile matches the current tile (merge is possible).

As a visual aid, blocked columns get a subtle red tint so you can see at a glance where you can't play.

### 15. Grid Height Cap
On tall screens the grid stretched to fill the viewport which made the tiles too tall and the board feel awkward. A `max-height: 600px` on the grid element keeps it compact at a playable size.

---

## What Stayed the Same

- 5×8 grid size — felt right from the first version
- Drop-to-column mechanic — no auto-fall, just tap where you want
- Gravity after every action — tiles always fall to the lowest available position
- Score = sum of merged tile values, multiplied by combo wave

---

## Tech Notes

- Pure HTML/CSS/JS — no frameworks, no build step, no dependencies
- Single file — the entire game is one `.html` file you can open anywhere
- Canvas starfield — no library, raw 2D canvas API
- All animations — CSS transitions and keyframes, no JS animation loops
- Save/load — `localStorage` JSON serialisation
- Smart merge — recursive simulation, depth-capped at 6 levels
- Special tiles — negative sentinel integers (`-1` = rainbow) so they can't collide with real tile values

---

*Conversation: [claude.ai/share/8ebf4909-387f-47dd-8d1c-1b51d493f8d9](https://claude.ai/share/8ebf4909-387f-47dd-8d1c-1b51d493f8d9)*
