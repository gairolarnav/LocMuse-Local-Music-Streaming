# Self-Hosted Media Server — Build Plan

**Hardware:** Old laptop (24/7) + external hard drive with existing music library
**Status:** Post-review. Phase order corrected, security model made enforceable, backup scope inverted to match the governing principle.

> **What changed from rev. 1 and why it matters**
>
> Rev. 1 ran an irreversible mass tag rewrite (Phase 2) three phases before any backup existed (Phase 5), on a single unreplicated drive. That ordering is now reversed. The security claim "nothing listens on a public port" was asserted but not enforced — Docker bypasses UFW, so it needed a binding strategy. The backup targeted the databases while leaving the files (the stated source of truth) unprotected. Watchtower has been archived upstream and no longer runs on current Docker engines. ReplayGain was sold as a client feature but absent from the tag pipeline.

---

## 1. The Plan

### What this is

A private, always-on server streaming your own music and audiobooks to phone and browser — album art, artist metadata, chapter navigation, offline downloads, playback progress synced across devices. No subscription, no tracking, no third party holding your library.

### What was deliberately cut, and why

| Cut | Reason |
|---|---|
| Custom server | Navidrome and Audiobookshelf already solve this. Rebuilding filesystem watching, tag parsing, MusicBrainz matching, transcode session management and HLS segmenting is 6–12 months to reach parity with zero novel outcome. |
| Custom mobile app | The hard parts of a music client are gapless playback, crossfade, ReplayGain, offline sync conflict resolution, and Android Auto. Symfonium already does all of it. This is where DIY projects die. |
| CD / Blu-ray ripping pipeline | No discs, no optical drive. Blu-ray additionally requires circumventing AACS, legally exposed under India's Copyright Act §65A. Your music files already exist — ingestion is a non-problem. |
| Reverse proxy + domain + public TLS | `tailscale serve` provides a real `*.ts.net` certificate with no Caddy, Let's Encrypt, DDNS or port forwarding. |
| Public internet exposure | Nothing listens on a routable interface. The attack surface is WireGuard, not service auth — **provided the port binding in §3.2 is followed.** |
| Ansible / Prometheus / CI | Deferred, not cancelled. See §6. Version control is *not* deferred — see Phase 0. |

### The one place not to be lazy

**Tag quality.** Navidrome treats embedded file tags as the source of truth and does no fuzzy matching. A library with inconsistent artist names, missing `ALBUMARTIST` fields and absent cover art renders as duplicate artists and blank tiles. One `beets` pass against MusicBrainz fixes this permanently. This is the difference between a Plexamp-grade experience and something abandoned in three weeks.

### Governing principle: tags are truth, the database is a cache

Neither server stores anything authoritative about *what the media is* — that lives in the files. Your library stays portable to any tool, forever.

**Caveat that rev. 1 got wrong:** the database is not purely disposable. Play counts, ratings, starred tracks and **playlists** exist only there. "Delete the DB and lose nothing" is false. The principle still governs backup *priority* — files first, database second — but the database is load-bearing and gets a real backup (§5.2).

### Revised phase plan

Ordering rule: **nothing irreversible happens before a verified redundant copy exists.**

| Phase | Work | Time | Exit criterion |
|---|---|---|---|
| 0 | Host prep — OS, power, network, git repo, filesystem decision | 60 min | `lsblk -f` output recorded; laptop survives lid-close + reboot headless |
| 1 | Copy library to ext4; second-drive rsync; restic + **verified restore** | 2–4 h (mostly copy time) | A file restored from B2 and byte-compared against the original |
| 2 | Docker stack on the *untagged* library | 30 min | `docker exec navidrome ls /music` lists files; web UI plays a track |
| 3 | Tailscale + `tailscale serve` + client validation | 30 min | Track plays on mobile data with Wi-Fi off; PWA installs |
| 4 | beets interactive import | Multiple sessions | `album_artist IS NULL` count is zero |
| 4b | ReplayGain batch (unattended, overnight) | Hours of CPU | Album *and* track gain present on a sampled set |
| 5 | Diun notifications; restore drill; dead-man's switch | 45 min | A deliberate test failure produces an alert you actually receive |

Phases 2 and 3 come *before* tagging deliberately: you see the real tag problems in the UI rather than guessing, and you validate the hardware under scan load before committing hours of manual attention to it.

---

