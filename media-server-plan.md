# Self-Hosted Media Server — Build Plan

**Hardware:** Old laptop (24/7) + external hard drive with existing music library
**Build time:** ~1 evening for tag cleanup, ~1 hour for everything else

---

## 1. The Plan

### What this is

A private, always-on server streaming your own music and audiobooks to phone and browser — album art, artist metadata, chapter navigation, offline downloads, playback progress synced across devices. No subscription, no tracking, no third party holding your library.

### What was deliberately cut, and why

| Cut | Reason |
|---|---|
| Custom server | Navidrome and Audiobookshelf already solve this. Rebuilding filesystem watching, tag parsing, MusicBrainz matching, transcode session management and HLS segmenting is 6–12 months to reach parity with zero novel outcome. |
| Custom mobile app | The hard parts of a music client are gapless playback, crossfade, ReplayGain, offline sync conflict resolution, and Android Auto. Symfonium already does all of it. This is where DIY projects die. |
| CD / Blu-ray ripping pipeline | You have no discs and no optical drive. Blu-ray additionally requires circumventing AACS, which is legally exposed under India's Copyright Act §65A. Your music files already exist — ingestion is a non-problem. |
| Reverse proxy + domain + TLS | Tailscale gives an encrypted private network with DNS. Caddy, Let's Encrypt, DDNS and port forwarding all become unnecessary, removing an entire class of exposure. |
| Public internet exposure | Nothing listens on a public port. The attack surface is WireGuard, not your service auth. |

### The one place not to be lazy

**Tag quality.** Navidrome treats embedded file tags as the source of truth and does no fuzzy matching. A library with inconsistent artist names, missing `ALBUMARTIST` fields and absent cover art renders as duplicate artists and blank tiles. One `beets` pass against MusicBrainz fixes this permanently. This is the difference between a Plexamp-grade experience and something you abandon in three weeks.

### Governing principle: tags are truth, the database is a cache

Neither server stores anything authoritative about *what the media is* — that lives in the files. Delete the SQLite DB, rescan, and you lose nothing but play counts. Your library stays portable to any tool, forever. That is the actual meaning of owning it.

### Phase plan

| Phase | Work | Time |
|---|---|---|
| 0 | Prepare laptop — Ubuntu Server, lid behaviour, Docker, Tailscale | 40 min |
| 1 | Mount drive permanently via fstab | 10 min |
| 2 | Tag cleanup with beets | 1 evening |
| 3 | Bring up Docker stack | 10 min |
| 4 | Tailscale + clients | 10 min |
| 5 | Backups and update automation | 15 min |

---

## 2. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| **Host OS** | Ubuntu Server 24.04 LTS | Headless, low idle draw, no forced reboots, 5-year support. Windows on a 24/7 box costs RAM and reboots unpredictably. |
| **Runtime** | Docker + Compose | Whole stack in one declarative file. Rebuild from scratch in 10 minutes. |
| **Music server** | Navidrome (Go) | Idles ~50 MB RAM, single binary, embedded SQLite. Natively speaks the Subsonic API — a 15-year-old standard with 20+ mature clients. |
| **Audiobook server** | Audiobookshelf (Node/Vue) | Per-user progress sync, chapter navigation, m4b merging, Audnexus metadata, official iOS/Android apps. |
| **Video (optional)** | Jellyfin | Only if you add movies later. Mounts the same music folder read-only alongside Navidrome with no conflict. |
| **Metadata DB** | SQLite, embedded per service | Zero admin. Postgres is pure overhead at single-user scale. |
| **Metadata sources** | MusicBrainz, Cover Art Archive, Audnexus | Free, open, no API keys for basic use. |
| **Tag pipeline** | beets + `fetchart`, `embedart`, `scrub` | Writes canonical tags into the files. Run once, then only for new additions. |
| **Transcoding** | ffmpeg (bundled in both images) | Nothing to configure. |
| **Search** | Built-in SQLite FTS | Meilisearch is unjustified below ~100k tracks. |
| **Auth** | Per-service local accounts | No OAuth, no external IdP, no account with anyone. |
| **Remote access** | Tailscale (WireGuard) | Replaces reverse proxy, TLS, DDNS and port forwarding in one install. |
| **Android client** | Symfonium (~₹500 one-time) | Gapless, crossfade, ReplayGain, offline sync, Android Auto. The real Plexamp equivalent. Free: Tempo, substreamer. |
| **iOS client** | Amperfy or play:Sub | Both support offline caching. |
| **Audiobook client** | Official Audiobookshelf app | Offline downloads, sleep timer, speed control, chapter jump. |
| **Web client** | Built-in Navidrome + ABS web UIs | Both are PWAs, installable on desktop. |
| **Updates** | Watchtower, weekly | Pulls new images Sunday 4 AM, cleans old layers. |
| **Backups** | restic → Backblaze B2 | Config + DBs only (~few hundred MB). Media stays on the drive. |

