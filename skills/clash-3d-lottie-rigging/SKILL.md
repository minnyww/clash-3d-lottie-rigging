---
name: clash-3d-lottie-rigging
description: End-to-end guide to build Clash Royale-style 3D PBR articulated Lottie character animations with a 9-state machine (idle, dash, attack, hit, victory, spawn, charge, special, defeat). Designed to be shared across AI agents so any agent can reproduce clean 3D puppet splitting, joint inpainting, seamless multi-part Lottie rigging, continuous spell VFX, and HTML state-machine integration.
---

# Clash Royale 3D PBR Articulated Lottie Rigging Skill

Use this skill whenever an AI agent needs to build an interactive 3D-rendered (PBR) character Lottie animation with a multi-state machine—preserving full 3D shading while achieving detailed, articulated limb/head/tail movement with **zero double-image ghosting** and **zero seam tearing**.

---

## 0. Project Architecture

```text
project-root/
├── .agent/skills/clash-3d-lottie-rigging/SKILL.md
├── assets/
│   ├── <char>-raw.png                  # 1024x1024 3D PBR render on #FFFFFF
│   └── <char>-parts/                   # Cleanly split & inpainted 512x512 RGBA parts
│       ├── body.png
│       ├── head_hat.png
│       ├── right_arm_staff.png
│       ├── left_arm.png
│       └── tail.png
├── scripts/
│   └── generate_<char>_lottie.py       # Self-contained splitter + Lottie JSON builder
├── lottie/
│   └── <char>.lottie.json              # Output Lottie 5.9.0 (512x512, 60 FPS, 745 frames)
├── state-machine.js                    # 9-State Machine controller over lottie-web segments
└── index.html                          # Interactive deck + live state graph + triggers
```

---

## 1. Character Asset Requirements (3D PBR)

When generating or preparing the base character PNG (`1024x1024` or higher):
- **Style**: Stylized 3D PBR mobile game character (Clash Royale aesthetic), crisp studio rim lighting, clean silhouette.
- **Pose for Clean Rigging**: Front-3/4 standing pose on a **solid pure white (`#FFFFFF`) background** with slight separation between weapon/staff, casting hand, torso, and tail so parts can be segmented cleanly.
- **Never** replace a requested 3D PBR character with flat 2D vector shapes; always rig the 3D rendered parts directly.

---

## 2. Background Removal & Enclosed Island Cleanup

1. **Edge Floodfill**: Floodfill from all 4 image borders removing near-white pixels (`r >= 238, g >= 238, b >= 238, max-min <= 16`).
2. **Enclosed White Islands**: Gaps between robe hems, legs, arms, and tail often trap pure white background islands that edge floodfill cannot reach. Remove neutral-white pixels (`r >= 242, g >= 242, b >= 242, max-min <= 10`) in lower/outer gap zones while protecting warm cream fur/paws (`r - b > 10`) and eye highlights.
3. **Alpha Edge Polish**: Apply a `0.8px` Gaussian blur on the alpha channel only at semi-transparent border pixels (preserving `0` and `>= 245` interiors).
4. **Canvas Normalization**: Crop to tight bounding box and paste centered onto a `512x512` transparent canvas (`target_size = int(512 * 0.82)`).

---

## 3. Articulated Puppet Splitting & Inpainting

To animate head, arms/weapon, and tail independently **without ghosting or tearing**:

1. **Define 5 Anatomical Layers**:
   - `head_hat` (head + hat/hair, extending ~10px into collar overlap)
   - `right_arm_staff` (weapon/staff + hand, pivoting at wrist inside sleeve cuff)
   - `left_arm` (casting arm/paw + sleeve, extending ~18px inward at shoulder cap)
   - `tail` (or cape/wings, rendered **behind** `body`, extending ~16px behind robe)
   - `body` (torso + robe + feet, with `head_hat`, `right_arm_staff`, `left_arm`, and `tail` erased)
2. **Inpaint Joint Seams on `body`**:
   - Never leave hollow holes behind erased limbs. Run `18–22` iterations of 8-neighbor diffusion inpainting strictly inside narrow seam patches at the neck collar and both shoulders, masked to the body silhouette.