## 2. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| **Host OS** | Ubuntu Server 26.04.1 LTS | Newer kernel helps old-laptop power management and USB storage handling; fresh 5-year window. 24.04 is also fine and supported to 2029 — don't reinstall just for this. |
| **Runtime** | Docker + Compose | Whole stack in one declarative file. Rebuild from scratch in 10 minutes. |
| **Filesystem** | ext4 | Journaled, native Unix ownership. **NTFS and exFAT are both disqualifying** — see Phase 0. |
| **Music server** | Navidrome (Go) | Idles ~50 MB RAM, single binary, embedded SQLite. Natively speaks the Subsonic API — a 15-year-old standard with 20+ mature clients. |
| **Audiobook server** | Audiobookshelf (Node/Vue) | Per-user progress sync, chapter navigation, m4b merging, Audnexus metadata, official iOS/Android apps. |
| **Video (optional)** | Jellyfin | Only if movies get added later. Mounts the same music folder read-only alongside Navidrome with no conflict. |
| **Metadata DB** | SQLite, embedded per service | Zero admin. Postgres is pure overhead at single-user scale. Backed up via `VACUUM INTO`, never by copying the live file. |
| **Metadata sources** | MusicBrainz, Cover Art Archive, Audnexus | Free, open, no API keys for basic use. |
| **Tag pipeline** | beets + `fetchart`, `embedart`, `scrub`, `replaygain` | Writes canonical tags into the files. `replaygain` uses the **rsgain** backend. |
| **Loudness** | rsgain (EBU R128 / ReplayGain 2.0) | Album *and* track gain. Non-destructive — tags only, audio untouched. |
| **Transcoding** | ffmpeg (bundled in both images) | Nothing to configure. |
| **Search** | Built-in SQLite FTS | Meilisearch is unjustified below ~100k tracks. |
| **Auth** | Per-service local accounts | No OAuth, no external IdP, no account with anyone. |
| **Remote access** | Tailscale (WireGuard) + `tailscale serve` | Private mesh *and* valid TLS. The TLS half is required for PWA install — see §3.2. |
| **Android client** | Symfonium (~₹500 one-time) | Gapless, crossfade, ReplayGain, offline sync, Android Auto. Free alternatives: Tempo, substreamer. |
| **iOS client** | Amperfy | Actively maintained. play:Sub has gone dormant — don't build around it. |
| **Audiobook client** | Official Audiobookshelf app | Offline downloads, sleep timer, speed control, chapter jump. |
| **Web client** | Navidrome + ABS web UIs (PWA) | Installable **only over HTTPS** — hence `tailscale serve`. |
| **Update notification** | Diun | Watches image registries and notifies. Does not touch containers. Watchtower was archived 17 Dec 2025 and pins Docker API v1.25, which Engine v27+ rejects outright. |
| **Backups** | restic → Backblaze B2 (config + DBs) · rsync → second drive (media) | Two tiers, two failure domains. Media is the irreplaceable half. |
| **Alerting** | healthchecks.io dead-man's switch | Ten minutes of work. Catches drive dropout, dead container, silent backup failure. |

### On Diun vs. auto-updates — make this decision consciously

Diun does strictly *less* than Watchtower: it notifies, you act. That's correct here because Navidrome and Audiobookshelf both run **one-way schema migrations on startup**. An unattended bad pull migrates your live DB before you know it happened; rolling back the image doesn't roll back the schema, and the old binary can no longer read its own database. Diun also needs the Docker socket only read-only (or not at all with a static image list), versus Watchtower's root-equivalent read-write access.

The cost is real: notify-only fails if you ignore notifications for eight months. **If you won't commit to a monthly patch window, don't use Diun** — use the maintained fork `nicholas-fedor/watchtower` and accept the migration risk. The wrong answer is archived Watchtower, which simply won't start.

---

## 3. Architecture

### 3.1 System topology

