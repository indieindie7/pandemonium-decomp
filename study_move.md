# Pandemonium! (Saturn) - player movement study (s_move.c)

Scope: 31 functions, 0x06011534-0x06031d64. Research notes only. Many tuning constants are
literal-pool words (DAT_xxxx) whose values the decompile does not show; those are marked "pool".
The mover_*/path_* helpers are not in this file, so only their call order is known here.
About half of the file is NOT player code: enemy/NPC AI misnamed player_* (see section 6).

## 1. Data layout

Two structs are involved: the object `obj` (param_1) and the player record `pl` (puVar5 / param_2).
Evidence: player_UpdateMovement 0602b8a8, player_StartKnockback 0602b774, player_SwitchActiveCharacter 06031d64.

obj (param_1):
- +0x10/+0x14/+0x18 position x/y/z (player_UpdateGround 06018c74, player_CopyMatrixAndRestart 06011534)
- +0x1c flags word (many bit tests in 0602e4c8; 0x10 = facing/flip, swaps left/right bits in 0602b8a8)
- +0x58/+0x5c/+0x60 per-object velocity x/y/z (zeroed in player_ResetMotionState 0602a658; +0x5c zeroed on ground hit in player_CheckGroundContact 06018bdc)
- +0x64 current animation pointer; +0x7e anim frame (int16, -1 = restart); +0x6c timer; +0x70 sfx/state id
- +0x78 / +0x79 / +0x7a / +0x7b state bytes. Handler index = +0x79, or +0x7b if non-zero (player_RunCurrentStateHandler 06011594)

pl (player record, one per character slot):
- +0x00 flag word (bit 0x40 = use actor_MoveWithCollision; 0x02/0x20 cleared on rail piece entry, end of 0602e4c8)
- +0x04 word of bit flags (invuln/pickup bits), cleared and set around landing (0602e4c8)
- +0x08 byte hearts (capped 0x20 by pickup case 0); +0x09 byte second counter (case 2)
- +0x0a byte health-like counter; <=0 triggers the death path: obj+0x7b=1, timer +0x2c=0x1e (30), music stop (0602e4c8)
- +0x0c score; +0x10 high nibble power-up (values 0xb / 0xc, cases 7-9)
- +0x14 / +0x18 / +0x1c: horizontal speed along rail, vertical velocity, gravity state (1 up, 2 apex/fall, 0 ground, -1 airborne); zeroed together in 0602a658
- +0x2c frame counter, +0x2e short (15 = anim timer in 0602a6dc), +0x32 / +0x34 limits (jump hold limit +0x34, pool DAT_0602c22a)
- +0x36 jump buffer counter (counts up to 6; set 3/6 on a fresh press) (0602b8a8)
- +0x38 knockback timer (set to 10 by player_StartKnockback 0602b774)
- +0x3a short idle/blink timer (random idle pick 0xa/0xb/0xc, 0602e4c8 near its end)
- +0x48/+0x49/+0x4a three decay timers (per axis); +0x4c extra (0602b8a8)
- +0x68 pointer to the character class struct (the char object also stores its slot index at +0x68, player_InitSlotDefaults 06031a88)
- packed word (DAT_0602e2f2 offset): bits 16-19 current character index, bits 24-27 target/next (6 = none) (player_TrySwapLeadCharacter 0602e0e4, player_SetCharacterId 06031a54)
- character class struct: +0x44.. anim table indexed by state byte*4 (0602e4c8 end), +0x44 also used as speed scalar in 0602b8a8 (may be two different structs), +0x48 second scalar, +0x60 anim for state 7.

Rail data seen from here: obj->+0x20 is a rail piece record: three rows of 3 shorts at +0..+0x10 (each <<4 into 16.16 vectors, a basis) and ints at +0x14/+0x18/+0x1c (origin). 0602e4c8 does 3x3 dot products of the position against it (moving platform / piece-local transform). Rail structures are documented in notes_rail.md.

## 2. Per-frame flow

player_RunCurrentStateHandler 06011594 (generic object tick, also for non-player objects):
1. player_GetActiveObjectIfIdle 060300e4 gets the active object.
2. State handler from table by +0x79 / +0x7b.
3. Optional player_CopyMatrixAndRestart 06011534 when button flags 0x48 set.
4. player_ApplyImpulseDecay 0601ca90 if flag, player_ApplyPushDecay 0601ccac if a push counter is set.