---

## 3. Architecture

### 3.1 System topology

```
┌─────────────────────────────────────────────────────────────────┐
│  OLD LAPTOP  —  Ubuntu Server 24.04 LTS, headless, lid ignored  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Docker Engine                         │  │
│  │                                                           │  │
│  │  ┌──────────────┐  ┌──────────────────┐  ┌─────────────┐  │  │
│  │  │  Navidrome   │  │  Audiobookshelf  │  │ Watchtower  │  │  │
│  │  │  (Go)        │  │  (Node / Vue)    │  │             │  │  │
│  │  │  :4533       │  │  :13378          │  │ weekly      │  │  │
│  │  │              │  │                  │  │ image pull  │  │  │
│  │  │  Subsonic    │  │  REST API + PWA  │  └─────────────┘  │  │
│  │  │  API + Web   │  │  Audnexus client │                   │  │
│  │  │  SQLite      │  │  SQLite          │                   │  │
│  │  │  ffmpeg      │  │  ffmpeg / tone   │                   │  │
│  │  └──────┬───────┘  └────────┬─────────┘                   │  │
│  └─────────┼───────────────────┼─────────────────────────────┘  │
│            │ read-only         │ read-write                     │
│  ┌─────────▼───────────────────▼─────────────────────────────┐  │
│  │      EXTERNAL HARD DRIVE  (fstab, mounted by UUID)        │  │
│  │   /mnt/media/music/       FLAC + MP3, tagged by beets     │  │
│  │   /mnt/media/audiobooks/  m4b with embedded chapters      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Tailscale daemon — WireGuard, MagicDNS name "mediabox"   │  │
│  └────────────────────────────┬──────────────────────────────┘  │
└───────────────────────────────┼─────────────────────────────────┘
                                │  encrypted mesh, no open ports
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
   ┌────▼─────┐          ┌──────▼──────┐        ┌───────▼──────┐
   │ Android  │          │  MacBook    │        │  Desktop PC  │
   │ Symfonium│          │  Browser    │        │  Browser     │
   │ + ABS app│          │  (web UI)   │        │  (web UI)    │
   └──────────┘          └─────────────┘        └──────────────┘
```

### 3.2 Access path — phone on mobile data reaching the server

```
  Phone (4G/5G, Symfonium)
        │
        │  1. Tailscale client resolves "mediabox" via MagicDNS → 100.x.y.z
        ▼
  ┌────────────────────────────────────────────────┐
  │  Tailscale coordination server                 │
  │  (key exchange + NAT traversal ONLY —          │
  │   never sees your media traffic)               │
  └────────────────────────────────────────────────┘
        │
        │  2. Direct WireGuard tunnel punched through both NATs.
        │     Falls back to an encrypted DERP relay only if
        │     hole-punching fails. E2E encrypted either way.
        ▼
  Laptop :4533  →  Navidrome  →  token auth  →  stream
```

Your ISP's CGNAT does not need to cooperate, no dynamic DNS is required, no service is reachable by internet scanners, and a Navidrome auth bug cannot be exploited by anyone outside your tailnet.

### 3.3 Ingestion and metadata flow

```
  Existing files on hard drive
  (mixed tags, missing art)
        │
        ▼
  ┌──────────────────────────────────────────────────┐
  │  beets import                                    │
  │   ├─ metadata match ─────────► MusicBrainz API   │
  │   │                            (canonical artist,│
  │   │                             album, MBIDs)    │
  │   ├─ fetchart plugin ────────► Cover Art Archive │
  │   └─ writes tags + art BACK INTO the files       │
  └──────────────────┬───────────────────────────────┘
                     │  clean, self-describing files
                     ▼
     /mnt/media/music/Artist/Album (Year)/01 Track.flac
                     │
                     │  filesystem watch + 6h scheduled scan
                     ▼
  ┌──────────────────────────────────────────────────┐
  │  Navidrome scanner                               │
  │   reads embedded tags → builds SQLite index      │
  │   (artist / album_artist / album / track / MBID) │
  └──────────────────┬───────────────────────────────┘
                     ▼
              Subsonic API surface


  Audiobook files (m4b)
        │
        ▼
  ┌──────────────────────────────────────────────────┐
  │  Audiobookshelf scanner                          │
  │   ├─ reads embedded m4b chapter atoms            │
  │   ├─ Audnexus API ──► cover, series, narrator    │
  │   └─ optional: merge multi-mp3 → single m4b      │
  └──────────────────────────────────────────────────┘
```

