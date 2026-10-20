# Polish review (v0.25.2, 2026-10-20)

A deep review before the content push (more heroes and maps). Art and audio quality are out of scope by the owner's choice; layout, readability, feel, flow, fairness, networking and code that will get in the way of new content are in scope.

How it was done: full unit test run, the online smoke test, a windowed solo autopilot run through all three stages, a title-menu run that captured every info page, and a 3-player co-op session with a scripted knockout (all with screenshots), plus a code review of the whole of `src/` split into four areas. Every finding below was confirmed in the code; a few are marked *unverified* where only the geometry or timing is inferred.

## Verdict

The game is in better shape than most hobby projects at this stage: the architecture is disciplined (fixed tick order, pure deterministic simulation, every RPC gated on ready peers, every client value validated), the menus are fully gamepad-navigable, the data (cards, relics, quests, Embers, evolutions, the Game Guide) matches the design doc almost everywhere, and tests plus smoke test pass. What is missing is the layer of *feel* and *fairness* polish that separates "works" from "finished": enemy attacks have no wind-up, enemies and bosses can pop in on screen, co-op clients miss death effects and get teleported at stage starts, one AFK friend can lock a level-up forever, and a new player gets no onboarding at all. None of these are big rewrites. Two structural changes (data-driven stage scaling and a hero picker built from `Characters.ALL`) should land before the first new hero or map.

## The 12 things to fix first

Ordered by player impact per hour of work.