Player tick (callers not in this file):
1. player_UpdateMovement 0602b8a8 (4166 bytes), in order:
   - pad_PollPlayerInput builds a move mask (1/2 left/right, swapped by flag 0x10; 0x80 action; 8 jump; 0x40 button)
   - jump buffer pl+0x36 and replay of the stored apex
   - coll_CheckTile; mover_UpdateRailNodes if a rail piece is attached
   - mover_SteerAlongRailSegment twice (speed scalar, x2, or >>2 when jump not held)
   - mover_ClampRailSpeed
   - mover_UpdateVerticalVelocity(obj, pl, &vy, &flag); vy clamped by pool DAT_0602c83a / c83c
   - path_EvalRailPositionSigned(obj, pl, speed, vy) gives the new rail position
   - then actor_MoveWithCollision (pl flag 0x40) or player_CheckGroundContact 06018bdc
   - landing: vy=0, gravity state=-1, state byte=6
   - path_ProjectOntoTrackFrames re-projects onto the track frames (moving platform)
   - player_ReverseRailDirection when the facing flag flips
   - jump and hit entries: player_StartJumpFromGround 0602b7e8, player_StartKnockback 0602b774
2. player_UpdateMainState 0602e4c8 (6022 bytes): pickups, timers, animation-state selection. Calls player_ApplyMotionAndRail, player_RefreshRailPlacement, player_UpdateRailPlacement, player_EnterRailPiece, then player_TrySwapLeadCharacter and anim_AdvanceFrame at the end.
3. player_TrySwapLeadCharacter 0602e0e4: on a pad trigger swaps lead and partner: player_SetCurrentRailPiece, path_EvalRailPosition (carry rail position to the new char), obj_RemoveFromActiveList, player_SwitchActiveCharacter 06031d64, obj_LinkIntoActiveList. Sets decay bytes +0x48..+0x4a and a global to 0xf.

Character management: player_SpawnCharacterObject 06031788 (loads a package entry, relocates anim tracks, appends records), player_SpawnAllCharacters 060318cc, player_InitSlotDefaults 06031a88, player_ResetCharacterSlots 06031990, player_SetCharacterId 06031a54, player_SwitchActiveCharacter 06031d64 (searches up to 13 (0xd) anims of the new character for the current one to keep the pose; index above 6 maps to 2 or 6 by a flag; calls obj_CopyState and obj_ReleaseSlotObject(pl,7); copies 15 words of state; resets the +0x3a timer).

## 3. Numbers

| Constant (as written) | Value | Controls | Where |
|---|---|---|---|
| state byte 9 | 9 | hit state id | player_StartKnockback 0602b774 |
| timer 10 | 10 frames | stun duration (pl+0x38) | 0602b774 |
| velocity halved (pool DAT_0602b7d8 mul) | x0.5 | velocity kept on hit; old values saved to globals | 0602b774 |
| 0xfffffff1 | -15 integer (not 16.16) | vertical value written on jump from ground | player_StartJumpFromGround 0602b7e8 |
| jump buffer 6 / 3 | 6 / 3 frames | press forgiveness (pl+0x36) | 0602b8a8 |
| speed scalar x2 / >>2 | x2, x0.25 | run speed multiplier, jump held / not held | 0602b8a8 |
| DAT_0602c83a / c83c | pool | vertical min/max clamp | 0602b8a8 |
| DAT_0602c22a | pool | max jump hold frames (pl+0x34) | 0602b8a8 |
| cap 0x2d | 45 frames | impulse decay length per axis | player_StartImpulseTimers 0601cc00 |
| (abs(v)>>1)+1 | 1..45 | timer init, bigger impulse lasts longer | 0601cc00 |
| DAT_0601cbf4[t] (ushort table) | pool | decay scale while t < 0x2d | player_ApplyImpulseDecay 0601ca90 |
| DAT_0601cbf8 / ((t+2)*0x1000) | pool | decay scale while t >= 0x2d | 0601ca90 |
| DAT_0601cd64[n-1] (ushort table) | pool | eased move toward push target | player_ApplyPushDecay 0601ccac |
| swap timers 0xf | 15 frames | post-swap grace | 0602e0e4 |
| powerup thresholds 0x25 / 0x4b | 37 / 75 frames | blink phase (&1, &3, &7) while invulnerable | player_GetPowerupLevel 06029eb8 |
| death timer 0x1e | 30 frames | pl+0x2c after health <=0 | 0602e4c8 |
| frame counter > 0xf clears pl+0x0a | 16 frames | short timer clear | 0602e4c8 |
| max hearts 0x20 | 32 | pickup case 0 | 0602e4c8 |
| idle pick: (rand&0x7f) <0x32 -> 0, <0x65 -> 10, else 11 | about 39% / 40% / 21% | idle animation variety | 0602e4c8 end |
| mode-1 anim timer 0xf | 15 frames | pl+0x2e | player_SetCharacterMode 0602a6dc |

Gravity, run speed and jump impulse are behind pool constants and the mover_* helpers (not in this file). The only concrete jump value visible here is the -15 kick.

## 4. How the rail works (as visible here)