### 3.4 Playback path — direct play vs transcode

```
  Client requests track
        │
        ▼
  ┌─────────────────────────────────────────┐
  │  Can the client play this codec at      │
  │  this bitrate? (client declares caps)   │
  └──────┬───────────────────────┬──────────┘
         │ YES                   │ NO — or user set a bitrate cap
         ▼                       ▼
   DIRECT PLAY            ┌──────────────────────────┐
   Raw file bytes         │  ffmpeg transcode        │
   streamed as-is.        │  FLAC ~1000 kbps         │
   Zero CPU.              │     → Opus 128 kbps      │
   Use on Wi-Fi.          │  Streamed, not cached.   │
                          │  ~1–2% CPU per stream.   │
                          └──────────────────────────┘
                             Use on mobile data.
                             ~8x bandwidth saving.
```

Audio transcoding is cheap — a laptop CPU handles a dozen concurrent Opus streams without strain. Video transcoding is the expensive case, which is why Jellyfin is optional and gated on hardware encode.

---


## 4. Data Model

Both servers derive their schema from file tags. You do not design this — you conform to it.

### Music (Navidrome)

```
artist ───────┐
              ├── album ────── track ────── media_file
album_artist ─┘                 │
                                ├── mbid (stable join key)
                                └── annotation (PER USER)
                                     ├── starred
                                     ├── rating
                                     ├── play_count
                                     └── last_played
```

**The one modelling trap:** `artist` and `album_artist` are distinct fields. On compilations every track has a different `artist` but a shared `album_artist`. If beets does not write `ALBUMARTIST`, Navidrome shatters one compilation into forty single-track albums. Verify this field is populated after import.

### Audiobooks (Audiobookshelf)

```
library_item (book)
  ├── metadata: title, author, narrator, series, seriesSequence, isbn
  ├── audio_file[]     (usually one m4b)
  ├── chapter[]        (title, start, end — from m4b atoms)
  └── media_progress   (PER USER)
        ├── currentTime (float seconds)
        ├── isFinished
        └── lastUpdate (epoch ms) ◄── conflict resolution key
```

`media_progress` makes or breaks the product. Cross-device resume compares `lastUpdate` timestamps, so the laptop's clock must be correct — `systemd-timesyncd` handles this by default, but confirm it is running.

### Video (Jellyfin, only if added later)

```
movie ── version/edition ── media_file ── stream[]
                                           ├── video (codec, resolution, HDR)
                                           ├── audio (codec, channels, language)
                                           └── subtitle (language, forced)
```

---

## 5. Operations, Limits, Next Steps

### Resource footprint

| Service | Idle RAM |
|---|---|
| Navidrome | ~50 MB (spikes during scan) |
| Audiobookshelf | ~200–300 MB (Node baseline) |
| Watchtower | ~10 MB |
| **Total** | **under 500 MB** |

The binding constraint on old hardware is **disk I/O during the initial scan**, not RAM or CPU. An external drive on USB 2.0 makes the first scan slow and has no effect on playback afterwards.

### Known limitations

- **Single drive, single point of failure.** Phase 5 covers metadata, not media. A second drive with a weekly `rsync` is the cheap insurance.
- **No discovery.** Stated non-goal. Navidrome offers smart playlists and similar-artist radio via Last.fm, nothing resembling Spotify's recommendation engine.
- **Two apps, not one.** Music and audiobooks live in separate clients. Symfonium bridges Navidrome and Jellyfin but not Audiobookshelf. Unifying them means building a gateway — deliberately out of scope.
- **Tailscale is a dependency.** Its coordination server is required for key exchange, never for media traffic. Self-hosted Headscale removes this if it ever matters.

### Where this goes next

The subscription savings were never the real return. The higher-value extension is treating this stack as cloud infrastructure practice that maps onto portfolio work:

1. **Codify it** — Ansible for the host setup, so a rebuild is one command not an afternoon.
2. **Observe it** — Prometheus + Grafana; graph transcode sessions, stream counts, disk usage, container health.
3. **Automate it** — GitHub Actions to lint the compose file, plus a scheduled job that runs a restore test against the backup and alerts on failure.
4. **Harden it** — fail2ban, unattended-upgrades, a documented recovery runbook.

That produces a legible "I operate production infrastructure" story. A hand-rolled music player does not.