1. **Telegraph attacks.** Nothing warns before a volley: `BossBrain.tick` fires the step the instant its wait ends ([boss_brain.gd:33](../src/enemies/boss_brain.gd)) and the 0.35 s attack pose starts *at* fire time ([enemy.gd:92](../src/enemies/enemy.gd)). Cheapest fix that reuses what exists: add a ~0.35 s wind-up to every enemy bullet's delay in `_spawn_enemy_pattern` (negative-age bullets are already invisible and harmless), play the pose and a flash at the start of the wind-up, and let the Mire Hag's wall unfurl from her (delay growing with offset) so it can't spawn on a player at her flank. Clients already fast-forward by RTT, so fairness holds.
2. **Co-op clients lose death effects and see enemies streak across the map.** The pool's free list is LIFO ([enemy_manager.gd:82](../src/enemies/enemy_manager.gd), [:497](../src/enemies/enemy_manager.gd)), so a dead enemy's slot is the first one reused; snapshots go every 4 ticks, and a client only re-activates a slot when the type changes ([enemy_manager.gd:427](../src/enemies/enemy_manager.gd)), so the same-type case lerps the old enemy to the new spawn point with no snap distance and `enemy_vanished` never fires. The host never sees this. Fix: reuse the oldest free slot (FIFO), add a snap distance to `Enemy._process`, and ideally a generation byte per slot in the snapshot.
3. **Clients are not moved to the stage-start spot until play resumes.** `respawn()` runs on the host only ([arena.gd:994](../src/arena/arena.gd)); the client only moves through `reconcile`, which returns zero when the ack is not in history ([client_predictor.gd:26](../src/player/client_predictor.gd)), which is always the case after a shop. The "3, 2, 1" ring and the YOU arrow are drawn at the old spot and the hero teleports at GO, every stage 2 and 3 start, for every client. Fix: in `apply_server_state`, when the ack is unknown or the phase is not PLAYING, set `state.position` to the server position.
4. **An abandoned ritual spawns enemies for the rest of the stage, boss fight included.** `_tick_ritual` only uses `inside` for progress; `wave_requested` fires every 1.2 s forever once ACTIVE ([map_events.gd:308](../src/arena/map_events.gd)), and the wave bypasses the boss-fight cap ([arena_rewards.gd:191](../src/arena/arena_rewards.gd)). Fix: stop waves when nobody is inside and once the boss is out.
5. **Enemies spawn on screen near walls and shooters fire from off screen.** Spawns are placed by distance from a player ([arena.gd:684](../src/arena/arena.gd)) but the camera is clamped to the arena bounds ([player.gd:144](../src/player/player.gd)), so a player kiting along a wall sees pop-in mid-screen. Cultists, Spitters, Paladins and champions have a `fire_range` of 220 to 260 px, more than the 180 px half-height of the view. Fix: test spawn candidates against each player's actual clamped view rect; cap non-boss `fire_range` near 170 or require the shooter to be in the target's view.
6. **Boss entrance has no stagecraft and spams sounds.** The boss lands 220 px from the centroid with no check against player positions, escorts pop in fully visible, and the horde "crumble" releases every enemy through `enemy_vanished`, playing a death sound each ([arena.gd:1032](../src/arena/arena.gd), [:1307](../src/arena/arena.gd)) through 16 voices in one frame. Fix: pick the direction farthest from players, a 0.5 to 1 s rise-in for the boss and escorts, and a distinct sweep visual with one sound.
7. **Shop: buying a relic bounces gamepad focus onto Ready.** After a buy the bought card is disabled, so `refresh()` moves focus to Ready ([shop_panel.gd:161](../src/ui/shop_panel.gd)); the next A press ends the player's shopping. Fix: move focus to the next enabled card or Reroll, Ready only as the last resort.
8. **One AFK player blocks everyone forever.** Level-up and shop countdowns start only after the first pick or Ready ([level_up_session.gd:353](../src/progression/level_up_session.gd), [shop_session.gd:408](../src/progression/shop_session.gd)). Fix: a longer idle countdown (60 to 90 s) that starts when the round opens.
9. **Host loss is silent.** No ENet timeout is set ([net.gd:93](../src/autoload/net.gd)) and nothing watches the time since the last snapshot, so a host crash freezes enemies while the client's hero keeps walking for 5 to 30 s before "Disconnected from the host." Fix: a "Lost connection to the host..." line after ~2 s without a snapshot, and a ~10 s ENet timeout.
10. **Pause menu online and Leave Game.** The title says "Paused" online, where the game does not pause ([pause_menu.tscn:34](../src/ui/pause_menu.tscn)); Leave Game has no confirmation, and the host's Leave disconnects everyone ([main.gd:225](../src/main/main.gd)). The lobby has no Leave button at all, so a client who joined the wrong host has to find Esc -> "Paused" -> Leave Game. Fix: "Menu" title with "Your hero stands still, others keep playing" online; a confirm for leaving mid-run (with "your friends will be disconnected" for the host); a Leave button in the lobby.
11. **No onboarding.** A run starts straight into PLAYING; the stage name never shows during play, the 4:00 timer is never explained, and the only sentence saying "you only move, weapons fire by themselves" is inside the hero Watch popup ([hero_demo_popup.gd:76](../src/lobby/hero_demo_popup.gd)). Fix: a 3 s stage-start banner ("Stage 1: The Crypt. Survive 4:00, then slay the Bone Warden"), a controls line in the lobby status, and a short "How to play" tab first in the Game Guide.
12. **Enemy pattern events share the reliable channel with everything else.** Patterns, cards, gem batches every tick and a reliable RPC per scythe / lightning / spear / Seeking Bolt volley all ride channel 0 ([arena.gd:1511](../src/arena/arena.gd), [weapon_system.gd:600](../src/combat/weapon_system.gd)). ENet's reliable-ordered delivery blocks the whole channel on one lost packet, and the 0.4 s fast-forward clamp cannot absorb a retransmit stall at internet latency, so volleys arrive late and in the wrong place. Fix: patterns on their own channel (or unreliable with 3-tick redundancy, deduped by seed and fire time) and cosmetic weapon events downgraded to unreliable.

## Also worth doing, by area

### Fairness and feel

