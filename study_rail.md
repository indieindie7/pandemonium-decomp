# Pandemonium! (Saturn) - rail / path system study (s_rail.c, 29 functions)

Scope: path_ functions, rail pieces, branches, world position and facing from the path. Research only.
Reading caveat: the decompile is full of inlined 16.16 multiply noise and most tuning constants are
DAT_xxxx pool words that the decompile does not resolve, so the "Numbers" section is thin and says so.
Field names below are inferred from use; "evidence" is the function where the access shows up.

## 1. Data layout

### Path object (the data a rail piece points to; reached as piece+0x24)
- +0x10 byte: number of keyframe records minus 1 (used as count*0x28-0x28 to find the last record) - path_GetKeyframeOffset @0601af5c.
- +0x11 byte: low 2 bits = loop/ping-pong style (value 3 -> offset 0) - path_GetKeyframeOffset.
- +0x12 short: frame limit used to decide "past the keyframes" - path_GetKeyframeOffset.
- +0x14 short: total frame count of the path. Mirrored frame = count - frame - 1 - path_SampleFramePosition @0601b970, path_GetKeyframeOffset, anim_StepFrameChecked @06024698.
- +0x1c pointer: keyframe array, stride 0x28 bytes. Each record has a frame range (shorts at +0x0c and +0x34 = next record's start) - path_GetKeyframeOffset. Result is (record - base) * const >> 3.
- +0x20 pointer: optional parent matrix block. Nine shorts at +0..+0x10 (3x3, each << 4 to reach 16.16) and three ints at +0x20/+0x24/+0x28 (translation) - path_SampleFramePosition, player_ComputeWorldPos @06028cb0, player_RefreshRailPlacement @0602dda8, player_ApplyMotionAndRail @0602d3dc. This lets a rail ride on a moving parent (platform/moving rail).
- +0x10.. (as an address) is passed to anim_SplinePosition as the spline control data, so the actual point data is an in-line spline table starting at path+0x10 - path_SampleFramePosition.

### Rail piece (object with a path attached)
- +0x0e ushort flags: bit 0x08 = this piece has a mirror piece (reverse-direction twin) stored at +0x10; bit 0x10 = piece is the reversed twin / direction flag; bit 0x01 / 0x11 test = "is a plain rail" in path_LocatePlayerOnRail @0602cd98; bit 0x40 = branch point that uses heading test in path_UpdateBranchChoice @0602853c.
- +0x10 pointer: mirror piece (used when flag 0x08 is set) - anim_GetLinkedSequence @06024070, path_LocatePlayerOnRail.
- +0x14 pointer: next linked piece (the "end" link); +0x10 is used as the prev/start link when no mirror - anim_GetLinkedSequence.
- +0x1c pointer: forward link to the neighbouring piece used by path_LocatePlayerOnRail when stepping off an end.
- +0x24 pointer: path object (above).
- +0x44 pointer: branch table. +0x48 = forward branch target, +0x50 = fork branch target, +0x04/+0x08 = distance thresholds for the fork - path_UpdateBranchChoice.
- +0x64 pointer: currently chosen branch (player/object side) - path_UpdateBranchChoice. +0x7e short: current node index on that branch.

### Moving object / player (param_1 in most functions)
- +0x10/+0x14/+0x18 ints: position in the rail's local space, 16.16 - player_ComputeWorldPos, player_RefreshRailPlacement.
- +0x1c uint flags: 0x10 = travelling in reverse along the piece (the key direction bit; fed to every path function as the "& 0x10" argument), 0x40 set by path_EvalRailPositionSigned when node index is negative; other bits (airborne, on-rail) are masked by unresolved DAT_ constants.
- +0x38/+0x3c/+0x40: world position (also used by mover_UpdateRailNodes for heading toward the rail target).
- +0x58/+0x5c/+0x60: previous/world delta vector (saved before the move, added to platform velocity) - player_ApplyMotionAndRail, path_LocatePlayerOnRail (atan2 of +0x60,+0x58 = movement heading).
- +0x24 pointer: path of the piece the object is on; +0x20 pointer: parent matrix block (null = world space); +0x78 pointer: current rail piece; +0x79 byte kind code; +0x7a byte object type (1 = player-like, 2 = other mover) - mover_UpdateRailNodes @0602b2fc, collide_IsOnDifferentRailPiece @060299bc.
- +0x7e short: node/frame index on the piece (frame count units); stored through the global context at +2 as well.

### Player context struct (param_2 in player_/mover_ functions; also a global pointer for the single player)
- [0] uint state/flag word; [1] more flag bits (bit set/cleared in player_ApplyMotionAndRail).
- [5] (+0x14) signed rail speed (one scalar along the rail) - mover_ClampRailSpeed @0602aa58 clamps it, mover_SteerAlongRailSegment writes it.
- [6] (+0x18) a second speed (limit / target), [7] (+0x1c) a boolean "reached limit" flag set to 1 together with [6].
- [8],[9],[10] (+0x20..+0x28): velocity vector x,y,z, 16.16; zeroed when entering a piece - player_EnterRailPiece @060292ac, mover_UpdateRailNodes.
- [0x0b] short (+0x2c) timer cleared on piece change.
- +0x3c/+0x40/+0x44: facing/direction vector along the rail (written by path_ComputeSegmentDirection, normalised with math_FixedSqrt, negated on reversal) - player_RefreshRailPlacement, player_ReverseRailDirection @060297b4.
- +0x48/+0x49/+0x4a bytes (of the global ctx): small clamped counters (min-merged) set when the piece changes - player_ApplyMotionAndRail.
- +0x78 pointer: current rail piece; +0x1c..+0x24 ints: piece-entry position; +0x04..+0x0c: previous-piece position - path_SwitchRailPiece @06029200.

## 2. Per-frame flow (as far as the call graph shows)
1. player_BlendPositionToTarget @06028e20: polls pad, samples the animation point of current piece (obj_SampleAnimPoint), decrements a countdown at +0x34 of a global block, calls player_ComputeWorldPos, and while the countdown is nonzero divides the remaining delta by it (math_DivideSigned32) so the object glides to the new target after a piece switch.
2. mover_UpdateRailNodes @0602b2fc: reads 6-byte node records (three shorts, index>>1, parity chooses sign) of the current piece, aims speed and heading at the next node, plays rail sounds for some kind codes, then calls mover_SteerAlongRailSegment @0602ae10 which projects velocity onto the segment direction, updates [5]/[6], and flips sign when heading differs from the segment by more than a threshold (atan2 compare).
3. mover_ClampRailSpeed @0602aa58: clamps [5] against min/max (positive and negative direction use different limits), applies a sine of the slope angle (math_SinTableLookup) for gravity-along-slope, mixes with a factor of (1.0 - friction) via 16.16 multiply.
4. player_ApplyMotionAndRail @0602d3dc: applies velocity, scales velocity by (const - [0x58 field]) i.e. a drag factor, zeroes velocity components under certain flags, adds platform velocity from the parent (+0x20/+0x24/+0x28 of the parent block), calls path_LocatePlayerOnRail @0602cd98 to find the piece/node the new position lies on, then obj_SetFacingFromParent, then on piece change player_StartImpulseTimers and path_SwitchRailPiece.
5. path_LocatePlayerOnRail @0602cd98: loop - path_FindNearestFrame @0601ba68 on the candidate piece; path_ProjectOntoTrackFrames @06024794 projects the point onto the frames; if the result is off the end it follows links (anim_GetLinkedSequence, mirror piece, forward link) and tries again; on success stores path at obj+0x24 and node at obj+0x7e and updates flags.
6. player_UpdateRailPlacement @060289a4 / player_RefreshRailPlacement @0602dda8: transform local position by the parent matrix into world position (shorts<<4 into 16.16), compute segment direction with path_ComputeSegmentDirection @0601b774, negate it if the direction bit differs from the piece's own, set optional output vector.
7. Branch: path_UpdateBranchChoice @0602853c (see section 4).

## 3. Numbers
- Fixed point: 16.16 everywhere; matrix shorts are 4.12-ish and shifted left 4 (<<4) to reach 16.16 (player_RefreshRailPlacement, player_ComputeWorldPos).
- Angles are 16-bit shorts; the branch tolerance is +-0x3f (63) units in path_UpdateBranchChoice @0602853c. The angle unit (0x10000 or 0x1000 per circle) is not proven, so the degree value is unknown.
- Direction bit: 0x10 in object flags (+0x1c). Negative node index sets 0x40 - path_EvalRailPositionSigned @0602896c.
- Keyframe stride: 0x28 bytes - path_GetKeyframeOffset @0601af5c, enemy_AdvanceRailKeyframe @0601aff4.
- Node record: 6 bytes (3 shorts), index>>1 - mover_UpdateRailNodes.
- Blend divisor: countdown at ctx+0x34, decremented once per frame - player_BlendPositionToTarget (the length in frames is set elsewhere).
- Drag: velocity is multiplied by (DAT_0602d57a - field+DAT_0602d578) once per frame - player_ApplyMotionAndRail; both constants are unresolved pool values, so no friction number can be quoted.
- Speed clamps, slope gravity, jump impulses, camera values: not in this slice (constants are pool words not visible; see notes_move / notes_cam_enemy for those).
- Landing fixup: path_EvalRailPosition @060286fc adds a y offset based on the distance to the next frame when flag 1 is set (bounded by a pool constant), a "stick to ground over a gap" feel.

## 4. How the rail works
- A path is a list of frames; a frame index along the path maps to a 3D point via anim_SplinePosition (called from path_SampleFramePosition @0601b970 and path_SampleObjectRailPoint @0601b4cc). If the direction flag is clear the frame index is mirrored (count - frame - 1) so one piece can be run both ways. If the path has a parent matrix (+0x20), the sampled point is transformed by the 3x3 short matrix, then translated.
- Position along the rail for a free 3D point: path_FindNearestFrame @0601ba68 (squared distances plus cross products to the neighbouring frames) then path_ProjectOntoTrackFrames @06024794 gives a node/frame index. This means the player's world position is the truth, and rail index is derived each frame, not the other way round.
- path_EvalRailPosition @060286fc: node index on a piece -> interpolated position. It samples the point just before and just after the node, builds a segment vector, normalises with math_FixedSqrt and interpolates, so fractional nodes are fine.
- Facing: path_ComputeSegmentDirection @0601b774 gives the direction between consecutive keyframes (stride 0x28); player_RefreshRailPlacement stores it at ctx+0x3c..0x44 and negates it when the direction bit disagrees. player_BuildFacingMatrices @0601c460 turns it into rotation matrices; player_SetCurrentRailPiece @0601c9fc rebuilds them when the angle changed (compares a short at a ctx offset) before storing the new piece at +0x78.
- Placement/snap: player_EnterRailPiece @060292ac clears tile flags (coll_ClearTileFlag), converts position into the piece's space, evaluates entry position and zeroes velocity. path_SwitchRailPiece @06029200 evaluates the new piece at the new node, flips the direction bit if the new piece shares the same path object as the old one (mirror twin), calls obj_ApplyPlatformDelta when the path differs, then calls player_SetCurrentRailPiece.
- Reversal: player_ReverseRailDirection @060297b4 swaps to the mirror piece (flag 0x08 / +0x10), negates the direction/velocity vectors (+0x3c/+0x40/+0x44 and +0x48), copies position into the new piece and calls path_SwitchRailPiece; if the piece changed it also calls player_StartImpulseTimers. collide_IsOnDifferentRailPiece @060299bc decides whether the transition is allowed (different path pointers, or a flag bit allows it).
- Branches: path_UpdateBranchChoice @0602853c takes the piece's branch table (+0x44). Default is the forward link (+0x48). If the piece has flag 0x40, the object is within the +4/+8 thresholds, and the angle between its heading and the fork direction (atan2 of the node delta) is within 0x3f, it takes the fork link (+0x50) instead. The choice is stored at obj+0x64 and a "changed" flag bit is set on the object; then anim_SelectChildSequence is called to start the new piece's sequence.
- Running off the end of a piece: path_LocatePlayerOnRail follows the end link via anim_GetLinkedSequence @06024070, which picks the next or previous linked piece depending on the direction bit and sets the new start frame to 0 or count-1; if the target has a mirror (flag 0x08) it is used and the direction bit flips.
- Moving rails: player_ApplyMotionAndRail adds the parent block's velocity (+0x20/+0x24/+0x28 ints) to the object's delta each frame, so riders move with the parent.
- Enemies use the same system: enemy_AdvanceRailKeyframe @0601aff4 and enemy_SpawnProjectileOnRail @0601ab64 use path_GetKeyframeOffset to step keyframe records and place projectiles along the rail.

## 5. State machines
- Direction state: forward/reverse (obj flag 0x10), toggled by piece-mirror or end-link traversal (path_SwitchRailPiece, player_ReverseRailDirection, anim_GetLinkedSequence).
- Rail attach state: attached to a piece (+0x78 set, +0x7e index valid) vs unresolved (path_LocatePlayerOnRail returns 0 and the caller keeps the old piece).
- Branch state: forward vs fork chosen per update (path_UpdateBranchChoice), reset to forward each call unless the fork test passes, so the choice is stateless apart from obj+0x64.
- mover_UpdateRailNodes kind codes at obj+0x79 (0x13, 0x14, 0x17, 0x1b seen) select special rail behaviours (sound request, speed override, effect spawn); the meaning of each code is not decoded here.
- Blend state: countdown at ctx+0x34 > 0 means "gliding to new target after a piece switch" (player_BlendPositionToTarget).

## 6. Suspected wrong names
- 0x0601b774 path_ComputeSegmentDirection - the hand comment says it returns a squared length of the delta between keyframes (it was previously path_ComputeSegmentDeltaLength), but player_RefreshRailPlacement uses it to fill a direction vector, so the current name is plausible; keep, but its comment is misleading.
- 0x06024070 anim_GetLinkedSequence -> path_GetLinkedPiece  it follows rail piece links (+0x10/+0x14, mirror flag 0x08) and is called by path_LocatePlayerOnRail; "anim" is only because pieces carry sequences.
- 0x060240e8 anim_AdvanceSequenceFrame / 0x06024698 anim_StepFrameChecked -> path_ prefix likely better (they step the frame index along the path with the 0x10 direction flag and call path_SampleFramePosition); low confidence.
- 0x060299bc collide_IsOnDifferentRailPiece -> path_CanSwitchToPiece  it compares two piece paths and flag bits and returns "transition allowed" (as the hand comment itself says), not a collision test.
- 0x0602aa58 mover_ClampRailSpeed -> mover_ApplyRailSpeedLimits  also folds in slope sine and friction, more than a clamp (low confidence).
- 0x0602853c path_UpdateBranchChoice calls obj_UpdateHitReaction 0x06028660 (callee) - that callee is used to fetch node positions here, so it is probably mis-named (path/obj position fetch), not hit reaction.
- 0x060289a4 player_UpdateRailPlacement and 0x0602dda8 player_RefreshRailPlacement are near duplicates; one is the init/first-frame version, the other the per-frame; names do not say which.

## 7. Takeaways for a modern 2.5D rail platformer
1. Derive rail progress from world position each frame (nearest-frame + projection) instead of storing a distance along the rail; it makes knockback, platforms and pushes free and always consistent.
2. Make every rail piece reversible by a mirror twin and a single direction bit; mirroring the frame index (count - i - 1) avoids authoring two directions.
3. Store the rail as spline keyframes (here 0x28 bytes each) and sample by fractional frame; build tangents by sampling the point before and after the node, so facing is smooth without authored angles.
4. Allow rails to carry a parent transform (3x3 + translation) so moving platforms and rotating sections are the same code as static rails; add parent velocity to the rider each frame.
5. Branches need only: forward target, fork target, a distance window, and a heading tolerance (about the +-0x3f test); the player steers into a fork by facing it, no explicit button.
6. After a piece switch, blend the position over a few frames (divide remaining delta by a countdown) to hide seams between pieces.
7. Keep piece-to-piece transitions gated by an explicit "can leave this piece" test so ledges, gates and one-way rails are data, not code.
8. Use one rail scalar speed plus a vector velocity: rail speed drives running; the vector handles knockback and platform motion, and both are reconciled by the locate step.
9. Clamp rail speed with different limits per direction, add slope-sine to speed, and a per-frame drag multiply; it gives a platformer feel without a physics engine.
10. Enemies and projectiles can reuse the same path code (keyframe stepping), so one rail library serves player, enemies and moving hazards.