```
┌─────────────────────────────────────────────────────────────────┐
│  OLD LAPTOP — Ubuntu Server 26.04 LTS, headless, lid ignored,   │
│  wired ethernet, battery retained as UPS, unattended-upgrades   │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           Docker Engine (log rotation capped)             │  │
│  │                                                           │  │
│  │  ┌──────────────┐  ┌──────────────────┐  ┌─────────────┐  │  │
│  │  │  Navidrome   │  │  Audiobookshelf  │  │    Diun     │  │  │
│  │  │  (Go)        │  │  (Node / Vue)    │  │             │  │  │
│  │  │              │  │                  │  │ notify only │  │  │
│  │  │  Subsonic    │  │  REST API + PWA  │  │ socket :ro  │  │  │
│  │  │  API + Web   │  │  Audnexus client │  └─────────────┘  │  │
│  │  │  SQLite      │  │  SQLite          │                   │  │
│  │  │  ffmpeg      │  │  ffmpeg / tone   │                   │  │
│  │  └──────┬───────┘  └────────┬─────────┘                   │  │
│  │         │                   │                             │  │
│  │  bound 127.0.0.1:4533  bound 127.0.0.1:13378              │  │
│  │  ── NOT 0.0.0.0. See 3.2. ──                              │  │
│  └─────────┼───────────────────┼─────────────────────────────┘  │
│            │                   │                                │
│  ┌─────────▼───────────────────▼─────────────────────────────┐  │
│  │   EXTERNAL HARD DRIVE — ext4, fstab by UUID,              │  │
│  │   nofail + x-systemd.device-timeout, USB autosuspend off  │  │
│  │                                                           │  │
│  │   /mnt/media/.mounted          sentinel file              │  │
│  │   /mnt/media/music/       :ro  FLAC + MP3, beets-tagged   │  │
│  │   /mnt/media/playlists/   :rw  Navidrome .m3u export      │  │
│  │   /mnt/media/audiobooks/  :rw  m4b, embedded chapters     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Tailscale daemon + tailscale serve (TLS termination)     │  │
│  │  https://mediabox.<tailnet>.ts.net                        │  │
│  └────────────────────────────┬──────────────────────────────┘  │
└───────────────────────────────┼─────────────────────────────────┘
                                │  encrypted mesh, no open ports
        ┌───────────────────────┼───────────────────────┐
   ┌────▼─────┐          ┌──────▼──────┐        ┌───────▼──────┐
   │ Android  │          │  MacBook    │        │  Desktop PC  │
   │ Symfonium│          │  Browser    │        │  Browser     │
   │ + ABS app│          │  (PWA)      │        │  (PWA)       │
   └──────────┘          └─────────────┘        └──────────────┘
```

### 3.2 The security model, and how it is actually enforced

Rev. 1 claimed "nothing listens on a public port" without a mechanism. **UFW does not constrain Docker.** Docker inserts its own `DOCKER` chain into iptables which is evaluated *before* UFW's rules, so `ports: "4533:4533"` binds `0.0.0.0` and is reachable from your entire LAN regardless of firewall configuration.

Enforcement is the port binding, not the firewall:

```yaml
ports:
  - "127.0.0.1:4533:4533"     # loopback only
```

then expose to the tailnet:

```bash
tailscale serve --bg --https=443 http://127.0.0.1:4533
tailscale serve --bg --https=8443 http://127.0.0.1:13378
```

This does three jobs at once:

1. **Nothing binds a routable interface.** The claim in §1 becomes true.
2. **Real TLS.** `tailscale serve` provisions a genuine `*.ts.net` certificate.
3. **PWAs actually install.** Service workers require a secure context. `http://100.x.y.z:4533` is not one — so on the rev. 1 design there is no install prompt, no offline shell, and the "both are PWAs" claim is false. HTTPS fixes it.

Also lock the tailnet down with an ACL so only your own devices reach the box; don't rely on tailnet membership alone if you ever share a node.

### 3.3 Access path — phone on mobile data reaching the server

```
  Phone (4G/5G, Symfonium)
        │
        │  1. MagicDNS resolves mediabox.<tailnet>.ts.net → 100.x.y.z
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
  tailscale serve :443 (TLS)  →  127.0.0.1:4533  →  Navidrome
```

Your ISP's CGNAT does not need to cooperate, no dynamic DNS is required, no service is reachable by internet scanners, and a Navidrome auth bug cannot be exploited by anyone outside your tailnet.

### 3.4 Ingestion and metadata flow

Two passes, deliberately separated. The first is bounded by *your attention* and MusicBrainz rate limits; the second is bounded by CPU and runs unattended.

