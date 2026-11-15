# Clash 3D PBR Articulated Lottie Rigging Skill

Reusable AI agent skill for building Clash Royale-style 3D PBR articulated Lottie animations with a 9-state machine (`idle`, `dash`, `attack`, `hit`, `victory`, `spawn`, `charge`, `special`, `defeat`).

## Install on Any Machine

```bash
npx skills add minnyww/clash-3d-lottie-rigging
```

## What This Skill Covers

- 3D PBR character asset preparation & enclosed white-island background cleanup
- 5-part anatomical puppet splitting (`head_hat`, `right_arm_staff`, `left_arm`, `tail`, `body`) with joint inpainting
- Dark-background extreme-pose seam verification before Lottie export
- 60 FPS, 745-frame Lottie 5.9.0 JSON + 9-state machine rigging rules (`child_sync_opacity()`, boundary keyframes, continuous spell VFX)
