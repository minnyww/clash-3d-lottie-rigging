---
name: clash-3d-lottie-rigging
description: Universal workflow for AI agents to build Clash Royale-style 3D PBR articulated Lottie animations with a 9-state machine for ANY character archetype (mages, knights, archers, dragons, mascots, monsters). Covers 3D asset prep, background/island cleanup, dynamic limb segmentation + joint inpainting, programmatic pivot measurement, archetype-specific secondary motion, continuous VFX, and dark-background seam verification.
---

# Universal 3D PBR Articulated Lottie Rigging Skill

Use this skill to turn **any** 3D-rendered (PBR) character—mage, knight, archer, dragon, demon, robot, or cute mascot—into an interactive, articulated Lottie animation with a 9-state machine, preserving full 3D shading with **zero double-image ghosting** and **zero seam tearing**.

---

## 1. Character Design & Asset Preparation (Any Archetype)

When generating or preparing the base character PNG (`1024x1024` or higher):
- **Style**: Stylized 3D PBR game character (studio rim lighting, rich materials, clean silhouette).
- **Rig-Friendly Pose**: Front-3/4 standing or hovering pose on a **solid pure white (`#FFFFFF`) background** with clear separation between the torso and movable appendages (weapon arm, offhand arm, wings, cape, or tail) so polygons can cleanly isolate each part.
- **Never** substitute flat 2D vector art when 3D PBR style is requested; always rig the 3D rendered layers directly.

---

## 2. Universal Background & Enclosed-Island Removal

1. **Border Floodfill**: Floodfill from all 4 borders removing near-white pixels (`r >= 238, g >= 238, b >= 238, max-min <= 16`).
2. **Enclosed White Islands**: Gaps between legs, capes, tails, and arms trap pure white background pockets unreachable from borders. Clean neutral-white pixels (`r >= 242, g >= 242, b >= 242, max-min <= 10`) in gap regions while protecting intentional highlights (eyes, glowing crystals, metallic speculars, warm cream fur/skin).
3. **Alpha Edge Polish**: Apply a `0.8px` Gaussian blur only to boundary alpha pixels (keeping `0` and `>= 245` interiors intact).
4. **Canvas Normalization**: Crop to tight bounding box and center onto a `512x512` transparent canvas (`target_size = int(512 * 0.82)`).

---

## 3. Dynamic Puppet Splitting & Joint Inpainting

Inspect the normalized `512x512` character image first, then split it into **only the parts that character actually has**:

- **Root (`body`)**: Torso, armor/robe, hips, legs/feet.
- **Head (`head`)**: Head + helmet/hat/horns/ears (extend ~10px downward into neck/collar overlap).
- **Primary Limb (`weapon_arm` / `right_arm`)**: Sword, staff, bow, hammer, or claw + forearm (pivot at shoulder or wrist inside sleeve/gauntlet cuff).
- **Secondary Limb (`offhand_arm` / `left_arm`)**: Shield, casting hand, or offhand (extend shoulder cap ~16–18px inward).
- **Back Appendages (`tail`, `wing_left`, `wing_right`, `cape`)**: Rendered **behind** `body`, extending ~14–16px behind the torso silhouette so they never detach when flapping or wagging.

### Joint Seam Rules (Zero Ghosting, Zero Gaps)
1. **Erase + Inpaint**: Erase every movable part from `body`, then run `18–22` iterations of 8-neighbor diffusion inpainting strictly inside narrow seam patches (neck collar, shoulders) masked to the `body` silhouette.
2. **Measure Coordinates Programmatically**: Never hardcode pivots or VFX origins from another character. Measure the current character's exact joint pivots, eye centers, and weapon/orb/muzzle tip directly from its normalized `512x512` pixel map.
3. **Constrain Joint Translation**: Keep joint `position` offsets tiny (`±3–6px`). Use **rotation** around the true anatomical pivot + **foreshortening scale (`108%–115%`)** for forward attacks so limbs never tear off the torso or clip through the face.
4. **Mandatory Dark-Background Pose Check**: Before exporting final Lottie JSON, composite all split parts onto `#121C30` across extreme poses (windup, attack thrust, victory peak, special slam) and visually inspect the rendered strip for white islands, gaps, or clipping.

---

## 4. Archetype Motion & Continuous VFX Guidelines

Adapt the secondary motion & VFX to match the character's role:
- **Mages / Casters**: Charge glow at measured staff-orb/hand coordinates *before* firing; launch continuous gradient beam + staggered sparkles from the exact orb/hand position.
- **Knights / Melee Fighters**: Heavy anticipation windup (`-25°` to `-35°`), fast snap slash/slam with weapon foreshortening (`112%–116%`) + crescent slash arc & impact shockwave at weapon tip.
- **Archers / Gunners**: Draw-back tension hold -> crisp release recoil -> continuous projectile streak + muzzle flash at weapon tip.
- **Dragons / Winged Mascots**: Independent flapping cycles on `wing_left` / `wing_right`, delayed S-curve follow-through on `tail`, squash-and-stretch hops, breath/fire burst from snout coordinates.

---

## 5. Lottie 5.9.0 & 9-State Machine Spec (`60 FPS`, `745 frames`)

Standard state segments (`markers` + `state-machine.js`):
- `idle` (`0–120`, loop): Breathing, delayed head follow-through, gentle tail/wing/weapon sway, optional natural blink.
- `dash` (`120–180`, loop): Forward lean, limbs swept back, speed trail.
- `attack` (`180–255`, one-shot → `idle`): Anticipation → snap strike/cast + VFX → recoil settle.
- `hit` (`255–305`, one-shot → `idle`): Knockback recoil + brief opacity flicker.
- `victory` (`305–425`, loop): 3 rhythmic celebratory hops/pumps spanning the **full** 120-frame loop (no dead pause at the end) + celebratory sparkles/aura.
- `spawn` (`425–495`, one-shot → `idle`): Drop-in from `y=-80` with squash-and-stretch landing.
- `charge` (`495–575`, loop): Weapon/hands raised, pulsing energy aura & rune ring.
- `special` (`575–665`, one-shot → `idle`): High windup / aerial leap → ultimate strike/blast.
- `defeat` (`665–745`, hold): Knockback jolt → slump/shrink (`100% → 76%`) → smooth full-character fade (`100 → 0` by `f=732`) + 38+ rising ash/ember/elixir particles.

---

## 6. Critical Lottie Technical Rules

1. **Sync Child Opacity Manually**: Lottie does **not** inherit parent `opacity`. Every child layer (`head`, arms, `tail`, wings, eyelids) must share `child_sync_opacity()` for `spawn` (`424:100 -> 425:0 -> 434:100`) and `defeat` (`665:100 -> 732:0 -> 745:0`).
2. **Lock Boundary Keyframes**: Always insert `(424, 100, LINEAR)` before `(425, 0, LINEAR)` on opacity so `victory` (`305–425`) never fades out linearly.
3. **Avoid Per-Layer Wipe Masks on Multi-Part Rigs**: Rectangular dissolve masks on separate `512x512` child layers create uneven horizontal cut lines; use synchronized fade + scale-down + rising particles instead.
4. **Preserve 3D Facial Fidelity**: Never draw cheap flat 2D `✕` or unshaded ovals over 3D PBR eyes. Keep 3D eyes clean on `defeat` and use gradient-matched lids + curved lash strokes for `idle` blinks.
