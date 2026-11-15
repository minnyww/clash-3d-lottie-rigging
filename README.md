# Clash 3D PBR Articulated Lottie Rigging Skill

Universal AI agent skill for building Clash Royale-style 3D PBR articulated Lottie animations with a 9-state machine (`idle`, `dash`, `attack`, `hit`, `victory`, `spawn`, `charge`, `special`, `defeat`) for **any character archetype** (mages, knights, archers, dragons, mascots, monsters).

## Install on Any Machine

```bash
npx skills add minnyww/clash-3d-lottie-rigging
```

## What This Skill Covers

- 3D PBR character asset preparation & enclosed white-island background cleanup
- Archetype-agnostic anatomical puppet splitting (`body`, `head`, `weapon_arm`, `offhand_arm`, `tail`/`wings`/`cape`) with joint inpainting
- Programmatic pivot/weapon-tip coordinate measurement
- Dark-background extreme-pose seam verification before Lottie export
- 60 FPS, 745-frame Lottie 5.9.0 JSON + 9-state machine rules (`child_sync_opacity()`, boundary keyframes, continuous archetype VFX)