```
  Existing files on ext4
  (mixed tags, missing art)
        │
        ▼
  ┌──────────────────────────────────────────────────┐
  │  PASS 1 — beets import  (interactive, sessions)  │
  │   ├─ metadata match ─────────► MusicBrainz API   │
  │   │                            (~1 req/sec)      │
  │   ├─ fetchart ───────────────► Cover Art Archive │
  │   ├─ embedart                                    │
  │   └─ writes tags + art BACK INTO the files       │
  └──────────────────┬───────────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────────┐
  │  PASS 2 — beet replaygain  (unattended, batch)   │
  │   rsgain decodes every file, computes R128       │
  │   loudness, writes ALBUM gain + TRACK gain +     │
  │   peak. Audio data untouched.                    │
  └──────────────────┬───────────────────────────────┘
                     │  clean, self-describing files
                     ▼
     /mnt/media/music/Artist/Album (Year)/01 Track.flac
                     │  filesystem watch + 6h scheduled scan
                     ▼
  ┌──────────────────────────────────────────────────┐
  │  Navidrome scanner                               │
  │   reads embedded tags → builds SQLite index      │
  └──────────────────────────────────────────────────┘
```

**Why the split.** Running ReplayGain inside the import means you sit idle waiting for CPU-bound decode between album decisions. Separated, pass 2 runs overnight. The cost of *deferring it past the library-surgery window* is that it becomes a second full-library rewrite against files that are live and backed up — so do it now, just not simultaneously.

**Track gain vs. album gain.** Track gain normalises each file independently, which flattens deliberate dynamics within a record — quiet interludes get shoved up to match the single. Album gain applies one offset across the album, preserving the mastering engineer's intent. Write both; good clients pick per playback mode (album gain for sequential listening, track gain for shuffle).

**Tag storage gotcha.** FLAC uses Vorbis comments (`REPLAYGAIN_ALBUM_GAIN`); MP3 can use ID3v2 `TXXX` or APEv2, and different tools historically chose differently. A library tagged by three tools over ten years has gain values clients can't find. Routing through beets normalises this — a real argument for `beet replaygain` over standalone rsgain.

### 3.5 Playback path

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
                          │  ~5–10% of a core on     │
                          │  old silicon, per stream │
                          └──────────────────────────┘
```

Realistically this path barely matters for a single user: Symfonium's offline sync means most mobile listening is from local cache, not a live stream. Rev. 1's "a dozen concurrent Opus streams" was solving a problem you don't have. Video transcoding is the genuinely expensive case, which is why Jellyfin stays optional and gated on hardware encode.

---

## 4. Data Model

Both servers derive their schema from file tags. You do not design this — you conform to it.

### Music (Navidrome)

```
artist ───────┐
              ├── album ────── track ────── media_file
album_artist ─┘                 │
                                ├── mbid (stable join key)
                                ├── replaygain_album_gain / _track_gain
                                └── annotation (PER USER, DB-only)
                                     ├── starred
                                     ├── rating
                                     ├── play_count
                                     └── last_played
