# LocMuse

Self-hosted music and audiobook server on a repurposed laptop. Streams to phone and browser over an encrypted mesh network — no subscription, no tracking, no third party holding the library.

**Status: pre-build.** The architecture is settled and reviewed; nothing has been deployed yet. This repo currently holds the design, not a running system. Commands and configuration here are untested against real hardware until Phase 3 is signed off.

---

## Why

Streaming services rent you access to a catalogue they can change or revoke. This owns the files instead, and treats them as the source of truth.

**Governing principle: tags are truth, the database is a cache.** Neither server stores anything authoritative about *what the media is* — that lives in embedded file tags. The library stays portable to any tool, forever. The databases hold play counts, ratings and playlists, which is why they still get backed up (see Limits in the architecture doc).

## Stack

**Platform** — Ubuntu Server 26.04.1 LTS · Docker + Compose · ext4

**Services** — Navidrome (music, Subsonic API) · Audiobookshelf (progress sync, chapters, m4b) · SQLite per service · ffmpeg

**Metadata** — beets with `fetchart`, `embedart`, `scrub`, `replaygain` · rsgain (EBU R128) · MusicBrainz, Cover Art Archive, Audnexus

**Access** — Tailscale (WireGuard) + `tailscale serve` for mesh networking and real TLS · per-service local accounts

**Ops** — Diun (notify-only image watcher) · restic → Backblaze B2 for config and databases · rsync → second drive for media · healthchecks.io dead-man's switch


## Hardware

| | |
|---|---|
| CPU | Ryzen 5 3550H, 4 cores |
| RAM | 16 GB DDR4 dual-channel |
| GPU | GTX 1650 4 GB (NVENC available if Jellyfin is added later) |
| System disk | 256 GB NVMe Gen3 |
| Media storage | External drive, ext4 — **not yet provisioned** |

Idle footprint of the full stack is under 500 MB, so RAM is not the constraint. The binding constraint is disk I/O during the initial library scan.

Compose files, fstab entries, udev rules and systemd units land here as each phase completes. Secrets stay out — `.env.example` only.