- Run-over beats boss-kill when both happen on the same tick: `_update_phase` checks "everyone down" before `_boss_defeated` ([arena.gd:832](../src/arena/arena.gd)). Swap the order.
- Bullet damage and the attack pose can come from the wrong enemy: the shooter is found by `find_hit(origin, 6.0)` ([arena.gd:1355](../src/arena/arena.gd)), so a boss with escorts on top of it deals the escort's bullet damage. Send the shooter's pool index in the pattern event.
- The escaping Grave Robber visibly dies (corpse, burst, death sound) while the toast says it got away ([map_events.gd:339](../src/arena/map_events.gd)). Add a "removed" reason that skips the death effect.
- Hex sigils and Bone Effigy markers keep animating through level-up and shop pauses (`_process` in `ability_marker.gd` and `bomb_blast.gd`) while the effect itself is frozen, so after a long pick the marker is gone and the enemies are still bound.
- Frozen and hexed enemies still get separation pushes ([enemy_manager.gd:176](../src/enemies/enemy_manager.gd)) and drift apart at up to 40 px/s.
- Piercing bolts can hit the same enemy twice when two overlap; only the last hit is remembered ([projectile_manager.gd:183](../src/combat/projectile_manager.gd)).
- Chain Lightning's 50% boss focus is overridden by the cone scoring in `_lightning_target` ([weapon_system.gd:547](../src/combat/weapon_system.gd)): a nearer add in the cone always wins. Carry the "boss picked" flag through.
- Clients remove enemy bullets on contact even during the host's 0.5 s safety, which reads as hits that did not count ([projectile_manager.gd:238](../src/combat/projectile_manager.gd)). Predict `bullet_safe_left` locally.
- Only bullet weapons give a hit spark and sound; scythes, skulls, aura, flames and spears only flash the enemy. A tiny spark on scythe and skull contacts would make Mortimer's hits read better.
- Pack surges have no cue beyond minimap dots. A short edge flash on the surge side would sell the "cluster from one direction" idea.
- Stale inputs are replayed after every pause: up to 6 queued client inputs from before the level-up run on the first PLAYING tick ([player.gd:512](../src/player/player.gd)). Clear queues and predictor history on every phase change.
- Inputs are one unreliable packet per tick with no redundancy ([player.gd:535](../src/player/player.gd)); each lost packet is a host freeze tick and a client correction. Send the last 2 or 3 inputs per packet.

### Co-op sessions

- Level-up cards open by reliable RPC but the phase that stops your hero arrives by unreliable snapshot ([level_up_controller.gd:276](../src/progression/level_up_controller.gd)); under loss the hero walks into the horde under the cards. Make the RPC set the phase.
- A player who left mid-run still competes for MVP and counts in the team score ([arena.gd:590](../src/arena/arena.gd), [summary_player_cards.gd:39](../src/ui/summary_player_cards.gd)); nobody gets the crown.
- A friend who joins while the run summary is up gets a frozen blank arena: `_final_stats` is only sent once ([arena.gd:869](../src/arena/arena.gd)); resend it in `_notify_ready`.
- No "X joined / X left" toast in the arena; teammates just appear or vanish.
- Duplicate names are not suffixed ("Waiting for Tom, Tom"). In the test session all three players were "Big T" and the revive circle drew the two names on top of each other.
- Teammate name labels sit on the hero sprites in a crowd (seen in the co-op screenshots); nudge them up or fade them when heroes overlap.
- A mid-run joiner bakes the floor twice, the second time inside the first snapshot handler ([arena.gd:1460](../src/arena/arena.gd)); send the stage with the config first.
- Server full and wrong address both time out into "Could not connect to the host." with no Cancel on the Connecting screen ([main_menu.gd:434](../src/ui/main_menu.gd)).
- Host upload is ~40 KB/s per client during a full horde (300 enemies × 9 B × 15 Hz). Fine on cable; send far-off enemies at 5 Hz if friends on weak uplinks report loss.

### HUD and screens (seen in screenshots)

