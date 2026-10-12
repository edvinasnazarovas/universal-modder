---
kind: game
title: "8-way aim strafing, crouched strafes and armed sprinting in GTA San Andreas"
game: "Grand Theft Auto: San Andreas"
games_also: []
game_version: "1.0 US gta_sa.exe (timestamp 0x427101CA), Mod Loader, CLEO 4.4.4 + CLEO+, SilentPatch"
platform: windows
engine: native   # RenderWare-based 32-bit C++ executable
route: native-hook
tools: ["MSVC 2022 / CMake", "plugin-sdk headers", "SafetyHook-style verified byte patches", "gta-reversed (reference)", "dumpbin /disasm", "Python IFP (ANP3) reader/writer", "Mixamo Pro Rifle Pack"]
anti_cheat: "none (single-player)"
status: working
agents: ["Claude Code (Opus 5.5)"]
humans: []
date: 2026-10-12
links: []
tags: [animation, ifp, anp3, aim, strafe, crouch, sprint, retarget, crash-fix, cleo]
---

# 8-way aim strafing, crouched strafes and armed sprinting in GTA San Andreas

> ASI plugins (native hooks) replace the clunky native aim strafe with 8-way, phase-synced directional clips
> retargeted from a Mixamo rifle pack, add crouched strafing and turn-in-place, fix several long-standing
> animation glitches, and rebuild armed sprinting without a third-party patcher. Everything was verified
> by the human playing the real game, with in-game trace logs and Windows crash records as the oracle.

## Setup
- Retail 1.0 US exe (fingerprint-checked before patching). Mod Loader for asset overrides (`modloader/<mod>/anim/ped.ifp`,
  `modloader/<mod>/models/gta3.img/fat.ifp`), ASI loader for plugins.
- Custom animations ship as a separate ANP3 IFP block loaded at runtime (`um_aimmove`), not by editing stock files.

## Route and why
Native hooks: GTA SA has no animation API. Data-only edits (ped.ifp) cannot add blending logic; CLEO scripts are
too coarse per-frame and run after the animation update.

## How the game works (what we had to learn)
- **Blending:** `CAnimBlendAssociation::UpdateBlend` clamps blend to 1, so you cannot out-weight an animation.
  Per bone, non-partial animations share `1 - sum(partial blends)`; root translations of partials with extract
  flags (0x40/0x80) are summed un-normalised. Directional clips therefore work as PARTIAL associations with
  translation flags: at weight 1 they replace the base pose while the base's blend (which tasks read) is untouched.
- **Static associations** (`CAnimManager::GetAnimAssociation` 0x4D3A60) hold per-node sequence pointers that a
  played copy inherits; nulling a node's pointer stops that clip driving the bone (used to make crouch fire/reload
  upper-body only). Those pointers dangle once the weapon's anim block is streamed out.
- **Player move groups:** `CPlayerPed::ProcessAnimGroups` (0x6098F0) picks 54-56 (player/fat/muscular), 57-59 rocket,
  60-62 two-handed, 63-65 bat, 66-68 chainsaw, from the held weapon; the result is compared/stored at 0x609A44.
- **Sprint gate:** `CTaskSimplePlayerOnFoot` refuses sprint when the group's RUN and SPRINT hierarchies are the same,
  which is true for every armed carry group. The group definitions are a static table of 6 name pointers per group
  (0x8A8AC0 onward); re-pointing the sprint slot at a different clip name (before groups are built) enables it.
- **`CPlayerPed::ReApplyMoveAnims`** (0x609650) runs on group change and replaces any association with ids 0-3 or 5
  whose hash differs from the group's, copying blend amount and delta.
- **Aim strafe internals:** `ControlGunMove` 0x61E0C0 (player site 0x687EF5) ramps 0.07/frame and zeroes small
  commands in `SetMoveAnim`; one-handed (AIMWITHARM) weapons never use GunMove: `PlayerControlZelda` 0x6883D0 turns
  the body instead. Crouch movement: `CTaskSimpleDuck::ControlDuckMove` 0x6923F0; `StopFireGun` needs the
  WEAPON_crouch blend at exactly 1.

## Build steps
1. Retarget Mixamo clips offline (Python): legs + root translation from the source, root rotation/pelvis/spine/arms
   from the native GunMove (or WEAPON_crouch) pose at the same cycle fraction; write an ANP3 block.
2. Runtime: blend only the two neighbouring of 8 directions, keep them on the lead clip's phase, hold other leg-driving
   partials at `1 - weight`, leave 15-bone upper-body overlays (aim/fire/reload) at full weight.
3. Measure foot-plant phases per clip offline; start strides on a plant and finish the current step on release.

## Verification
The human played; plugins logged state. Crashes were resolved from Windows Application-log events (module + offset)
mapped to functions with the linker `.map`. Animation glitches were caught with an on-demand (F9) snapshot of the
whole association stack. Not verified: missions/cutscenes broadly, two-player mode.

## Gotchas
1. **CJ stuck mid-stride while idle, slides when moving.** **Cause:** a custom start clip reused anim id 5
   (WALK_START); `ReApplyMoveAnims` swapped it on a weapon switch for the group's native walk-start at blend 1, delta 0,
   which then froze on its last frame forever. **Fix:** never reuse ids 0-3/5 for your own clips; fade stray id 5s.
2. **Crash when aiming, random.** **Cause:** reading a weapon group's static-association sequence pointers after its
   anim block was streamed out. **Fix:** require the block loaded (`ms_aAnimAssocGroups` 0xB4EA34, stride 0x14, block +0x10)
   and every pointer inside the hierarchy's sequence array.
3. **Crash on every quit (`CLEO+.cleo_unloaded +0x2D710`).** **Cause:** CLEO+ registers a script-delete delegate;
   CLEO 4.4.4 unloads plugins before deleting scripts, then calls it. **Fix:** pin loaded `*.cleo` modules
   (`GetModuleHandleEx` PIN) or update CLEO.
4. **Hierarchy duration reads 0.** **Cause:** compressed until `CAnimManager::UncompressAnimation` (0x4D41C0). **Fix:** uncompress before validating.
5. **"Removing a mod" left behaviour behind.** Stories Sprinting was three parts: an IFP/img override, a pointer-patch
   ASI with a text table, and a CLEO script that NOPs GTA's group write (0x609A4E) and forces the bat group for
   one-handed weapons. The CLEO part was missed at first. Check `CLEO/` and the game root, not just `modloader/`.
6. **Restoring the group remap crashed at load.** The fat/muscular variants of each group load names from
   `fat.ifp`/`muscular.ifp`; every re-pointed name must exist in those too.
7. **Diagonal strafes crossed the legs / moonwalked.** A 4-way forward+side blend crosses legs; phase-offsetting it
   moonwalks. Use 8 directional clips and blend only neighbours.
8. **Out-weighting the duck base made the duck task abort.** Tasks read their base anim's blend; leave it, cover it
   with partials instead.

## Assets
Mixamo "Pro Rifle Pack" (user-supplied) retargeted to CJ's skeleton offline. No generated assets.

## Cost and time
Many sessions over several days; most time went into in-game diagnosis loops with the human.

## Open questions
- An own two-handed sprint built from the rifle pack drove the torso too hard (hip twist); a native-arm, damped-torso version is untested.
- Lead-thigh/buttock skin clipping on diagonal strafes is unresolved (pelvis-follow 0.75 only moved it 60 to 56 degrees).
