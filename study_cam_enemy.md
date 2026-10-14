# Study notes: camera / enemy AI / collision (s_cam_enemy.c, Pandemonium! Saturn)

Scope: 30 functions in s_cam_enemy.c. Up front: the two "cam_" functions are NOT camera code (no camera follow
logic exists in this slice; look in the rail/move slices). No boss_ or ai_ function is in this slice, so "boss"
has no direct evidence here. Most DAT_ constants are unresolved in the decompile; numbers below are literals visible
in code, unresolved ones are cited by DAT name only.

## 1. Camera
- 0x06004cf4 cam_ComputeScreenOffset: if byte +0x0e == 0 calls pkg_CalcGroupLoadSize + mem_Alloc, then stores
  (field +0x14 - buffer ptr) into a global. A package/load-relocation helper, not a camera function.
- 0x06012788 cam_SetParams: writes two globals and clears bit 0 of a flags global. No camera evidence.
- Enemy AI reads the player position through obj_UpdateHitReaction (first call in every enemy_Update* handler); it
  returns target pos A, pos B and a distance (0x060332d0, 0x0603342c, 0x06033d74).

## 2. Data layout (object struct, param_1)
| off | size | meaning | evidence |
|---|---|---|---|
| +0x08 | ptr | owner/parent object (class 4 skips its owner) | 0x0602388c |
| +0x0c | byte | record type (1, 2, 8 in 0x06022968; 1 in 0x0601729c) | 0x06022968 |
| +0x0e | ushort | flags; bit 0x40 = "player near/active" in every enemy handler; on list records it is a radius index (<<5) | 0x060332d0, 0x0603342c, 0x06033d74, 0x06033ff4, 0x060346c8, 0x0602388c |
| +0x0f | byte | subtype; trigger kinds 1, 6, 0x28..0x33 | 0x060235e8 |
| +0x10/14/18 | int x3 | world position (16.16) | 0x0601729c, 0x0601aff4 |
| +0x1c | uint | flag word: 0x10 = rail direction, 0x40 = alert/phase, other bits via unresolved DAT masks; a mask is cleared when a contact handler accepts | 0x0601aff4, 0x0603342c, 0x06023314 |
| +0x20 | ptr | parent transform: shorts 3x3 (<<4), translation ints +0x14/+0x18/+0x1c, platform delta +0x20/+0x24/+0x28 | 0x0601ab64, 0x06023314 |
| +0x24 | ptr | typedef: +0x14/+0x16 short frame counts fwd/back, +0x1e short steer accumulator, +0x25 type id, +0x28 hit damage, +0x2a ushort hit flags, +0x30 last-hit obj, +0x3c/3e/40 half extents (<<5), +0x40 shoot cooldown, +0x42 shoot frame, +0x44 ptr anim-set | 0x060332d0, 0x0601ab64, 0x06020ee4, 0x06022968 |
| +0x38/3c/40 | int x3 | collision centre | 0x060231ec, 0x0602388c |
| +0x44 | int | collision radius | 0x060231ec |
| +0x48 | int | foot/bottom offset used in the stomp check | 0x06023314 |
| +0x50..+0x56 | short x4 | orientation (copied to spawned projectile) | 0x060332d0 |
| +0x58/5c/60 | int x3 | velocity/delta per frame | 0x06017e48, 0x06023314 |
| +0x64 | ptr | current anim sequence; +6 = frame count | 0x06033d74 |
| +0x6c | int | countdown timer; in StateD packed (hi16 = rail frame) | 0x06033ff4 |
| +0x70 | int | spawn/child type (0x1e set by shooter; 0x17 special) | 0x060332d0, 0x0602388c |
| +0x74 | sbyte | child marker -2 (0xfe) / -1 (0xff): projectile classes | 0x0601ab64, 0x06023bbc |
| +0x79 | byte | state id (0x0c wander, 0x11 no-shoot, 0x1e special) | 0x06033fa8, 0x060332d0, 0x06023bbc |
| +0x7a | sbyte | class 1 player, 2 enemy, 3 projectile/hazard, 4 special; indexes handler tables | 0x06023314, 0x06023bbc |
| +0x7e | short | anim frame; -1 = restart sequence | all enemy handlers |
Tile/polygon data (0x0601729c coll_FetchTileRecord): record table at base + (packed>>16)*0x30, tile entries (packed&0xffff)*0x1c
from container +0x24. Static geometry records carry a polygon array with 0x30-byte stride and a count short (0x0602388c, 0x06023bbc).
Global world list holder: list +0x24 static, +0x1c dynamic, +0x2c / +0x30 hazard/trigger, +0x34 rail-piece-gated, +0x3c event (0x0602388c, 0x06023bbc).

## 3. Per-frame flow (collision)
1. coll_ResolveMove 0x06017e48: path_EvalRailPositionSigned (zero velocity, step along rail), fetch tile record (0x0601729c),
   coll_TestSegmentVsPolygons 0x06021f68 twice; negative result restores velocity; else Atan2 of motion vs DAT_06018034, scale or actor_MoveByDelta.