```

**Modelling trap 1:** `artist` and `album_artist` are distinct. On compilations every track has a different `artist` but a shared `album_artist`. If beets does not write `ALBUMARTIST`, Navidrome shatters one compilation into forty single-track albums. Verify after import.

**Modelling trap 2:** `annotation` and playlists live *only* in SQLite. They have no file representation. This is why §5.2 exists.

### Playlists — the read-only mount conflict

Rev. 1 mounted music read-only *and* called the database disposable. Those are incompatible: Navidrome's escape hatch for playlist portability is exporting `.m3u` files into the music tree, which a read-only mount forbids. So playlists would exist nowhere but a DB you'd declared throwaway.

Resolution: a separate writable `/mnt/media/playlists/` mount. Music stays `:ro`; playlists round-trip to disk as files, consistent with the governing principle.

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

`media_progress` makes or breaks the product. Cross-device resume compares `lastUpdate`, so the laptop clock must be correct — `systemd-timesyncd` handles this by default; confirm it's running.

**Warning:** ABS needs a read-write mount for m4b merging, which means **ABS can destroy media**. Its "embed metadata" tool rewrites files in place. Don't run it against unbacked-up files.

---

## 5. Operations

### 5.1 Host prerequisites (Phase 0 detail)

**Filesystem — answer this first.** Run `lsblk -f`. If the drive came off Windows it's NTFS or exFAT and both are disqualifying: NTFS via ntfs-3g is FUSE, single-threaded and CPU-heavy (beets rewriting tens of thousands of files through it is brutal); exFAT has no journaling, so an unclean shutdown corrupts the directory table; and neither carries Unix ownership, so container UIDs can't work without `uid`/`gid` mount options. Copy to ext4 before anything else.

**fstab.** Three separate failure modes, all headless:

```
UUID=<uuid>  /mnt/media  ext4  defaults,nofail,x-systemd.device-timeout=10  0  2
```

- Without `nofail`, a missing drive drops you to an emergency shell on a machine with no monitor.
- If the mount silently fails but Docker starts anyway, bind mounts create an *empty* `/mnt/media`, both scanners see a vanished library and mark everything missing. Guard with a sentinel file `/mnt/media/.mounted`, `RequiresMountsFor=/mnt/media` on the compose unit, and a healthcheck that greps for it.
- USB autosuspend on external enclosures drops the device and leaves a stale mount returning I/O errors until reboot. Disable autosuspend via udev for that device; check whether the enclosure needs the UAS quirk flag.

**Permissions — the most common failure mode in containerised media stacks, and absent from rev. 1 entirely.**

- Both images run as configurable `PUID`/`PGID`. Wrong UID means Navidrome scans an empty library and reports nothing wrong.
- beets runs as *you*, on the host. Every rewritten file gets your ownership and umask. A `077` umask silently strips container read access mid-import. Set `umask 002` for the beets run.
- Decide UID/GID before Phase 1. Create a shared media group. ABS's UID needs write on `audiobooks/` and `playlists/`; Navidrome needs read on `music/` only.

**Power and network.**
- Ignore lid close (`HandleLidSwitch=ignore` in logind.conf).
- Check battery health. A swollen cell on a 24/7 box is a hazard — but a *healthy* battery is a free UPS and protects against exactly the unclean-shutdown corruption described above. Configure clean shutdown at low charge rather than removing it.
- Use wired ethernet. Linux Wi-Fi power saving drops the NIC on an idle headless box; if you must use Wi-Fi, `iw dev wlan0 set power_save off`.
- Enable `unattended-upgrades` now, not in a "harden it later" phase. Five minutes.
- Confirm swap exists. ABS spikes hard during m4b merging; 4 GB with no swap is an OOM kill mid-merge.

**Docker log rotation.** `json-file` is unbounded by default and will eat the root disk. In `/etc/docker/daemon.json`:

```json
{ "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" } }
```

**Version control from day one.** `git init` a config repo in Phase 0 containing the compose file, `.env.example`, fstab entries, udev rules and scripts, with secrets excluded. Retrofitting this after two months of hand-edited config is most of the work of doing it right the first time. Ansible can wait; version control can't.

### 5.2 Backups — two tiers

Rev. 1 inverted this: it backed up the *cache* to B2 and left the *truth* on one unreplicated drive, with media redundancy demoted to a "known limitation."

**Tier 1 — media (irreplaceable).** Second drive, weekly `rsync`. This is Phase 1, before beets touches anything. A raw pre-tagging copy is your only undo for a bad mass rewrite.

**Tier 2 — databases and config (small, but load-bearing).** restic → B2, daily.

**The SQLite problem.** Never let restic read a live database. Navidrome runs SQLite in WAL mode — the DB is three files (`.db`, `.db-wal`, `.db-shm`), and committed transactions can exist *only* in the WAL. Two ways this corrupts silently:

1. restic reads `.db` at 03:00:12 and `.db-wal` at 03:00:14. A checkpoint in between folds the WAL into the main file and truncates it. Your snapshot pairs a pre-checkpoint main file with a post-truncation WAL. Those transactions are in neither.
2. Reading a large file isn't atomic. Page 40 read at T0 and page 9000 at T3, with writes in between, gives a b-tree that doesn't match itself: `database disk image is malformed`.

restic exits 0 in both cases. You find out on the day the drive dies.

**The fix — snapshot with `VACUUM INTO`, then back up the snapshot:**

```bash
rm -f /var/backups/media-staging/navidrome.db     # VACUUM INTO won't overwrite
sqlite3 /srv/.../navidrome.db \
  "VACUUM INTO '/var/backups/media-staging/navidrome.db'"