- Toasts collide with the level-up header: "Ritual complete! ..." sat right on "Level 2! Choose one" in both the solo and co-op runs. Hide or move toasts while a panel is open.
- At boss arrival three things stack at the top: the boss bar, a toast and the "rises!" banner. Suppress toasts during the banner.
- The title screen's version label shows through the bottom edge of every full-screen page (Game Guide, Records, Ember Shrine, Playtest Checklist); hide it while a `ListScreen` is open, as the menu already does for its panel.
- The quest line (9 px, no clip, up to ~70 characters for "Champion Hunter (...)") runs into the toast area ([hud.gd:94](../src/ui/hud.gd)); clip it or drop the parenthesis.
- "players 1" in the top-right status is noise in solo ([arena_hud_feed.gd:68](../src/arena/arena_hud_feed.gd)); "Press Start when everyone is ready." shows for a solo lobby.
- The timer label turns into the word BOSS for the whole fight; players lose the clock. Show the fight time small under the boss bar.
- The host's invite line sits in the play area all run. Show it for the first 10 s of a stage and in the pause menu; F1 still copies.
- "Return to character select" / "Waiting for the host to return to character select..." / "Pick a character and press Ready." are the only places that say "character"; everything else says hero. Use "Return to lobby".
- The run summary hint says "Esc back" only (gamepad: B), and on the Overview page B falls through to the pause menu. Clients waiting at the summary are not told they can leave.
- Focused and selected toggle buttons differ only by a slightly lighter fill (`game_theme.tres`); in the hero picker and the difficulty presets this is hard to tell apart on a gamepad. Give focus a thicker or differently coloured border.
- Settings: sliders show no value, no borderless window option, no mute-in-background, no view of the controls, no reset; changes are only saved on Back. Exit Game quits instantly and bypasses `Main.quit_game()`.
- The feedback screen focuses the TextEdit, which traps a gamepad (no way to leave the box). Focus the Kind button when the last input was a pad.
- Gamepad Start inside Difficulty / Custom Game / Game Guide opens the pause menu on top (Esc goes back, Start does not).
- Information only in tooltips never reaches gamepad players ("Spend them at the Ember Shrine", "score x1.25" explanation).
- Button labels mix Title Case ("Play Solo", "Start Run") and sentence case ("Send feedback", "Refund all"); Back sits in a different place on each screen.

### Progression data (small text and number nits)