3. **Prevent Joint Separation & Clipping**:
   - Keep joint translations tiny (`±3–6px` max). Never translate a shoulder or neck `20px+` away from `body`.
   - Use **rotation** around the anatomical pivot + **foreshortening scale (`108%–115%`)** for forward attacks/thrusts so weapons surge toward the viewer without detaching from the cuff or clipping through the face.
4. **Mandatory Visual Verification**:
   - Before finalizing, composite the 5 split parts onto a **dark background (`#121C30`)** across all extreme poses (attack windup, attack thrust, victory peak, special windup/slam) and inspect the rendered strip image visually to confirm **zero white islands, zero gaps, and zero clipping**.

---

## 4. Lottie 5.9.0 Rig & 9-State Machine Spec (`60 FPS`, `745 frames`)

Standard state machine segments (`markers` in Lottie JSON & `state-machine.js`):
- `idle`: `0–120` (loop) — breathing, head follow-through, gentle tail wag, subtle staff/paw bob, natural fur-shaded blink (`f=66..77`)
- `dash`: `120–180` (loop) — forward lean, arms swept back, tail trailing
- `attack`: `180–255` (one-shot → `idle`) — anticipation windup (`180–194`) → snap thrust + foreshortening scale (`194–207`) → continuous orb-to-target spell beam + stars → settle
- `hit`: `255–305` (one-shot → `idle`) — sharp knockback recoil + opacity flicker
- `victory`: `305–425` (loop) — 3 even celebratory hops spanning the **entire** 120-frame loop (no dead pause at the end), staff/paw pumps, fast tail wag, rising victory sparkles, glowing rune ring
- `spawn`: `425–495` (one-shot → `idle`) — drop-in from `y=-80` with squash-and-stretch landing
- `charge`: `495–575` (loop) — raised staff/paw, pulsing orb & rune aura
- `special`: `575–665` (one-shot → `idle`) — high windup → aerial slam + ultimate spell beam
- `defeat`: `665–745` (hold) — knockback jolt → slump/shrink (`100% → 76%`) → smooth full-character fade (`100 → 0` by `f=732`) + 38+ rising ash/elixir particles

---

## 5. Critical Lottie Pitfalls (Must Follow)

1. **Parent Opacity Does NOT Inherit**:
   - In `lottie-web`, child layers inherit `position`, `rotation`, `scale`, and `anchor`, **NOT `opacity`**.
   - Every child layer (`HeadHat`, `RightArmStaff`, `LeftCastingArm`, `Tail`, `Eyelids`) **must** share a synchronized `child_sync_opacity()` curve for `spawn` (`424:100 -> 425:0 -> 434:100`) and `defeat` (`665:100 -> 732:0 -> 745:0`). Otherwise `Body` vanishes while the head/arms/tail float in mid-air.
2. **Always Lock Boundary Keyframes Before Sudden Changes**:
   - Never leave a gap between `276` (`100%`) and `425` (`0%`) on `opacity`—that causes `Victory` (`305–425`) to linearly fade out! Always insert `(424, 100, LINEAR)` right before `(425, 0, LINEAR)`.
3. **Avoid Per-Layer Rectangular Dissolve Masks on Multi-Part Rigs**:
   - Applying a local-space wipe mask to 5 separate `512x512` layers causes uneven horizontal cut lines across joints. Use synchronized opacity fade + scale-down + rising ash/ember particles instead.
4. **No Cheap 2D Shapes on 3D PBR Faces**:
   - Never draw flat 2D `✕` shapes or flat single-color ovals over 3D PBR eyes during `defeat`. For `idle` blinks, use a gradient fur-matched lid + dark curved lash stroke, and hide it during `defeat`.
5. **Continuous Spell Energy**:
   - Anchor `OrbCrystalGlow` (parented to weapon/staff) and `PawMagicAura` (parented to casting arm) at the exact measured pixel coordinates of the orb/paw. Ramp orb glow *before* launching `ArcaneSpellBeam` and staggered `MagicStar` projectiles (`delay = idx * 2`) from the exact orb coordinates.
