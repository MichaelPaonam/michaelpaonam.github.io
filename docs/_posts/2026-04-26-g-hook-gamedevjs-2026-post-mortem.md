---
layout: post
title: "G-Hook: Gamedevjs 2026 Post-Mortem"
date: 2026-04-26
categories: gamedev
---

I built a 2D top-down grappling hook time-trial game in 10 days for the Gamedevjs 2026 game jam. The engine was Defold (one of the jam sponsors), targeting HTML5. The core mechanic was inspired by Fanny from Mobile Legends — fire cables at anchor points, get pulled toward them, chain hooks for speed, race through checkpoints.

This is what went right, what went wrong, and what I'd do differently.

<iframe frameborder="0" src="https://itch.io/embed-upload/17312646?color=333333" allowfullscreen="" width="800" height="550"><a href="https://0xpaona.itch.io/g-hook">Play G-hook on itch.io</a></iframe>

## Timeline

The jam ran mid-to-late April. I started April 17 with a blank Defold project and submitted April 26. Rough breakdown:

- **Day 1 (Apr 17):** Project scaffolding, first working prototype of the hook mechanic, basic collision setup.
- **Days 2–3 (Apr 18–19):** Cable auto-release, mouse aiming, initial level geometry via Wavedash (Defold's tilemap tool).
- **Days 4–6 (Apr 22–23):** Level loader with collection proxies, two playable levels, sprite assets, camera panning.
- **Days 7–8 (Apr 24–25):** Level select screen, splash screen, third level shell, spike hazards.
- **Days 9–10 (Apr 25–26):** Camera clamping, backgrounds, crate walls, exit gate, speed gates, final polish, "thank you" screen.

## The core mechanic: pull, don't swing

My first attempt used a pendulum-swing approach — raycast to an anchor, create a rope constraint, let the player orbit around it. In zero-gravity top-down, this felt lifeless. There's no "down" to swing from, so the rope just made the player orbit aimlessly.

I scrapped it and went with a pull-toward system. When you fire a cable, the player is pulled directly toward the anchor point. Multiple cables (up to 3) pull toward the bisector of the last two. The cable auto-releases when you arrive close enough. This created the momentum-based gameplay I wanted — fire at a distant anchor, get pulled at speed, release at the right moment to coast, fire the next cable to redirect.

The implementation is pure Lua math on every frame — no physics joints:

```lua
local PULL_FORCE = 500
local DAMPING_HOOKED = 0.02  -- near-zero friction while pulled

-- Each frame: compute pull direction from active cables
-- Apply force toward bisector of last two anchor positions
-- Auto-release when within AUTO_RELEASE_DIST (120px)
```

This gave me full control over the feel without fighting Box2D's joint solver in a zero-gravity environment.

## What went right

**Claude Code as a co-pilot.** I used Claude Code from the very first commit. The initial prompt was literally "I'm joining a game jam, I need grappling hook mechanics similar to Fanny, Defold, HTML5, 2D top-down — how do I approach this?" Having an AI that could reason about Box2D constraints, Defold's message-passing architecture, and Lua idioms saved hours of docs-diving. It wrote the first draft of the player script, the render pipeline, and the screen-to-world coordinate conversion.

**Choosing Defold.** The engine's collection proxy system made level loading trivial — each level is a self-contained `.collection` file, loaded/unloaded by a central loader script. The HTML5 build pipeline (via `bob.jar`) just works. Zero-gravity was a single line in `game.project`:

```ini
[physics]
gravity_y = 0.0
gravity_x = 0.0
```

**Speed gates as a late mechanic.** On day 9, I added the speed aura — a particle effect that activates above 700 px/s — and made some checkpoints require that speed to pass. This added skill expression to what was otherwise just "hook and go." Players have to chain hooks correctly to build enough speed.

**Keeping scope tiny.** Three short levels. No enemies. No powerups. One mechanic, explored through level design.

## What went wrong

**Art.** I'm not an artist. The final game uses colored squares for most geometry and a spider sprite for the player. The cable rendering is literally Defold's `draw_line` debug primitive — single-pixel white lines. It works mechanically but looks like a prototype.

**Sound.** I added a background music track on day 8 and never got to sound effects. Hook fire, cable snap, speed aura activation, checkpoint hit — all silent. This hurt game feel significantly.

**Level design iteration time.** Defold's tile editor is functional but not fast for iterating on level layouts. I'd place anchors, build the game, test the route, realize the spacing was wrong, and repeat. A hot-reload workflow or an in-editor play mode would have saved time.

**Late camera work.** I didn't add camera clamping until day 9. For the first week, the camera would happily follow the player off into void space. This wasn't a huge problem during development but made the game feel unpolished whenever you overshot a section.

## Technical decisions I'd revisit

**Single-file player script.** All player logic — movement, cable management, chain tracking, speed aura, checkpoint handling, restart — lives in one ~250-line script. For a jam this was fine; for anything longer-lived I'd split cable management into its own module.

**Manual rope math instead of physics joints.** The right call for this game, but it means the cable has no elasticity, no visual sag, no physical interaction with obstacles. A hybrid approach — manual pull force but a visual catenary curve — would look better without changing the feel.

**No automated testing.** Defold doesn't have a built-in test runner for game scripts. Every change meant building and manually playing through. For a 10-day jam this is acceptable, but I caught at least two regressions late (camera breaking after restart, checkpoint ordering off by one).

## Tools and workflow

- **Defold Editor** for scene composition, tilemap painting, and builds.
- **Claude Code** for all script writing and debugging, running from the project root.
- **Git + GitHub** with feature branches for major additions (controls rework, level one, assets).
- **`build.sh`** wrapping `bob.jar` for HTML5 bundling — avoids the Defold Editor's bundle GUI.
- **Python HTTP server** for local testing (WASM requires HTTP, not `file://`).

## Numbers

- 10 days, ~40 commits across 5 PRs
- 3 playable levels
- ~800 lines of Lua (player, camera, level, checkpoint, HUD, loader, cable, utilities)
- 1 background music track, 0 sound effects
- Final HTML5 bundle: ~3MB

## What I'd do next

If I picked this back up: proper cable visuals (sprite-based with catenary math), sound effects for every action, 2–3 more levels that explore the multi-cable mechanic more deeply, and a global leaderboard via a simple REST API. The speed gate mechanic has room to grow — gates that require specific chain counts, or anchors that only activate at certain speeds.

The game works. It feels good to swing through a level at full speed, chaining hooks and barely clearing a speed gate. For 10 days and a first game jam, I'll take it.