- Three evolutions shrink in one dimension versus their maxed base: Death Spiral radius 9 vs Skulls L5 radius 11; Inferno Path 22 vs Hellfire L5 24; Sanctuary is only +37% DPS over Holy Aura L5, the weakest evolution. Set evolved radius to at least the base's L5 and bump Sanctuary.
- "N% faster" cooldown stats are interval cuts (Quick Hands 10% -> 11.1% more attacks, Cracked Hourglass 20% -> 25%), and Arcane Focus compounds (L5 = +101%, not +75%). Cards show real numbers, so only the Guide and DESIGN wording is off.
- `--character` bypasses locks and `HeroUnlocks.record_win` writes `next + 1` unconditionally ([hero_unlocks.gd:52](../src/progression/hero_unlocks.gd)): winning as Morwen with only Kael unlocked skips Mortimer. Dev-only, but DESIGN says it unlocks nothing.
- A "no boss" custom full run counts as a win for unlocks and Embers; a Hard single stage pays ~32 Embers for 5 minutes versus 40 for a 20-minute full run. Decide whether that is intended.
- The stage-3 boss still grants evolutions the moment before VICTORY; harmless, but the Guide's "the next boss your team beats" oversells it.
- Sharpened Edge's card preview only shows main damage although auto weapons gain too.
- Non-Kael heroes see only 7 relics (Bone Charm and Hunter's Eye are Bolt Gun only), so a stage-2 shop can show 2 or 3 cards in a 4-card layout. Add weapon-flavoured relics for the other kinds or a neutral filler.

## Before adding heroes and maps

These are the hard-coded assumptions a 5th hero or a 4th stage would hit. The first two are the ones to do first.

1. **Stage scaling is seven 3-entry tables that clamp.** `ENEMY_HP_BY_DEPTH`, `BOSS_HP_BY_DEPTH`, `SPAWN_RATE_BY_DEPTH`, `DAMAGE_BY_DEPTH`, `AUTO_DAMAGE_BY_DEPTH`, `COIN_DROP_BY_DEPTH`, `RAMP_HEAD_START_BY_DEPTH` ([arena.gd:32-47](../src/arena/arena.gd)) are read through `by_depth()` which clamps the index, so a 4th stage is silently exactly as hard as the 3rd. Also `RunConfig.LIMITS["stage"]` caps at 3 (Custom Game would show the 4th map's name and play stage 3), `--start-stage` clamps to 3, and the Game Guide hard-codes stages 2 and 3. Move the scaling onto `StageDef` (or one scaling resource sized by `Stages.ALL`), compute the clamps from `Stages.ALL.size()`, and add a test that every table matches the stage count.
2. **The hero picker is four hand-placed cards.** `_cards = [%Card0 .. %Card3]` ([lobby.gd:46](../src/lobby/lobby.gd)) and the nodes in `lobby.tscn`; a 5th hero gets no card, no lock text and no Watch button, and the Watch row would index out of range. Build the cards from `Characters.ALL` (one card scene, a wrapping grid for 5+).
3. **Two parallel arrays indexed by `MainWeapon`** crash on a new weapon kind: `Upgrades.COUNT_NAMES` ([upgrades.gd:13](../src/progression/upgrades.gd)) and `Arena.MAIN_WEAPON_SOUNDS` ([arena.gd:27](../src/arena/arena.gd)). Put the count name and attack sound on `CharacterStats`.
4. **`Characters.base_damage()` keys the damage share by weapon kind, not hero** ([characters.gd:24](../src/player/characters.gd)): a second Scythe hero with different damage gets Mortimer's Sharpened Edge numbers. Store `starting_damage` on the player's duplicated stats in `Player.setup`.
5. **Music must be registered in code**: `Music.RENDER_ORDER` ([music.gd:20](../src/autoload/music.gd)) is the only list of tracks; a stage whose `music` is not in it is silent forever. Build the list from `StageDef.music`.
6. **A new weapon kind touches seven files** (`WeaponSystem.fire_main`, `AutoAim.pick` / `reach_of`, `HeroDemo._attack`, `Compendium.main_weapon_text`, the upgrade preview labels, the arena shot sound). A hero that reuses an existing kind is pure data, which is the cheap path for the first new heroes. Longer term a `MainWeaponDef` resource per kind (sound, aim rule, reach, fire callable, demo script, guide text) would make kinds data too.
7. Smaller: `StageDef.PropStyle` is an enum with three branches in `floor_baker.gd` (a new look is code, reusing one is data); `SpawnDirector` falls back to Crypt enemies when a stage's table is empty; `RunStats.fell_to` is a boss display name, so two stages with the same boss mark the wrong trophy; tests pin `Characters.ALL.size() == 4` and the auto weapon count; the `--character` docs say 0-3.

## Process

- **One unit test is flaky because tests read your real save files.** `test_hex_snare_roots_enemies_and_they_take_extra_damage` failed on a clean checkout with 12 instead of 6: `Net._own_boosts()` loads `user://embers.cfg` unless autopilot is on ([net.gd:61](../src/autoload/net.gd)), your save has Keen Eye rank 1, so the test's Kael has a 2% crit chance and the hex damage roll crit. Give the test runner a flag (or an environment variable read by `Net`, `Settings`, `Embers`, `HeroUnlocks`, `RunRecords`) that ignores `user://`, or have `run_tests.ps1` point Godot at a scratch user dir.
- Online smoke test: passed. Solo run: victory, no script errors. Co-op session: connected, knockout and revive worked, no errors or MTU warnings.
- `--local-only` leaves "Invite: finding your public address..." on the host HUD for the whole run (harmless in tests, but the text never resolves).

## What checked out (no need to re-review)

Every menu sets focus on open and Esc / B goes back; list screens follow focus; level-up and shop countdowns show on every peer and disconnects shrink the waiting set; names are clipped and sanitised; typing never leaks into movement; no unreliable RPC can exceed the MTU; no empty packed array is sent alone; all `any_peer` RPCs validate their input; host leaving returns clients cleanly to the title with cursor, music and input restored; aim is rolled once on the shooter and never re-rolled; all ~30 DESIGN.md numbers checked match the code; every evolution is reachable; the card system never runs dry early; Embers math is exact.