sqlite3 /var/backups/media-staging/navidrome.db "PRAGMA integrity_check;"
```

`VACUUM INTO` runs in a read transaction, so it sees one coherent point-in-time view regardless of writers. Output is a single self-contained file in a directory nothing else touches — both failure modes become structurally impossible. Verify with `integrity_check` and abort loudly on anything but `ok`.

Operational notes:
- Run `sqlite3` on the host against bind-mounted paths. Don't assume the CLI exists inside the images.
- Even though it only reads, SQLite may need to touch `-shm`, so it wants write access to the source directory. Run as root via systemd timer to sidestep the UID collision.
- `chmod 700` the staging dir — it briefly holds the whole DB in the clear. Put it on the system disk, not the media drive.
- Exclude `*-wal` / `*-shm` from the restic run.
- `VACUUM` defragments, which reorders pages, so restic dedupes vacuumed output *worse* than raw files — each run stores close to a full copy. At tens of MB this is irrelevant; know it so repo growth doesn't surprise you.

Schedule with a systemd timer, not cron, and include `RequiresMountsFor=/mnt/media` so it refuses to run against an unmounted drive rather than backing up an empty directory and pruning good snapshots out of retention.

**Restore drill.** The script is a hypothesis until you've restored from it once and compared row counts against the running instance. An untested backup is a belief.

### 5.3 Alerting

A dead-man's switch is ten minutes and non-optional on a box you never look at: a systemd timer that pings healthchecks.io **only if** the mount sentinel exists, containers are up, and the last restic run exited 0. Without it you discover the drive unmounted three weeks ago when you try to play something.

Prometheus + Grafana stays deferred — that's observability, this is liveness. Different problems.

### 5.4 Update procedure

Diun notifies. You then, monthly:

1. Read the release notes for schema migrations.
2. Run the backup script manually — this is the rollback point.
3. `docker compose pull && docker compose up -d` with **pinned major tags**, not `:latest`.
4. Verify playback and scan.

### 5.5 Resource footprint

| Service | Idle RAM |
|---|---|
| Navidrome | ~50 MB (spikes during scan) |
| Audiobookshelf | ~200–300 MB idle; spikes hard during m4b merge |
| Diun | ~15 MB |
| **Total** | **under 500 MB idle** |

The binding constraint on old hardware is **disk I/O during the initial scan**, not RAM or CPU. USB 2.0 makes the first scan slow and has no effect on playback afterwards.

### 5.6 Realistic timings

Rev. 1's "~1 evening for tag cleanup" had no library size attached and is wrong for anything substantial. MusicBrainz rate-limits unauthenticated clients to roughly 1 req/sec, and interactive import needs a human decision per album at 10–20 seconds each. At 1,500 albums that's six-plus hours of *your attention*. Under ~300 albums, one evening is fine. Above that, plan multiple sessions, or run `--quiet` with a confidence threshold and manually review only the rejects. Always `beet import -t` (pretend mode) on a sample first, and know how to resume an interrupted import before you start one.

`scrub` strips all existing tags and relies on beets rewriting afterwards — run it standalone or with `-W` and you lose embedded art.

---

## 6. Limits and Next Steps

### Known limitations

- **No discovery.** Stated non-goal. Smart playlists and Last.fm similar-artist radio, nothing resembling Spotify's recommender.
- **Two apps, not one.** Symfonium bridges Navidrome and Jellyfin but not Audiobookshelf. Unifying means building a gateway — deliberately out of scope.
- **Tailscale is a dependency.** Its coordination server is required for key exchange, never for media traffic. Headscale removes this if it ever matters.
- **Single-site backup.** B2 covers config and DBs; media redundancy is a second drive in the same room. Fire or theft takes both. Accepted for now — offsite media backup is a cost decision, not a technical one.

### Where this goes next

The subscription savings were never the return. The value is that this maps onto portfolio infrastructure work:

1. **Codify it** — Ansible for host setup, so a rebuild is one command. Cheap *because* Phase 0 put the configs in git.
2. **Observe it** — Prometheus + Grafana; stream counts, transcode sessions, disk usage, container health.
3. **Automate it** — CI to lint the compose file, plus a scheduled restore test that alerts on failure.
4. **Harden it** — fail2ban, tailnet ACLs, a written recovery runbook.

That produces a legible "I operate production infrastructure" story. A hand-rolled music player does not.

### A note on this document

It has now survived two rounds of adversarial review, and the marginal return on a third is lower than the return on running Phase 0 tonight. Everything remaining is configuration detail discoverable in the first hour of building. A thorough plan starts to feel like progress; it isn't — it's a hypothesis about a machine you haven't touched. Build Phases 0–3, then revise this against what actually happened.