- Movement is 1D: pl+0x14 speed along rail and pl+0x18 vertical go into path_EvalRailPositionSigned (0602b8a8), which yields the world position and facing. Collision runs on that result (actor_MoveWithCollision or player_CheckGroundContact 06018bdc), not on free 3D motion.
- Attached piece: mover_UpdateRailNodes updates the node pair; mover_SteerAlongRailSegment steers along a segment in two sub-steps per frame; mover_ClampRailSpeed limits speed per segment.
- On a moving piece, path_ProjectOntoTrackFrames re-projects the position onto track frames; a compare of two projected values decrements pl+0x2c (0602e4c8).
- Reversal: player_ReverseRailDirection when facing bit 0x10 flips (0602b8a8).
- Piece switch: player_SetCurrentRailPiece + path_EvalRailPosition carry position over a character swap (0602e0e4). player_EnterRailPiece (end of 0602e4c8) takes three values from the char class (+0x44/+0x48/+0x4c) at start/respawn and clears pl flags 0x22.
- Moving platforms: obj->+0x20 piece basis + origin used in dot products (0602e4c8).

## 5. State machines

pl state byte (written in 0602b8a8 and 0602e4c8), values seen: 0 idle, 1 idle after death check, 2, 3 (special/spin; becomes 7 on a flag), 4, 5, 6 landing, 7 jump/airborne (anim from class+0x60 unless char index 5), 8 forced when pl+0x0a != 0 and a flag, 9 knockback/hurt, 0xa/0xb/0xc idle variants, 0xd just switched character.
Transitions seen:
- jump press: player_StartJumpFromGround 0602b7e8: 9->2 if hurt, then state 7 via obj_ReleaseSlotObject(pl,7)
- hit: player_StartKnockback 0602b774: state 9, timer 10
- ground contact: state 6 (0602b8a8)
- anim end (+0x7e wraps) in 0602e4c8: 4/5 -> 1 or 0 by health; 3 or 9 -> 6; 7 holds the last frame unless char index is 5
- death: health <= 0 -> obj+0x7b=1, music stop, sound 6 (0602e4c8)
- swap: 0602e0e4 sets state 0xd, then by new char index (3,5,7,9) state 6/3/2
Character index (nibble 16-19): 1 and 5 have special cases (state 9 with index 1 plays a sound and effect 7; index 5 keeps the airborne anim). This fits Nikki double jump vs Fargus spin, but the mapping is not proven here.
Generic objects: state handler table by obj+0x79/+0x7b (06011594).

## 6. Suspected wrong names

0x06018c74 player_UpdateGround -> enemy_StateIdleChase  AI state: class struct, hit reaction, parent-matrix transform, no pad input
0x06019284 player_StateHandlerA -> enemy_StateHandlerA  spawns 5 debris, Atan2/sin, class sfx
0x06019850 player_StateHandlerB -> enemy_StateHandlerB  path_FindNearestFrame + random pick, enemy behaviour
0x06019c64 player_SetStateCallback -> enemy_SetStateCallback  installs handler B on an object
0x06019d68 player_StateHandlerC -> enemy_StateHandlerC  turn rate from class, obj_TurnYawToward
0x0601a2dc player_StateHandlerD -> enemy_StateHandlerD  same class pattern
0x0602e4c8 player_UpdateMainState -> player_UpdatePickupsAndAnimState  pickups, score, hearts, idle anim pick, rail placement
0x0602a7f0 player_UpdateScriptedAnim -> player_UpdateDeathRespawnSequence  fade, music restart, mode reset (uncertain)
Names that look right: player_SwitchActiveCharacter 06031d64, player_GetActiveObjectIfIdle 060300e4, player_ApplyImpulseDecay 0601ca90, player_ApplyPushDecay 0601ccac (generic, also used on non-players via 06011594).

## 7. Takeaways for a modern 2.5D rail platformer

1. Keep player motion 1D along the rail (speed + vertical) and let one function turn that into world position and facing; collide on the result (0602b8a8 order).
2. A jump input buffer of about 6 frames (pl+0x36) is cheap forgiveness; add coyote time the same way.
3. Knockback = state + short stun timer (10 frames) + keep half the velocity; per-axis decaying impulse timers (45 frame cap, length proportional to impulse size, table-eased falloff) give a smooth shove.
4. Drive decay and push with eased lookup tables (0601cbf4, 0601cd64) so tuning is data, not code.
5. A per-character class struct (speed scalar, anim table, special anims) lets Nikki/Fargus-style characters share one movement function.
6. Character swap should carry rail position (path_EvalRailPosition) and pose (find the matching anim in the new character) so it feels instant.
7. Speed is a scale on a per-character base (x2 / x0.25 by button), not an absolute.
8. Invulnerability blink by phase masks (&1, &3, &7 by remaining time) shows remaining time without a bar.
9. Moving platforms: store a piece-local transform (3x3 + origin) and reproject the player each frame instead of parenting.
10. Weighted random idle variants (about 40/40/20) are a cheap personality win.