2. coll_TestObjectVsStaticGeometry 0x0602388c: gate = class 1, or class 3 with +0x74 == -2 and +0x70 != 0x17, or class 4 if its handler (table +0x10)
   agrees. Class 1 is first transformed by actor_TransformToParent. Walks list +0x24 then +0x1c: flag mask test, per-axis bounding prefilter
   (centre delta + radius <= 2*radius), coll_SphereOverlapTest, then coll_TestSegmentVsPolygons per polygon. Earliest hit stored,
   per-class handler table (PTR_DAT_06023bb8 + class*4) called; true return clears a flag bit in +0x1c and stores hit time.
3. coll_CheckObjectAgainstLists 0x06023bbc: class 2 (and class 3 with +0x74 == -2) test list +0x30 only; class 4 returns; others test +0x2c, then +0x34
   (only when collide_IsOnDifferentRailPiece accepts), and class 1 alone walks event list +0x3c: it calls trigger_PushEvent(id|DAT, 4) when the record's
   +0x12 flag is set, then coll_SweepHitboxSlots. Hits go through per-class handler tables then coll_ObjectVsTriggerContact.
4. coll_ObjectVsObjectContact 0x06023314: relative velocity (a - b, plus b's platform delta), coll_SphereOverlapTest broad phase, then up to 4 hitbox
   slots of a (0x10 stride, slot ends when word +0xc == -1) against up to 4 of b via coll_SweptHitboxContact; keeps earliest time and slot pair.
   Class 1 vs 2 with a flag bit and feet at/above the other's top at contact time writes a stomp value (DAT_0602359a) into a global hit record
   (+0 / +8 stomp flags, +4 time, +0xc slot bits &7), then calls the class handler.
5. coll_ObjectVsTriggerContact 0x060235e8: sphere test (radius = ushort +0x42 << 5); kinds: subtype 1, 6, or 0x28..0x33 with (+0x48>>12)&0xf == 0xc = solid-gated;
   == 0xd = second-flag-gated; per-slot gating from coll_GetTileSlopeFlags 0x060173dc; slot index stored in bits 12+ of the hit word.
6. coll_SweepHitboxSlots 0x060237b8: 4 slots vs one record, no gating, class handler on earliest time.
7. coll_ResolveHitFlags 0x06020ee4: contact -> hit-flag bits in typedef +0x2a (1, 2, 4, 8, 0x10, 0x20, 0x40/0x60, 0x22). Bit 4 stores attacker (+0x30) and damage
   (+0x28). Uses player_GetPowerupLevel, sfx_QueueSoundRequest. Type ids 0xc, 0xd, 0x15, 0x16, 0x28..0x33 look like pickups/triggers.
8. Tile layer: coll_CheckTile 0x060174ec projects velocity onto the tile plane (slide), Atan2 slope, FixedSqrt normalise, friction scale DAT_06017aee
   (unresolved), sets flag bit 0x40 "on surface", returns tile type ((>>16)&0xf, 5 special). coll_IsBlockedAtTile 0x06017424 blocks only when relative x,z are 0
   and slope/solid flags agree.

## 4. Collision primitives
- coll_SphereOverlapTest 0x060231ec: r = (extra + radius) * 4; if |pos - centre|^2 < r^2 returns true; otherwise computes |v|^2 and does a closest-approach
  quadratic (moving sphere vs sphere). All fixed-mul with the SR S-bit saturation boilerplate.
- coll_TestSegmentVsPolygons 0x06021f68: per polygon builds dot products (>>5, >>4), up to 4 refinement iterations with Atan2, SinTableLookup, AngleComplement;
  returns hit distance or -1.
- coll_SweptHitboxContact 0x06022968: type 2 builds box half extents from typedef +0x3c/+0x3e/+0x40 << 5; type 8 uses +0x28/+0x2c/+0x30 and +0x18 (sphere);
  box handled by a slab test over 6 faces (3 axes x 2 sides), entry time via math_DivideSigned32, tracks max entry/min exit, result face index 0..5
  chooses the contact normal. Output time in *param_4 (fraction of the frame move); returns 0 on miss. Subtype 9 on type 2 has special rules.
- Contact time must be <= the object's stored best time (+1 slack) to count (0x06023314, 0x060237b8).

## 5. Enemy AI state machines (shared skeleton)
Skeleton: obj_UpdateHitReaction -> if flag +0x0e & 0x40 run behaviour else idle anim -> anim_SelectChildSequence. Anim set (typedef +0x44): +0x04 notice range,
+0x08 second range, +0x44 turn rate, +0x48 idle, +0x50 turn/alert, +0x54 attack/walk. Switching = write +0x64, set +0x7e = -1, set flag bit.
- enemy_UpdateShooterAI 0x060332d0: obj_FaceMovementDirection with turn rate; countdown +0x6c; at 0 with facing OK, flag clear, +0x79 != 0x11: +0x70 = 0x1e,
  spawn child via function pointer, child gets zero velocity, copied orientation, +0x74 = 0xfe; timer reloads from typedef +0x40.
- enemy_UpdateChaseAI 0x0603342c: hit flag 8 sets timer 0x1e (30). Dist < animset+4 -> attack/walk; dist > animset+8 -> alert. Angle test: Atan2(target) minus
  facing beyond +-0x3f (~5.5 degrees of a 16-bit circle) -> turn. Turn-around = mirror frame index (count - frame) and XOR flag 0x10.
- enemy_UpdateWander 0x06033d74: random timer Random mod 0x4b (75) + 0x1e = 30..104 frames toggles walk (+0x54) / idle (+0x48); at anim end picks by Random&1;
  turns when obj_CheckFacingTowardTarget < 0. enemy_EnterWander 0x06033fa8: +0x79 = 0x0c, timer = Random mod 75 + 0xf (15..89), installs handler pointer.
- enemy_UpdateStateD 0x06033ff4 (hover/flyer): picks a random target near path_FindNearestFrame (frame -3 + rand, y/z jitter), packed into +0x6c; steers by the accumulator
  typedef +0x1e (clamped, damped when y error 0); reverses at rail ends (frame 0 or count-1 toggles 0x10); hit flag 0x80 blocks reversal; writes +0x54 from a global angle.
- enemy_UpdateStateE 0x060346c8: same accumulator, ramped by DAT increments, zeroed when error small; flag 0x40 toggles on activate, clears accumulator on deactivate.
- enemy_UpdatePatrolAnimState 0x0601a764: rail patrol idle/turn/attack picks; flips at rail ends via path_FindNearestFrame.
- enemy_AdvanceRailKeyframe 0x0601aff4: timer 0 -> -1 steps to the next keyframe (0x28-byte records: pos xyz at +0, short +0xc frame index, negative = skip); direction bit 0x10
  forward/back, wraps by count byte +0x10; recomputes world pos and delta, applies parent matrix; timer = 0xf (15).
- enemy_SpawnProjectileOnRail 0x0601ab64: fires when frame+1 == typedef +0x42, flag 0x40, +0x79 != 0x11, dist <= DAT_0601aca4, facing OK; may flip direction if the target is
  behind along the rail (path_SampleFramePosition frame +-1, squared distances). Spawns type 0x1c if type id (+0x25) == 8 else 0x1a; +0x74 = 0xfe, +0x75 = 0xfd; timer from +0x40.
- enemy_UpdateIdleBlink 0x0601cf54: idle/blink, timer Random mod 0x3c + 0x5a = 90..149; blink 0x3c (60).

## 6. Numbers (literals visible)
- Timers: wander 30..104, enter-wander 15..89, idle-blink 90..149, blink 60, chase alert 30, keyframe step 15.
- Facing tolerance 0x3f. Hitbox half-extent scale <<5. Radius *4 before fixed mul. 4 hitbox slots of 0x10 bytes.
- Not resolved (DAT only): gravity, friction DAT_06017aee, stomp offset DAT_06023598, shoot range DAT_0601aca4, chase y-band DAT_060336a2, steer clamps DAT_060343b4/b8.

## 7. Boss
No boss-specific code here. Bosses probably reuse these skeletons with bigger typedef ranges; check other slices.

## 8. Suspected wrong names
- 0x06004cf4 cam_ComputeScreenOffset -> pkg_RelocateGroupBase  computes load offset from package size + alloc
- 0x06012788 cam_SetParams -> sys_SetGlobalsClearFlag  two global writes and a bit clear, no camera use
- 0x0601b3f0 enemy_SelectSequenceFromTarget and 0x0601d8a8 enemy_UpdateHitReactionAnim -> one name, enemy_UpdateIdleSampleTarget  identical bodies
- 0x06033ff4 enemy_UpdateStateD -> enemy_UpdateHoverToRailTarget  random rail-frame target + steer accumulator
- 0x060346c8 enemy_UpdateStateE -> enemy_UpdateHoverBob  steer accumulator only, no rail target
- 0x06022968 coll_SweptHitboxContact  name fine, comment "sliding" wrong: it returns time + face index, does not slide

## 9. Takeaways for a modern 2.5D rail platformer
- Contact is time-of-impact: every test returns a frame fraction plus a face index; keep the earliest across slots (swept AABB/sphere).
- Up to 4 small hitbox slots per object instead of one big box, so attack and body boxes can differ per animation frame.
- Per-class handler tables (player, enemy, projectile, special): collision only finds the pair, class code decides the outcome.
- Stomp rule: flag set when the player's feet are above the enemy top at contact time; separates stomp from damage cleanly.
- Cheap per-axis centre-vs-2*radius broad phase before any math; separate lists for static, dynamic, hazard, trigger, event.
- Triggers only test on the same rail piece (collide_IsOnDifferentRailPiece): keeps branching rails from interfering.
- Enemy AI is one tiny skeleton: range gate, anim-set table, random timers, facing check; reversing = mirror the rail frame and flip a bit.
- Enemies ride the same rail data as the player (keyframes, parent transform, direction bit), so no separate nav system.
- Data-drive it: ranges, cooldowns, shoot frame, turn rate live in a per-type definition; random timers are short (0.5-3 s at 30 fps).
