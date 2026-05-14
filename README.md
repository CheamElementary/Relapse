# Relapse

**Voice-first Linux system for livestreaming the truck + full voice control of the 2025 Freightliner Cascadia while driving.**

> Hands on the wheel, eyes on the road. Voice and ears only.  
> Unix philosophy: *“Everything is a file.”*

## Vision
Relapse is a dedicated system running on an **Orange Pi 5 Plus** in the truck that lets me safely "relapse" back into coding, livestream the road/engine, and eventually issue voice commands to the truck itself — all without taking my hands off the wheel or eyes off the road.

It is built around the **Hear-Eval-Respond Loop (HERL)** concept.

## Hardware

**Truck (Production Target)**
- Orange Pi 5 Plus (RK3588)
- Arducam B0497 (IMX678 / STARVIS2 sensor) – USB/UVC
- Focusrite Scarlett Solo + RODE microphone (engine bay)
- Starlink Flat High Performance (US Roaming)

**Home Office (Heavy Backend)**
- 2× Threadripper 2990WX (128 GB RAM each)
  - One running **Proxmox**
  - One running **TrueNAS CORE**
- Gigabit fiber + Cloudflare Argo Tunnels

## Architecture
- **Local recording** on Orange Pi (1080p50 HEVC + audio) → segmented MP4s
- Optional live RTMP streaming
- Automatic sync of recordings to home Proxmox/TrueNAS when connectivity allows
- Lightweight services on truck, heavy AI/compute offloaded home

## Current Contents
- `history/` — Voice session transcripts (project origins and thinking)
- `main.py` — Early placeholder for HERL core
- Capture scripts (coming next: `scripts/` folder)

## Development Phases

**Phase 0** – Solid base image (Armbian) on Orange Pi  
**Phase 1** – Reliable local capture + automatic upload/sync  
**Phase 2** – Offline voice layer (STT + HERL command router)  
**Phase 3** – Voice control of Cascadia systems  
**Phase 4+** – Additional coderpath.com tools

## How to Resume This Project
In any new chat with Grok (or another AI), just say:

> “Load the full Relapse project from https://github.com/CheamElementary/Relapse”

## Links
- **Linear Project**: https://linear.app/coderpath/project/relapse-6f193ac0a63a
- **Gitea Mirror**: https://gitea.coderpath.com/miles/Relapse

---

*Last updated: May 14, 2026*
