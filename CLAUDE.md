# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the games

No build step. Open any HTML file directly in a browser:

```bash
open shooter.html
open tictactoe.html
```

## Repository conventions

- **Zero dependencies, zero build tools.** Everything is plain HTML + CSS + JS in a single self-contained file per game.
- **No external assets.** All sprites are drawn procedurally with the Canvas 2D API (colored rectangles simulating pixel art).
- **Git workflow:** after every meaningful piece of work (new feature, bug fix, refactor), stage the relevant files, write a clean descriptive commit message, and push to GitHub immediately. Never leave work uncommitted — every session should end with a clean `git status`. The `gh` CLI binary lives at `/opt/homebrew/bin/gh` — the shell alias `gh` is broken (`history|grep`), so always use the full path.
- **Commit message format:** short imperative subject line (e.g. `Fix particle arc radius crash on enemy death`), followed by a blank line and a brief body if the change needs explanation.

## shooter.html architecture

All game code lives in one `<script>` block, organized in this order:

1. **Config constants** — canvas size (`W=800, H=600`), speeds, cooldowns
2. **Input manager** — `keys{}` for keyboard, `mouse{}` for position + click state; reset `mouse.clicked` each frame in the game loop
3. **Sprite functions** — `drawPlayer`, `drawGrunt`, `drawCharger`, `drawShooter`, `drawBoss`, `drawBullet`, `drawHeart`; all use `ctx.save()`/`ctx.restore()` and draw relative to a translated origin
4. **Particle system** — global `particles[]` array; `spawnParticles`, `spawnExplosion`; **critical:** `updateParticles` decrements `life` first, then filters (never filter before decrement or `ctx.arc` gets called with radius 0)
5. **Entity classes** — `Player`, `Bullet`, `Enemy`; `Enemy.update()` returns a `Bullet`, an array of `Bullet`s (boss burst), or `null`
6. **LevelManager** — holds `LEVELS[]` wave definitions; `isDone(enemies)` returns true when both `spawnQueue` and the live enemy array are empty; `nextWave()` returns `'wave'` or `'level_done'`
7. **Game class** — state machine (`menu → playing → level_complete → playing` / `game_over`); `loop()` calls `update()` then `draw()` each frame via `requestAnimationFrame`

### State machine
```
menu ──Enter──► playing ──all enemies dead, more waves──► playing (next wave)
                        ──last wave cleared──► level_complete ──3s──► playing (next level)
                        ──last level cleared──► victory ──5s──► menu
                        ──player hp=0──► game_over ──Enter──► menu
```

### Enemy types & level definitions
| Type | HP | Behavior |
|------|----|----------|
| grunt | 1 | walks straight at player |
| charger | 2 | accelerates toward player |
| shooter | 2 | maintains distance, fires back |
| boss | 20 | orbits player, 4-way bullet burst |

`LEVELS` array (index 0–4) defines waves as `{ grunt: N, charger: N, shooter: N, boss: N }` objects. Level 5 (index 4) is the boss fight.

### Canvas state rules
- Always call `ctx.save()`/`ctx.restore()` in draw functions that translate/rotate
- Reset `ctx.globalAlpha = 1`, `ctx.shadowBlur = 0`, `ctx.textAlign = 'left'` after any HUD or overlay drawing
- `ctx.arc` radius must be > 0; use `Math.max(0.5, radius)` when the radius is computed from a decaying value
