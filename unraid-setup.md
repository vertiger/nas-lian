# Unraid NAS Setup & Tuning

Server: `root@192.168.86.20` (nas-lian)
Hardware: Intel i5-13600K (6P+4E, 20 threads), 64 GB RAM, no swap
Storage: ZFS `cache_nvme` pool (5x NVMe RAIDZ1, 4.55 TB, 22% used), 24x HDD array (375 TB, XFS, **88% used — monitor**)
OS: Unraid **7.2.4**, kernel 6.12.54, ZFS 2.3.4
API: GraphQL at `/graphql`, local API key active, Unraid Connect plugin v4.10.0
BIOS: All NVMe PEG ports set to **PCIe Gen3** (changed 2026-04-17, fixed RxErr on SN700)
Last reboot: 2026-04-17 (manual, for EFI PCIe Gen3 change)

---

# PostgreSQL

## PG17 — Arr Stack (port 5432)

Container: `postgresql17` (postgres:17), config at `/mnt/user/appdata/postgresql17/`
ZFS dataset: `cache_nvme/pg17data` (8K recordsize, lz4, primarycache=metadata)
Databases: 17 total (~7.6 GB after fresh restore) — Radarr (4×2), Sonarr (2×2), Lidarr (2), Prowlarr (2)
CPU pinning: cores 12-19 | Extra: `--shm-size=256m`

Unraid template: `/boot/config/plugins/dockerMan/templates-user/my-postgresql17.xml`
`prowlarr-user` granted SUPERUSER for housekeeping/VACUUM.

### Current Configuration

| Setting | Value | Notes |
|---|---|---|
| `shared_buffers` | 4 GB | 99.6%+ cache hit — working set fits easily |
| `effective_cache_size` | 8 GB | ZFS `primarycache=metadata` — ARC caches zero PG data blocks |
| `work_mem` | 32 MB | Worst case: 29 × 32MB × 2 = 1.8 GB |
| `maintenance_work_mem` | 2 GB | PG17 removed silent 1 GB cap on VACUUM TID store |
| `random_page_cost` | 1.1 | NVMe |
| `effective_io_concurrency` | 200 | NVMe |
| `maintenance_io_concurrency` | 200 | NVMe — VACUUM, CREATE INDEX |
| `max_connections` | 50 | ~27 in use |
| `shared_preload_libraries` | pg_stat_statements | |
| `wal_compression` | lz4 | PG15+ — faster than pglz |
| `checkpoint_timeout` | 15 min | |
| `max_wal_size` | 2 GB | 92% timed checkpoints |
| `min_wal_size` | 512 MB | Pre-allocated WAL segments |
| `bgwriter_lru_maxpages` | 1000 | |
| `bgwriter_lru_multiplier` | 4.0 | |
| `bgwriter_delay` | 50 ms | |
| `track_io_timing` | on | |
| `track_wal_io_timing` | on | PG17 — feeds `pg_stat_io` |
| `compute_query_id` | on | Always available in `pg_stat_statements` |
| `log_min_duration_statement` | 1000 ms | |
| `log_checkpoints` | on | |
| `huge_pages` | off | |
| `default_toast_compression` | lz4 | |
| `full_page_writes` | off | ZFS COW prevents torn pages |
| `wal_init_zero` | off | ZFS handles allocation |
| `wal_recycle` | off | ZFS handles space management |

Autovacuum aggressive. Cache hit rate: 95–99.9% across all main DBs.

## PG Immich (port 5433)

Container: `PostgreSQL_Immich` (tensorchord/pgvecto-rs:pg16-v0.3.0)
Config: `/mnt/user/appdata/PostgreSQL_Immich/postgresql.conf`
Database: `immich` — 223 MB, 62 tables, 8,521 assets, pgvecto.rs vector search

### Current Configuration

`shared_buffers=512MB`, `effective_cache_size=2GB`, `work_mem=64MB`, `maintenance_work_mem=512MB`, `random_page_cost=1.1`, `effective_io_concurrency=200`, `max_connections=30`, `wal_compression=on`, `checkpoint_timeout=15min`, `max_wal_size=2GB`, `track_io_timing=on`, `log_min_duration_statement=1000`, `log_checkpoints=on`, `default_toast_compression=lz4`, `huge_pages=off`, `full_page_writes=off`, `wal_init_zero=off`, `wal_recycle=off`

Container memory: 54 MiB. Cache hit: 98.5%, index hit: 99.8%. `geodata_places` (123 MB) has 62 MB of indexes — don't drop, used during reverse geocoding import.

### Vector Search (pgvecto.rs)

- HNSW indexes (`clip_index`, `face_index`): valid, actively used (80K+ idx_scans)
- Index size shows 0 bytes in pg_relation_size — normal (pgvecto.rs uses external mmap storage)
- 7,752 CLIP embeddings, 8,629 face embeddings
- Indexes reindexed after shared_buffers increase

## Monitoring

```sql
-- Top slow queries
SELECT left(query,100), calls, round(total_exec_time::numeric/1000,1) as total_sec,
  round(mean_exec_time::numeric,1) as avg_ms FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;

-- Cache hit ratio
SELECT datname, round(100.0*blks_hit/NULLIF(blks_read+blks_hit,0),2) as pct FROM pg_stat_database WHERE datname NOT LIKE 'template%';

-- Checkpoint stats
SELECT * FROM pg_stat_bgwriter;
```

## Future

- pgvecto.rs v0.3.0 → check for newer versions (0.4.x+ has performance improvements)
- PG Immich: consider upgrading to PG16 (already on PG16, but pgvecto.rs image may lag)

---

# ZFS (cache_nvme pool)

## Pool Status

| Property | Value |
|---|---|
| Layout | 5x NVMe RAIDZ1 |
| Size | 4.55 TB (22% used) |
| Fragmentation | 30% |
| ashift | 12 (4K sectors) |
| Autotrim | on |
| Compression | lz4 (pool-wide) |
| Atime | off |
| ARC max | 7.8 GB (Unraid default: RAM/8) |
| Docker storage | ZFS driver, 806+ layer datasets |
| PCIe | All slots set to **Gen3** in EFI (2026-04-17) |

### NVMe Drive Inventory

| PCI BDF | NVMe | Model | Size | Wear % | Hours | Notes |
|---|---|---|---|---|---|---|
| 01:00.0 | nvme1 | WD Red SN700 | 1 TB | 11% | 11,396 | Had 2,952 PCIe RxErr pre-Gen3 fix |
| 02:00.0 | nvme2 | Samsung 960 EVO | 1 TB | 47% | 16,028 | |
| 03:00.0 | nvme0 | WDC WDS100T2B0C (SN550) | 1 TB | **65%** | **40,413** | **Plan replacement** within 1-2 years |
| 08:00.0 | nvme3 | WD Red SN700 | 1 TB | 11% | 11,569 | |
| 0a:00.0 | nvme4 | WD SN560 | 1 TB | 16% | 6,033 | Gen4 drive at Gen3 (capped). Controller sensor 81°C — monitor |

## Datasets

| Dataset | Recordsize | Purpose | Notes |
|---|---|---|---|
| `cache_nvme` (root) | 128K | Appdata, system, docker layers | Default |
| `cache_nvme/pg17data` | **8K** | PostgreSQL 17 data | |
| `cache_nvme/pgImmichData` | **8K** | Immich PostgreSQL data | |
| `cache_nvme/media` | **1M** | Unified media share (TRaSH structure) | |
| `cache_nvme/data` | 128K | Gaming/emulation data | Auto-created by Unraid |

## Tuning

- Async read/write queue depth: 32 (persisted in `/boot/config/go`)
- PG datasets: `recordsize=8K`, `compression=lz4`, `atime=off`, `logbias=throughput`, `primarycache=metadata`
- Custom mountpoints auto-mount on pool import (survives reboots)

### Prefetch — KEEP ENABLED

Prefetch has 4.3% hit rate (poor for PG, good for video streaming). `zfs_prefetch_disable` is global — can't be per-dataset. Solution: `primarycache=metadata` on PG datasets (done) keeps PG data out of ARC. Prefetch remains useful for video/media.

### Dataset Auto-Creation

Unraid 6.12+ auto-creates ZFS datasets per share when configured via WebGUI. Direct filesystem writes create plain directories. Mover empties datasets but doesn't destroy them.

---

# HDD Array

| Property | Value |
|---|---|
| Disks | 24 HDDs (7.3–29.1 TB each), XFS |
| Parity | 1 disk (sdo, 29.1 TB) |
| Capacity | 375 TB (87% used) — monitor, XFS degrades >90% |
| Scheduler | bfq (set in go file) |
| Read-ahead | 32 MB |
| NCQ | 32 |
| SMART | 0 errors on all drives |

HDDs are well-configured for media streaming. No significant improvements available.

---

# NVMe SSD Firmware

| Drive | Model | FW | Status |
|---|---|---|---|
| nvme0 | WD Red SN700 1TB | 111150WD | Up to date |
| nvme1 | Samsung 960 EVO 1TB | 3B7QCXE7 | Final firmware (EOL) |
| nvme2 | WD Blue SN550 1TB | 211070WD | **CAUTION** — newer FW may be for different NAND |
| nvme3 | WD Red SN700 1TB | 111150WD | Up to date |
| nvme4 | WD PC SN560 1TB | 74114000 | Unknown — check via WD Dashboard |

---

# Boot Scripts (`/boot/config/go`)

- NVMe scheduler `none`, HDD scheduler `bfq` via udev
- `powertop --auto-tune` at boot
- Log partition remounted at 384 MB
- ZFS async read/write max active = 32 (`/sys/module/zfs/parameters/`)

---

# Plex Configuration & Tuning

**Container:** `linuxserver/plex` (host networking)
**Version:** 1.43.1.10611-1e34174b1
**CPU pinning:** cores 12-19 (E-cores only — keeps P-cores free for PG/Arr)
**GPU:** Intel UHD 770 (Raptor Lake-S), `/dev/dri` passed through (rwm), QSV hardware transcoding enabled
**SHM:** 256 MB
**Memory:** unlimited | **Restart:** unless-stopped
**Binds:** `/mnt/user/appdata/Plex:/config`, `/mnt/user/media/:/volume1`, `/mnt/user/media/temp/:/transcode`
**Inotify:** max_user_watches=524288, max_user_instances=128

## Libraries

| Library | Section | Type | Locations |
|---|---|---|---|
| Movies | 7 | movie | `/volume1/movies`, `/volume1/movies-ger`, `/volume1/movies-4k`, `/volume1/movies-manual` |
| Movies 3D | 24 | movie | `/volume1/movies-3d` |
| TV Shows | 1 | show | `/volume1/series`, `/volume1/series-ger` |
| Music | 9 | artist | `/volume1/music` |

**Items:** ~15,900 movies, ~23,966 TV episodes, ~202,967 music tracks, ~667 3D movies

## Key Settings (Preferences.xml)

| Setting | Value | Assessment |
|---|---|---|
| `HardwareAcceleratedCodecs` | 1 | ✅ QSV enabled |
| `TranscoderHEVCEncodingMode` | always | ✅ HEVC via QSV saves bandwidth |
| `TranscoderQuality` | 0 (Automatic) | ✅ |
| `BackgroundPreset` | faster | ✅ |
| `AllowHighOutputBitrates` | 1 | ✅ |
| `TranscodeCountLimit` | 6 | ✅ |
| `FSEventLibraryUpdatesEnabled` | 1 | ✅ inotify-based real-time updates |
| `FSEventLibraryPartialScanEnabled` | 1 | ✅ partial scan for efficiency |
| `ScannerLowPriority` | 1 | ✅ won't hog CPU |
| `ScheduledLibraryUpdateInterval` | 86400 (24h) | ✅ fine as safety net with FSEvent |
| `LanNetworksBandwidth` | 192.168.86.0/24 | ✅ LAN direct play |
| `secureConnections` | 1 | ✅ TLS for remote |
| `DlnaEnabled` | 0 | ✅ disabled, reduces attack surface |
| `IPNetworkType` | v4only | ✅ appropriate for home network |
| `GenerateBIFBehavior` | never | ✅ saves CPU/storage |
| `GenerateChapterThumbBehavior` | never | ✅ saves CPU/storage |
| `allowMediaDeletion` | 1 | ✅ intentional — delete from WebUI |
| `watchMusicSections` | 0 | ✅ intentional — daily scan sufficient for music |
| `ButlerTaskBackupDatabase` | 0 | ✅ intentional — covered by appdata backup |
| `ButlerStartHour` / `EndHour` | 12 / 16 | ✅ intentional — daytime = users at work, no playback impact |
| `collectUsageData` | 0 | ✅ |
| `CinemaTrailersType` | 0 (disabled) | ✅ |
| `ManualPortMappingPort` | 16151 | ✅ set but mode=auto |
| `WanTotalMaxUploadRate` | 200000 (kbps) | ✅ ~200 Mbps WAN cap |
| `GlobalMusicVideoPath` | `/volume1/music-videos` | ✅ |

## Butler Tasks

| Task | Enabled | Interval | Notes |
|---|---|---|---|
| BackupDatabase | ❌ no | 3 days | Intentional — covered by appdata backup |
| OptimizeDatabase | ✅ yes | 7 days | Good — keeps DB performant |
| CleanOldBundles | ✅ yes | 7 days | |
| CleanOldCacheFiles | ✅ yes | 7 days | |
| DeepMediaAnalysis | ✅ yes | 1 day | |
| LoudnessAnalysis | ✅ yes | 1 day | |
| IntroMarkers | ✅ yes | 1 day | |
| CreditsMarkers | ✅ yes | 1 day | |
| AdMarkers | ✅ yes | 1 day | |
| RefreshLibraries | ✅ yes | 1 day | |
| RefreshLocalMedia | ✅ yes | 3 days | |
| RefreshPeriodicMetadata | ✅ yes | 1 day | |
| GarbageCollectBlobs | ✅ yes | 7 days | |
| GenerateChapterThumbs | ❌ no | 1 day | Good — disabled, matches GenerateChapterThumbBehavior=never |
| GenerateMediaIndexFiles | ❌ no | 1 day | Good — BIF disabled |
| GenerateVoiceActivity | ❌ no | 1 day | |

**Butler window:** 12:00–16:00 (daytime) — intentional, users at work so no playback impact.

## Storage (appdata ~140 GB total)

| Component | Size | Notes |
|---|---|---|
| Metadata | 124 GB | Posters, art, thumbnails — largest by far |
| Plug-in Support | 9.4 GB | Databases + plugins |
| ├─ `library.db` | 1.8 GB | Main library DB |
| ├─ `library.blobs.db` | 1.8 GB | Blob storage |
| ├─ `library.db-wal` | 63 MB | WAL (active transactions) |
| ├─ `trakttv.db` | 75 MB | Trakt sync plugin |
| Media | 4.1 GB | Optimized versions / sync |
| Cache | 2.3 GB | Transcoder cache, analysis |
| Logs | 3.1 MB | |

## Plex + PostgreSQL

`cgnl/plex-postgresql` — LD_PRELOAD shim that intercepts SQLite calls and routes to PG. **Not recommended** (too young, low adoption, no SQLite issues observed on our setup). Revisit late 2026 if project stabilizes (v2.0+, 100+ stars). Watch `github.com/cgnl/plex-postgresql` releases.

---

# Media Folder Structure

Unified `/mnt/user/media/` share (TRaSH structure). Enables hardlinks, atomic moves, single Docker bind. `books/` is a separate share.

## Structure (`/mnt/user/media/`)

```
media/
├── movies/                ← Radarr (main)
├── movies-ger/            ← Radarr-ger
├── movies-3d/             ← Radarr-3D
├── movies-4k/             ← Radarr-4k
├── movies-manual/         ← Not managed by Arr
├── series/                ← Sonarr (main)
├── series-ger/            ← Sonarr-ger
├── series-3d/
├── music/                 ← Lidarr
├── music-videos/
├── other-videos/
├── recordings/
├── photo/
├── download/              ← SABnzbd + qBittorrent
│   ├── incomplete/        ← qBittorrent incomplete
│   ├── movies/            ← Radarr main
│   ├── movies-3d/         ← Radarr-3D
│   ├── movies-4k/         ← Radarr-4k
│   ├── movies-ger/        ← Radarr-ger
│   ├── series/            ← Sonarr main
│   ├── series-ger/        ← Sonarr-ger
│   ├── music/             ← Lidarr
│   └── temp/              ← SABnzbd incomplete
└── temp/                  ← Plex transcode

books/                     ← SEPARATE share at /mnt/user/books/ (not under media)
├── audiobooks/
├── books/
├── comics/
├── italian/
└── Rosetta Stone/
```

## Docker Path Mapping

All containers get ONE bind: `/mnt/user/media:/media`

| Container | Bind | Container path |
|---|---|---|
| radarr | `/mnt/user/media:/media` | Root: `/media/movies` |
| radarr-ger | `/mnt/user/media:/media` | Root: `/media/movies-ger` |
| radarr-3d | `/mnt/user/media:/media` | Root: `/media/movies-3d` |
| radarr-4k | `/mnt/user/media:/media` | Root: `/media/movies-4k` |
| sonarr | `/mnt/user/media:/media` | Root: `/media/series` |
| sonarr-ger | `/mnt/user/media:/media` | Root: `/media/series-ger` |
| lidarr | `/mnt/user/media:/media` | Root: `/media/music` |
| plex | `/mnt/user/media:/volume1` + `/mnt/user/media/temp:/transcode` | `/volume1/movies`, `/volume1/series`, `/volume1/music` |
| sabnzbd | `/mnt/user/media/download:/downloads` + `temp:/incomplete-downloads` | `/downloads/movies`, etc. |
| qbittorrentvpn | `/mnt/user/media/download:/data` | Categories auto-create subdirs via `auto_tmm` |
| tdarr | `/mnt/user/media/:/mnt/media` + `temp/:/temp` | |

## Hardlinks

Hardlinks only work on the same filesystem. Downloads on ZFS cache → media on ZFS cache = hardlinks work. After mover runs (ZFS → XFS array), hardlinks break — this is expected. Mover breaks hardlinks when moving from cache to array (cross-filesystem).

---

# Mover

**CA Mover Tuning** plugin — prevents mover from running during parity check.

| Setting | Value | Notes |
|---|---|---|
| Schedule | `0 */4 * * *` (every 4h) | 6 HDD wake-ups/day |
| Start threshold | 78% | ~1.33T on 1.7T cache |
| Stop threshold | 60% | ~1.0T, 18% swing (~300 GB) per run |
| Fill-up safety | 95% | Emergency backstop |
| Notifications | `movedOnly` | |
| Age mode | yes | Oldest-first |

---

# Arr Stack & Recyclarr

## Instances

| App | Port | Purpose |
|---|---|---|
| Radarr | 7878 | Movies (main) |
| Radarr | 7879 | Movies (4K) |
| Radarr | 7880 | Movies (3D) |
| Radarr | 7881 | Movies (German) |
| Sonarr | 8989 | TV (main) |
| Sonarr | 8990 | TV (German) |
| Lidarr | — | Music |
| Prowlarr | — | Indexer manager |

## Recyclarr

Running (`ghcr.io/recyclarr/recyclarr:latest`), syncing TRaSH custom formats to all 6 instances.
Config: `/mnt/user/appdata/recyclarr/recyclarr.yml` (423 lines)

Already configured:
- German Bluray tiers 1-3 + Web tiers 1-3
- HDR, DV Boost, HDR10+ Boost, Remux tiers
- LQ, BR-DISK, Extras, Upscaled blocking
- Plex-IMDB naming scheme
- Quality definitions synced

**German language CFs:** All instances have German LANGUAGE custom formats. Scoring strategy:

- **radarr, radarr-4k, sonarr** (Group A — DL preferred, German-only rejected):
  - German DL: +11,000 / German DL (undefined): +11,000
  - German: **-6,000** (rejects German-only; DL releases net +5,000 since both CFs match)
  - German Scene: +1,700 / 1080p Booster: +650 (HD) / 2160p Booster: +9,000 (UHD)
  - German LQ, Microsized, LQ (Release Title), Line/Mic Dubbed: -35,000 each
  - No "Not German or English" — foreign films with English subs accepted
- **radarr-ger, sonarr-ger** (Group B — German primary):
  - German DL: +11,000 / German: +10,000 / German Scene: +1,700 / 1080p Booster: +650
  - Not German or English: -35,000 / LQ penalties: -35,000 each
  - min_format_score: 5,000 (gates out English-only releases, max English score ~4,250)
- **radarr-3d**: No language CFs (3D gate at 100,000 makes them irrelevant)

**Anime CFs (sonarr only):** Anime quality differentiation with English/dual-audio preference. Quality definition: `anime` (min=5 MB/min, max=unlimited).

- **Anime group tiers** (TRaSH default scores):
  - BD Tier 01-08: +1,400 → +700 (premium to low BD encode groups)
  - Web Tier 01-06: +600 → +100 (premium to low web groups)
- **Anime Dual Audio: +5,000** (JP+EN releases — matches "dual-audio"/"multi-audio" in title only, ~50% detection rate at grab time)
- **Anime Raws: 0** (allowed but naturally deprioritized — raw groups don't match tier CFs so score stays 0)
- **VOSTFR: -10,000** (blocks French-subbed anime)
- **Anime LQ Groups: -10,000** (blocks poor-quality encode groups; overrides tier score if both match)
- **Version tags:** v0=-51, v1=+1, v2=+2, v3=+3, v4=+4 (prefer repacks over initial releases)
- **Score hierarchy:** Dual Audio+quality (6,400) > DA+unknown (5,000) > subbed+quality (1,400) > dubbed+quality (300) > untiered/raw (0) > VOSTFR/LQ (blocked)
- **Not added:** Dubs Only (no penalty needed), 10bit/Uncensored (informational only, score 0)

## Notes

- UmlautAdaptarr: removed — Radarr support not implemented, Lidarr sync silent. Revisit when Radarr support ships ([PCJones/UmlautAdaptarr releases](https://github.com/PCJones/UmlautAdaptarr/releases)).
- `x265 (HD)` set to 0 (not blocked) for German groups like ZeroTwo/VECTOR
- Quality profiles: Language=`Any`, Propers/Repacks="Do Not Prefer", merge qualities into groups

## Downloaders

### SABnzbd (Usenet — in use)

Container: `sabnzbd` (lscr.io/linuxserver/sabnzbd), port 8081
Version: 4.5.5

#### Key Settings

`cache_limit=4G`, `direct_unpack=1`, `propagation_delay=5min`, `connections=20` (primary), `unwanted_extensions=exe,com,cmd,bat,scr,pif,vbs,...` (action: abort), `pre_check=1`, `ionice=-c2 -n4`, `nice=-n10`, `bandwidth_perc=50`, `deobfuscate_final_filenames=1`, `fail_hopeless_jobs=1`, `fast_fail=1`, `enable_recursive=1`, `untar.sh` as default script. Sorting disabled (Arr manages renaming). SSL on both servers.

#### Server Config
- **Primary:** Newshosting (priority 0, 20 connections, SSL 563)
- **Fill:** Tweaknews (priority 1, 20 connections, SSL 563, retention 4200 days)

#### Categories

| Category | Dir (relative) | Used by |
|---|---|---|
| `movies` | `movies` | Radarr main |
| `movies-ger` | `movies-ger` | Radarr-ger |
| `movies-3d` | `movies-3d` | Radarr-3D |
| `movies-4k` | `movies-4k` | Radarr-4k |
| `tv` | `series` | Sonarr main |
| `tv-ger` | `series-ger` | Sonarr-ger |
| `audio` | `music` | Lidarr |
| `prowlarr` | (default) | Prowlarr |
| `*` (default) | (default) | Fallback — runs `untar.sh` |

### qBittorrent (Torrents)

Container: `binhex-qbittorrentvpn` (`ghcr.io/binhex/arch-qbittorrentvpn`, v5.1.4)
Data: `/mnt/user/media/download/:/data` | Stop-on-complete enabled (seeding time=0, action=Stop)

#### Key Settings

| Setting | Value |
|---|---|
| Torrent Management Mode | Automatic (categories auto-route) |
| Default Save Path | `/media/download` |
| Pre-allocate disk space | Disabled (mover can't move pre-allocated) |
| Disk cache | 0 (OS page cache) |
| Seeding time limit | 0 min (stop immediately) |
| Protocol | TCP |
| UPnP/NAT-PMP | Disabled (VPN port forwarding) |
| Max connections | 500 global, 100 per torrent |
| `auto_tmm_enabled` | true |

#### Categories

| Category | Save path | Used by |
|---|---|---|
| `movies` | `movies` | Radarr main |
| `movies-ger` | `movies-ger` | Radarr-ger |
| `movies-3d` | `movies-3d` | Radarr-3D |
| `movies-4k` | `movies-4k` | Radarr-4k |
| `series` | `series` | Sonarr main |
| `series-ger` | `series-ger` | Sonarr-ger |
| `music` | `music` | Lidarr |

#### VPN (WireGuard / NordLynx)

| Setting | Value |
|---|---|
| `VPN_CLIENT` | `wireguard` |
| `VPN_PROV` | `custom` |
| `LAN_NETWORK` | `192.168.86.0/24` |
| `NAME_SERVERS` | `103.86.96.100,103.86.99.100` |
| Privileged | On |
| Extra params | `--sysctl="net.ipv4.conf.all.src_valid_mark=1"` |

Config file: `/mnt/user/appdata/binhex-qbittorrentvpn/wireguard/wg0.conf`

```ini
[Interface]
PrivateKey = <YOUR_NORDLYNX_PRIVATE_KEY>
Address = 10.5.0.2/32
DNS = 103.86.96.100, 103.86.99.100

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <SERVER_IP>:51820
PersistentKeepalive = 25
```

**Auto server selection script:** `scripts/nas/nordvpn-wireguard-config.sh`
- Queries NordVPN API for best P2P WireGuard server by country
- Generates `wg0.conf` with your private key + server's public key
- Restarts container and verifies VPN
- Schedule via Unraid User Scripts (weekly)

Setup:
1. Get access token: https://my.nordaccount.com/dashboard/nordvpn/manual-configuration/
2. `export NORDVPN_TOKEN=<your_token>`
3. `./scripts/nas/nordvpn-wireguard-config.sh 209` (209=Switzerland, 81=Germany, 153=Netherlands)

**Note:** NordVPN does NOT support port forwarding. Downloads work fine, seeding is limited (peers can't initiate connections). For private trackers with ratio requirements, consider ProtonVPN or PIA (both support port forwarding in binhex container).

**Companion tools:**

| Tool | Purpose |
|---|---|
| **qbit_manage** | Auto-tag by tracker, orphan detection, share limits, NoHL tagging, cross-seed awareness |
| **qBitrr** | Stalled download auto-blacklist, proactive missing content search, hit-and-run protection |
| **cross-seed** | Auto-discover cross-seeding opportunities |
| **qBit-Mover** | Pause torrents → Unraid mover → resume (prevents locked file issues) |

---

# All Docker Containers

## Running

### Media Management
| Container | Image | Purpose | Memory |
|---|---|---|---|
| plex | linuxserver/plex | Media server | 896 MB |
| Plex-Auto-Languages | journeyover/plex-auto-languages | Auto language switching | |
| tautulli2 | tautulli/tautulli | Plex analytics | |
| immich | ghcr.io/imagegenius/immich | Photo management | 939 MB |
| overseerr | lscr.io/linuxserver/overseerr | Media request management | 567 MB |

### Arr Stack
| Container | Image | Port | Memory |
|---|---|---|---|
| radarr | linuxserver/radarr | 7878 | 778 MB |
| radarr-4k | linuxserver/radarr | 7879 | 623 MB |
| radarr-3d | linuxserver/radarr | 7880 | 353 MB |
| radarr-ger | linuxserver/radarr | 7881 | 334 MB |
| sonarr | linuxserver/sonarr | 8989 | 241 MB |
| sonarr-ger | linuxserver/sonarr | 8990 | |
| lidarr | linuxserver/lidarr | | 243 MB |
| binhex-prowlarr | binhex/arch-prowlarr | | 367 MB |
| recyclarr | ghcr.io/recyclarr/recyclarr | | |
| flaresolverr | flaresolverr/flaresolverr | | 393 MB |

### Downloaders
| Container | Image | Status | Notes |
|---|---|---|---|
| sabnzbd | lscr.io/linuxserver/sabnzbd | Running | Usenet |
| binhex-qbittorrentvpn | ghcr.io/binhex/arch-qbittorrentvpn | Running | qBit v5.1.4, WireGuard VPN (NordLynx) |

### Databases
| Container | Image | Memory | Notes |
|---|---|---|---|
| postgresql17 | postgres:17 | 326 MB (cold, grows to ~4.3 GB) | PG17, tuned |
| PostgreSQL_Immich | tensorchord/pgvecto-rs:pg16-v0.3.0 | 54 MB | PG16, shared_buffers=512MB |

### Infrastructure
| Container | Image | Purpose |
|---|---|---|
| pihole | pihole/pihole | DNS ad blocking (v6.4.1, macvlan br0, IP 192.168.86.2). See Pi-hole section for improvement suggestions. |
| CloudflaredTunnel | cloudflare/cloudflared | Cloudflare tunnel |
| resilio-sync | lscr.io/linuxserver/resilio-sync | File sync (903 MB) |
| syncthing | lscr.io/linuxserver/syncthing | File sync |
| uptime-kuma | louislam/uptime-kuma:2 | Service monitoring (v2.2.1, host network, port 3001, 27 active monitors + 2 paused, status page `/status/home`) |
| influxdb | influxdb:2 | Time-series database (v2.8.0, bridge, port 8086, org `nas-lian`, bucket `telegraf`, 90d retention) |
| telegraf | telegraf-smart:latest | Metrics collection (v1.38.2, host network, privileged, PID host). Collects: CPU, mem, disk, ZFS, Docker, SMART (SATA+NVMe), sensors. Custom exec: ZFS scrub, qBittorrent VPN, SABnzbd. |
| grafana | grafana/grafana-oss | Monitoring dashboards (bridge, port 3000, InfluxDB Flux data source configured) |

### Stopped / Created
| Container | Image | Notes |
|---|---|---|
| tdarr | ghcr.io/haveagitgat/tdarr | Media transcoding — stopped |
| kometa | lscr.io/linuxserver/kometa | Plex metadata manager — never started |

### ZED (ZFS Event Daemon)
Running (PID 13801). Custom ZEDLET routes ZFS events → Unraid notify → Telegram. Persisted to flash + boot script. See [ZED section](#zfs-error-notifications-zed) below.
Events handled: `statechange` (FAULTED/DEGRADED/REMOVED/UNAVAIL → alert), `checksum` (→ warning, 10-min throttle), `data` (→ alert, 10-min throttle), `scrub_finish` (parses error count from `zpool status`), `resilver_finish`. `trim_finish` suppressed (too noisy — 5 events/day).


---

# Monitoring

## Overview

Three monitoring layers:
1. **ZED** — ZFS event daemon → Telegram (pool errors, scrub results, resilver/TRIM)
2. **TIG Stack** — Telegraf + InfluxDB + Grafana (performance metrics, dashboards, historical trends)
3. **Uptime Kuma** — availability monitoring (is it up?) + Telegram alerts

## TIG Stack (Telegraf + InfluxDB + Grafana)

Deployed 2026-04-16. All containers created via Unraid Docker UI (templates at `/boot/config/plugins/dockerMan/templates-user/`).

### InfluxDB

Container `influxdb` (influxdb:2, v2.8.0, bridge, port 8086)
Appdata: `/mnt/user/appdata/influxdb2/`
Org: `nas-lian`, bucket: `telegraf`, retention: 90d
WebUI: `http://192.168.86.20:8086`

**Important:** Pin to `influxdb:2` tag — `latest` will point to InfluxDB v3 Core after **May 27, 2026**.

Storage estimate: ~50 metrics × 2 samples/min × 60 min × 24h × 90d ≈ 13M data points, ~2-5 GB.

### Telegraf

Container `telegraf` (telegraf-smart:latest, v1.38.2, host network, privileged, PID host)
Appdata: `/mnt/user/appdata/telegraf/`
Config: `/mnt/user/appdata/telegraf/telegraf.conf`
Scripts: `/mnt/user/appdata/telegraf/scripts/`
Extra params: `--restart=unless-stopped --pid=host --user=root:281 --group-add=281`

**Custom image** `telegraf-smart:latest` extends official Telegraf with smartmontools + nvme-cli + sudo (GLIBC mismatch prevents using host smartctl binary).

Bind mounts: `/var/run/docker.sock`, `/run/udev` (RO), `/` → `/host` (RO, for ZFS kstat at `/host/proc/spl/kstat/zfs`)

#### Telegraf config summary

```
[agent] interval=30s, flush_interval=30s
[global_tags] host="nas-lian"
Output: influxdb_v2 → localhost:8086, org nas-lian, bucket telegraf
```

| Plugin | Interval | Notes |
|---|---|---|
| `cpu`, `mem`, `disk`, `diskio`, `net`, `system`, `processes`, `kernel` | 30s | Standard system metrics |
| `zfs` | 30s | `kstatPath="/host/proc/spl/kstat/zfs"`, poolMetrics + datasetMetrics |
| `docker` | 30s | Via docker.sock |
| `postgresql` ×2 | 30s | PG17 (port 5432, arr DBs) + PG16 (port 5433, immich) via `host=127.0.0.1` |
| `smartctl` | **5m** | `use_sudo=true`, `timeout=120s` (29 devices × ~3s each) |
| `sensors` | 30s | Hardware sensors |
| exec: `zfs-scrub.sh` | 5m | ZFS scrub status for all pools |
| exec: `qbittorrent-vpn.sh` | 5m | VPN leak detection |
| exec: `sabnzbd-status.sh` | 2m | Queue depth, speed, paused state |

**PostgreSQL access:** Both PG instances need `host all all 172.17.0.0/16 trust` in `pg_hba.conf` — connections from Grafana/Telegraf arrive from Docker bridge gateway `172.17.0.1`, not `127.0.0.1`. Rule added before the `scram-sha-256` catch-all, reloaded with `pg_reload_conf()`.

#### Custom exec scripts

Scripts at `/mnt/user/appdata/telegraf/scripts/`:

**`zfs-scrub.sh`** — Parses `zpool status` for all pools, outputs scrub state + progress % (in-progress) or error count (completed). **Known issue:** `zpool` not available inside container — script outputs nothing. Needs host-side execution (cron + file relay, or nsenter).

**`qbittorrent-vpn.sh`** — Compares host IP vs container IP via `curl ifconfig.me`. Outputs `leak=1` if IPs match. Uses `docker exec` (needs docker.sock access).

**`sabnzbd-status.sh`** — Queries SABnzbd API (`localhost:8081`) for queue status, speed, pause state. Fixed 2026-04-18: rewrote with `grep -oP` (jq not available in Telegraf image). SABnzbd reachable via host networking on port 8081. Outputs `sabnzbd speed=...,queue_items=...,paused=...,reachable=1i`.

### Grafana

Container `grafana` (grafana/grafana-oss, bridge, port 3000)
Appdata: `/mnt/user/appdata/grafana/` (must be `chown 472:472` — Grafana runs as UID 472)
Dashboard: `http://192.168.86.20:3000` (admin/adminadmin)

#### Data Sources

| Name | Type | URL | Notes |
|---|---|---|---|
| InfluxDB (Flux) | `influxdb` | `http://172.17.0.1:8086` | Org `nas-lian`, bucket `telegraf`. Must use Docker bridge gateway IP — default bridge has no DNS. UID: `afjhubu6lx79cd` |
| PostgreSQL PG17 | `grafana-postgresql-datasource` | `172.17.0.1:5432` | User `postgres`, db `postgres`, version 1700. UID: `cfjhubu9dt534d` |
| PostgreSQL PG16 Immich | `grafana-postgresql-datasource` | `172.17.0.1:5433` | User `postgres`, db `immich`, version 1600. UID: `bfjhubucd6ry8a` |

#### Dashboards

Grafana v13, community + custom dashboards with Flux queries.

| Dashboard | Source | UID | Key Panels |
|---|---|---|---|
| **Telegraf Metrics (InfluxDB 2.0 Flux)** | Community (ID 15650) | `7JdIfTn9z` | CPU, memory, disk I/O, network, system overview — all Flux queries |
| **HC Monitor (Docker)** | Community (ID 14419) | `UMVMRJjGk` | Per-container CPU %, memory, network RX/TX, container status |
| **SMART Disk Health** | Custom | `d61fd40d-...` | Drive temperatures (time series + gauges), NVMe available spare %, power-on hours table |
| **PostgreSQL Overview** | Custom | `6b43f729-...` | PG17+PG16 cache hit ratio gauges, active connections, DB sizes, temp files, bgwriter stats, Telegraf transactions/s |

**SMART dashboard notes:**
- Uses `smartctl.temperature` field (not `temp_c`)
- Spun-down drives (STANDBY) report temp=0 with no model/serial tags — filtered with `model != ""` before pivot
- Available spare % is NVMe-only (SATA doesn't have this metric)
- 13/28 physical drives reporting when 15+ in standby — expected Unraid behavior
- SATA `smartctl_attributes.raw_value` for Temperature_Celsius is packed (unusable) — rely on `smartctl.temperature` instead

## Uptime Kuma

Container `uptime-kuma` (louislam/uptime-kuma:2, v2.2.1, host network, port 3001)
Appdata: `/mnt/user/appdata/uptime-kuma/` (must be on cache — SQLite needs low-latency storage)
Dashboard: `http://192.168.86.20:3001`
Status page: `http://192.168.86.20:3001/status/home`
Monitors: 27 active + 2 paused (Pi-hole — ipvlan unreachable from host)
Notifications: Telegram (same bot token + chat ID as Unraid)

### Monitor Groups

| Group | Monitors |
|---|---|
| Media | Plex, Radarr ×4, Sonarr ×2, Lidarr, Immich, Overseerr |
| Downloads | SABnzbd, Transmission, Prowlarr, qBittorrent |
| Infrastructure | Pi-hole DNS+Web (paused), PG17, PG16, Syncthing, Resilio, Unraid WebUI |
| External | Internet (1.1.1.1), Google DNS, TMDb, TVDb, MusicBrainz, NZBgeek, altHUB, DrunkenSlug, Plex.tv |

## Alert Thresholds

| Metric | Warning | Critical |
|---|---|---|
| ZFS pool state | DEGRADED | FAULTED |
| ZFS errors (cksum/read/write) | > 0 | > 10 |
| HDD temperature | > 45°C | > 50°C |
| NVMe temperature | > 65°C | > 75°C |
| Array capacity | > 85% | > 92% |
| Cache capacity | > 80% | > 90% |
| PG cache hit rate | < 97% | < 90% |
| PG connections (PG17) | > 40 | > 48 (max 50) |
| PG connections (PG16) | > 20 | > 28 (max 30) |
| Docker container restart | any | > 3 in 1h |
| qBittorrent VPN leak | VPN IP = host IP | Immediate |
| SABnzbd queue stalled | speed=0 with items | > 1h |
| RAM usage | > 80% | > 95% |

## Known Issues

- **PCIe RxErr on 01:00.0** — still accumulating correctable errors despite Gen3. WD Red SN700 needs physical reseat or slot swap.
- **InfluxDB exits with code 2 on first boot** — auto-recovers via restart policy. ~4 min metrics gap after each reboot.
- **sdb (disk17, ST12000VN0007 IronWolf 12TB): 602 UDMA CRC errors** — bad SATA cable or marginal port. 66K hours. Replace cable; plan drive replacement.
- **nvme0 (WDC SN550): 65% worn, 40K hours** — plan replacement within 1-2 years.
- **nvme4 (WD SN560) controller sensor at 81°C** — drive temp 59°C is fine. NAND throttle ~85°C. Monitor.
- **Array disk imbalance** — disk4/5/8 at 97% while disk1 at 21%. Consider running Unbalanced plugin.
- **PG17 log databases** show 58-63% cache hit — expected for append-only write-heavy tables, not actionable.
- **trim_finish notifications suppressed** — 5 events/day (one per vdev), too noisy.

---

# Installed Plugins

### Essential
| Plugin | Purpose |
|---|---|
| Community Applications | App store for Docker/plugins |
| CA Mover Tuning | Prevents mover during parity check |
| CA Update Applications | Auto-update containers |
| CA Appdata Backup | Backup appdata |
| CA Cleanup Appdata | Clean orphaned appdata |
| Fix Common Problems | Health check |

### Storage
| Plugin | Purpose |
|---|---|
| Unassigned Devices (+preclear) | External drive management |
| Unbalanced | Rebalance array disk usage |
| Disk Location | Physical disk bay mapping |
| Dynamix File Integrity | File checksum monitoring |

### Monitoring
| Plugin | Purpose |
|---|---|
| Dynamix System Stats/Info/Temp | System monitoring |
| GPU Stat | GPU monitoring |
| Intel GPU Top | Intel iGPU monitoring |
| Corsair PSU | PSU monitoring |
| Network Stats | Network monitoring |
| Statistics Sender | Unraid telemetry |
| IPMI | IPMI management |

### Networking
| Plugin | Purpose |
|---|---|
| Tailscale | VPN mesh (nas-lian, 6 devices) |
| WakeOnLAN | Remote wake |

### Utilities
| Plugin | Purpose |
|---|---|
| User Scripts | Custom scripts (gdrive backup, proton pass backup, nordvpn-wg-config, plex-languages-restart, VM start/stop) |
| Tips and Tweaks | System tuning (flow control, TSO, VM dirty ratio) |
| NerdPack/NerdTools | Additional CLI tools |
| rclone | Cloud storage sync |
| Dynamix Cache Dirs | Keep directory structure in RAM |
| Speed Test | Network speed testing |

---

# Pi-hole DNS

## Current State (2026-04-16)

| Property | Value |
|---|---|
| Container | `pihole` (pihole/pihole:latest) on macvlan `br0` |
| IP | 192.168.86.2 |
| Version | Core v6.4.1, Web v6.5, FTL v6.6 (latest) |
| Upstream DNS | **Quad9 (9.9.9.9, 149.112.112.112)** |
| DNSSEC | **Enabled** |
| Listening mode | **LOCAL** (safe — only responds to local subnets) |
| Blocked domains | 1,631,565 unique (Hagezi Pro + TIF + SmartTV + OISD NSFW) |
| Memory | 157 MiB |
| Conditional forwarding | **Enabled** — `192.168.86.0/24` → `192.168.86.1` (domain: `lan`) |
| Local DNS records | 4 records (nas-lian, pihole, plex, immich) → `.lan` domain |
| Client visibility | **Limited** — Google/Nest Wifi proxies all DNS; Pi-hole sees router IP (192.168.86.1) for 99.99% of queries. Hardware limitation, no fix without replacing router or using Pi-hole DHCP. |
| Config | `/mnt/user/appdata/pihole/pihole/pihole.toml` |

## Blocklists

### Active

| ID | List | Groups | Notes |
|---|---|---|---|
| 18 | Perflyst SmartTV | Default, IoT | Samsung/LG/Vizio telemetry |
| 20 | **Hagezi Pro** (`hagezi/dns-blocklists@latest/adblock/pro.txt`) | Default, Kids, IoT | ~410K domains, replaces all Firebog lists |
| 21 | **Hagezi TIF** (`hagezi/dns-blocklists@latest/adblock/tif.txt`) | Default, Kids, IoT | Threat intelligence — malware, phishing, C2 |
| 22 | **OISD NSFW** (`nsfw.oisd.nl`) | Kids only | ~497K adult content domains |

## Groups

| ID | Name | Lists | Purpose |
|---|---|---|---|
| 0 | Default | Hagezi Pro, TIF, SmartTV | Standard blocking for all clients |
| 1 | Kids | Hagezi Pro, TIF, **OISD NSFW** | Adult content blocking added |
| 2 | IoT | Hagezi Pro, TIF, SmartTV | Telemetry blocking for smart devices |

**No clients assigned** — requires static DNS (192.168.86.2) on target devices to bypass Google Wifi DNS proxy. Assign via Pi-hole WebUI → Groups → Clients.

## Allowlist

Hagezi Pro blocks some legitimate Roche/work domains. Allowlisted via `pihole allow` and `pihole --allow-regex`:

| Domain | Type | Reason | Hagezi Rule |
|---|---|---|---|
| `navify.com` | regex allow `(\.\|^)navify\.com$` | Roche Diagnostics NAVIFY platform (Tumor Board, Clinical Decision Support, calculation services). False positive — filed as issue on hagezi/dns-blocklists. | `\|\|navify.com^` |
| `eu-ec.walkme.com` | exact allow | WalkMe in-app guidance used by Roche for internal tools/training | `\|\|eu-ec.walkme.com^` |
| `xp.atlassian.com` | exact allow | Atlassian Experience Platform — analytics for Jira/Confluence | `\|\|xp.atlassian.com^` |

**Not allowlisted** (telemetry, acceptable to block):
- `prod.log.shortbread.aws.dev` — AWS cookie consent logger
- `ams-pageview-public.s3.amazonaws.com` — pageview analytics
- `dn0qt3r0xannq.cloudfront.net` — CDN analytics

**Not blocked by Pi-hole** (NXDOMAIN — internal-only, no public DNS):
- `fleet.kubemea.roche.com` — Fleet/osquery endpoint (needs WARP/RCN)
- `rkamsemea01.emea.roche.com`, `rbamsemea01.emea.roche.com` — Roche EMEA servers
- `ghe-rss.roche.com`, `overflow.roche.com` — GitHub Enterprise, Roche Overflow

### Tips and Tweaks Settings
```
FLOW_CONTROL=yes, TSO_OFFLOAD=yes
RX/TX_BUFFER=256, VM_BACKGROUND=1, VM_DIRTY=2
```

---

# User Scripts

| Script | Purpose |
|---|---|
| gdrive backup | Google Drive backup |
| nordvpn-wireguard-config | Weekly NordVPN WireGuard server rotation (Monday 5 AM) |
| plex-languages-restart | Restart Plex Auto Languages (hourly) |
| vm-win11-start | Start Windows 11 VM |
| vm-win11-shutdown | Shutdown Windows 11 VM |

---

# Tailscale

Connected devices: nas-lian (active), kvm-lian, kvm-b6ee, gpd-win4 (offline 658d), iPad Pro, iPhone 15

---

# Unraid API

## Overview

GraphQL API built into Unraid 7.2+. Provides programmatic access to system monitoring, Docker, VMs, array, shares, network, and config. No REST endpoints — GraphQL only at `/graphql`.

**Backend:** Node.js process (`unraid-api` v4.32.2+12933aa0) → Unix socket `/var/run/unraid-api.sock` → nginx proxy. No TCP port — socket only.

## Authentication

| Key | Purpose | Notes |
|---|---|---|
| Local API key | LAN/Tailscale access | 64-char hex, stored in `myservers.cfg`. Auto-generated — has NO permissions (UNAUTHENTICATED for most queries) |
| ADMIN API key | Full API access | Create via CLI: `unraid-api apikey --create --name "name" --roles ADMIN --json`. Required for useful queries |
| Connect API key | Unraid.net cloud access | `unraid_Br...` prefixed, for remote access |

Local key is all you need for on-LAN automation. Connect account linked to `[REDACTED_EMAIL]` / `vertiger`.

## Usage

```bash
# Query via ADMIN API key (must use Unix socket from NAS, or nginx proxy from LAN)
# From NAS (via SSH):
curl -s --unix-socket /var/run/unraid-api.sock \
  -H "x-api-key: <ADMIN_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ info { os { hostname } } }"}' \
  http://localhost/graphql

# From LAN (via nginx proxy):
curl -X POST http://192.168.86.20/graphql \
  -H "Content-Type: application/json" \
  -H "x-api-key: <ADMIN_API_KEY>" \
  -d '{"query":"{ info { os { platform } } }"}'

# Also reachable via Tailscale: http://100.110.47.46/graphql
```

**Note:** The localApiKey from `myservers.cfg` returns UNAUTHENTICATED for most queries. You need a CLI-created ADMIN key for actual data access.

## UI Access

- **API Keys**: Settings → Management Access → API Keys
- **GraphQL Sandbox**: Settings → Management Access → Developer Options → enable sandbox toggle (currently off — disable after introspection for security)
- **CLI**: `unraid-api apikey --create --name "name" --roles ADMIN --json`

## Available Roles

| Role | Access |
|---|---|
| ADMIN | Full access |
| VIEWER | Read-only |
| CONNECT | Remote access features |
| GUEST | Limited |

Fine-grained permissions: `DOCKER:READ_ANY`, `ARRAY:READ_ANY`, `SYSTEM:READ_ANY`, etc.

## Use Cases

- Monitoring scripts (read-only VIEWER key)
- Docker container automation
- Infrastructure-as-code workflows
- Custom dashboards (Grafana → GraphQL datasource)

**Docs:** https://docs.unraid.net/API/

## iOS / Mobile Access

No official Unraid iOS app exists. Options:

| App | Auth Method | Tailscale | Notes |
|---|---|---|---|
| **Unraid Deck** (recommended) | **Unraid API key** (GraphQL API) + optional Agent plugin for push notifications | Explicitly supported | Native iOS, widgets, push notifications, Docker/VM management. Settings → Management Access → API → create key with "Unraid Deck" permission preset. Agent plugin (`unraid-deck-agent`) is only for push notifications, not primary auth. Use `100.110.47.46` as server address. |
| **ControlR** | Plugin + QR code pairing (own auth) | Works (any reachable IP) | Install `controlrd` plugin, scan QR from Plugins → ControlR. Does NOT use the Unraid API key. |
| **Unraid Connect** (browser) | Unraid.net account | N/A (cloud dashboard) | Already configured (`[REDACTED_EMAIL]`). Go to `connect.myunraid.net`. Monitoring only — WAN access disabled. No native iOS app — browser only. |

Unraid Deck is the only app that uses the native GraphQL API key. ControlR and Connect use their own auth mechanisms.

---

# NUT

NUT plugin installed with usbhid-ups driver — this is for the **Corsair PSU monitoring**, not a UPS. No UPS connected.

---

# Backups

## Appdata Backup (CA plugin)

Weekly backups at `/mnt/user/backup/appdata/`. Includes PostgreSQL appdata (`postgresql17.tar.gz` / `.tar.zst`) — filesystem-level, crash-consistent on ZFS. Sufficient for disaster recovery but not point-in-time recovery.

## rclone / Google Drive Backup

User script `gdrive backup` intended to sync Google Drive → `/mnt/user/backup/gdrive/` (1.2 GB, stale from May 2025). Google Photos sync is commented out.

**Issue**: `rclone.conf` does NOT exist — the gdrive remote is not configured. Script fails silently.

### Setup Guide

#### 1. Create your own Google API Client ID (MANDATORY for performance)

Using the shared rclone Client ID results in severely throttled performance (all rclone users share the same quota).

1. Go to https://console.developers.google.com/
2. Create new project → Enable Google Drive API
3. Credentials → Configure Consent Screen → External → add your email as test user
4. Create OAuth Client → Desktop app → save Client ID + Secret
5. **CRITICAL: Publish the app** (Audience → PUBLISH APP). Testing mode tokens expire after 7 days, breaking automated backups.

#### 2. Configure rclone remote on Unraid

Config file location: `/boot/config/plugins/rclone/.rclone.conf` (persists on USB flash)

```bash
# SSH to Unraid, use SSH tunnel for browser auth:
# From your laptop: ssh -L 53682:localhost:53682 root@192.168.86.20
rclone config
# n) New remote → name: gdrive → Storage: drive
# Enter YOUR Client ID and Secret
# Scope: 1 (full access)
# Answer Y to browser auth → authenticate in browser at localhost:53682
```

#### 3. Test

```bash
rclone lsd gdrive:                # List top-level folders
rclone about gdrive:              # Storage usage
rclone ls gdrive: --max-depth 1   # List files
```

#### 4. Recommended backup script

Replace the existing `gdrive backup` user script with:

```bash
#!/bin/bash
# Google Drive Backup to Unraid

RCLONE_CONFIG="/boot/config/plugins/rclone/.rclone.conf"
LOG_DIR="/mnt/user/appdata/rclone/logs"
BACKUP_DIR="/mnt/user/backup/gdrive"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/gdrive-backup-${DATE}.log"

mkdir -p "$LOG_DIR" "$BACKUP_DIR"
find "$LOG_DIR" -name "gdrive-backup-*.log" -mtime +30 -delete

echo "=== Google Drive Backup Started: $(date) ===" | tee -a "$LOG_FILE"

rclone copy gdrive: "$BACKUP_DIR" \
  --config "$RCLONE_CONFIG" \
  --progress \
  --transfers 4 \
  --checkers 8 \
  --tpslimit 10 \
  --bwlimit 50M \
  --drive-acknowledge-abuse \
  --fast-list \
  --log-file "$LOG_FILE" \
  --log-level INFO \
  --retries 3 \
  --retries-sleep 10s \
  --low-level-retries 10 \
  --stats 60s \
  2>&1 | tee -a "$LOG_FILE"

EXIT_CODE=$?
echo "=== Finished: $(date) (exit: $EXIT_CODE) ===" | tee -a "$LOG_FILE"

if [ $EXIT_CODE -eq 0 ]; then
  /usr/local/emhttp/webGui/scripts/notify -s "Google Drive Backup" -d "Completed successfully" -i "normal"
else
  /usr/local/emhttp/webGui/scripts/notify -s "Google Drive Backup FAILED" -d "Exit code $EXIT_CODE. Check $LOG_FILE" -i "alert"
fi
```

Key flags:
- `rclone copy` (not sync) — never deletes local files, safe for backup
- `--tpslimit 10` — stays within Google API rate limits
- `--bwlimit 50M` — doesn't saturate network
- `--fast-list` — up to 20x faster directory listing
- `--drive-acknowledge-abuse` — downloads flagged files
- Unraid notification on success/failure

Schedule: **daily at 3 AM** via User Scripts Custom Cron: `0 3 * * *`

#### 5. `rclone copy` vs `rclone sync`

| | `copy` | `sync` |
|---|---|---|
| Adds new/changed files | Yes | Yes |
| Deletes local files not on Drive | **No** | **Yes** |
| Risk of data loss | Low | High |

Use `copy` for backup. If you want exact mirror with `sync`, add safety: `--max-delete 100 --backup-dir /path/to/deleted/$(date +%Y%m%d)`

### Google Photos — rclone backend is DEAD

As of March 2025, Google removed the `photoslibrary.readonly` scope. rclone can only download photos that rclone itself uploaded — **cannot download your existing photo library**. The backend is deprecated (Tier 5).

**Alternative: Google Takeout** (recommended for now)
1. Go to https://takeout.google.com → select Google Photos only
2. Export to Google Drive (or download directly)
3. If exported to Drive, the rclone backup picks it up automatically
4. Schedule recurring Takeout (every 2 months for 1 year)
5. Original quality, all metadata preserved, GPS data included

**Immich** is already running on this NAS as secondary Google Photos backup. Consider using `immich-go` to import Takeout data:
```bash
immich-go upload from-google-photos --server=http://localhost:2283 --api-key=KEY /path/to/takeout/
```

**NEW: gphotosdl** (Google Photos Downloader) — a proxy tool that enables rclone to download **original resolution** photos/videos with EXIF/GPS preserved. Works alongside the existing rclone googlephotos remote.

```bash
# Install gphotosdl (check https://github.com/rclone/gphotosdl for latest)
# 1. Login (one-time, opens browser)
gphotosdl -login

# 2. Start proxy (terminal 1)
gphotosdl

# 3. Download via rclone with proxy (terminal 2)
rclone copy -vvP \
  --gphotos-proxy "http://localhost:8282" \
  "gPhotos:media/by-month/" \
  "/mnt/user/backup/gphotos/"
```

Requires rclone v1.69+. The proxy runs a headless browser to bypass the API's compression/EXIF stripping.

**Also watch**: rclone PR #9135 (`gphotosmobile` backend) — uses Android mobile API. Not merged yet.

### Google API Limits

| Limit | Value |
|---|---|
| Queries/min | 12,000 |
| Daily upload | 750 GB (locked out 24h if exceeded) |
| Daily download | ~10 TB |
| Token refresh | Automatic (rclone handles it) |
| Token expiry | Never (if app is in Production mode) |

## Immich

Secondary backup of Google Photos — not the primary copy. Photos stored at `/mnt/user/photo/immich`.

---

# Parity

Single parity (sdo, 29.1 TB ST32000NT000 IronWolf Pro HAMR). Monthly checks, all clean (0 errors since Aug 2025). Latest check: Apr 3, 2026 (239K seconds = ~2.8 days).

Single parity is acceptable — media content is replaceable.

---

# Disk Inventory (24 data disks + 1 parity)

| Drive | Size | Model | Serial |
|---|---|---|---|
| sdo (P) | 29.1T | ST32000NT000-3TP103 | K1S15S11 |
| sdh | 29.1T | ST32000NT000-3TP103 | K1S1EEPA |
| sde | 23.6T | WDC WUH722626ALE6L4 | SZG1MWTM |
| sdx | 23.6T | WDC WUH722626ALE6L4 | SZG5TL8M |
| sdg | 21.8T | WDC WUH722424ALE6L4 | 65J14KSB |
| sdt | 21.8T | WDC WUH722424ALE6L4 | 65J14ERB |
| sdc | 18.2T | ST20000NM007D-3DJ103 | ZVT0HHHM |
| sdd | 18.2T | ST20000NM007D-3DJ103 | ZVT050KT |
| sdf | 18.2T | ST20000NM007D-3DJ103 | ZVT1V3E0 |
| sds | 18.2T | ST20000NM007D-3DJ103 | ZVT5PNN4 |
| sdw | 18.2T | ST20000NM007D-3DJ103 | ZVT5KXW2 |
| sdr | 16.4T | WDC WD181KRYZ-01AGBB0 | 3GKZ8J2E |
| sdu | 16.4T | WDC WD181KRYZ-01AGBB0 | 3RG71YDA |
| sdv | 16.4T | ST18000NM000J-2TV103 | ZR52ZTE4 |
| sdy | 16.4T | ST18000NM000J-2TV103 | ZR52ZTFF |
| sdp | 14.6T | ST16000VN001-2RV103 | ZL22EVCR |
| sdq | 14.6T | ST16000VN001-2RV103 | ZL22EWL9 |
| sdb | 10.9T | ST12000VN0007-2GS116 | ZJV1MHSX | **602 UDMA CRC errors, 66K hours — replace cable + plan drive replacement** |
| sdi | 10.9T | ST12000VN0007-2GS116 | ZJV1JJDF |
| sdm | 10.9T | ST12000VN0007-2GS116 | ZCH0F7B8 |
| sdn | 10.9T | ST12000VN0007-2GS116 | ZJV2D16A |
| sdj | 7.3T | WDC WD80EFZX-68UW8N0 | R6GNSATY |
| sdk | 7.3T | WDC WD80EFZX-68UW8N0 | VK0H5DSY |
| sdl | 7.3T | WDC WD80EFZX-68UW8N0 | VK1E8L3Y |

Disk tiers: 2x 29T, 2x 24T, 2x 22T, 5x 18T, 4x 16T, 2x 15T, 4x 11T, 3x 7T

---

---

# ZFS Error Notifications (ZED)

## Overview

ZED (ZFS Event Daemon) monitors kernel ZFS events and runs scripts in response. It ships with OpenZFS and should be available on Unraid 6.12 with ZFS 2.3.4. It's the standard way to get notified about zpool errors, scrub results, and device failures.

## Key Events to Monitor

| Event | Script | What it detects |
|---|---|---|
| Device fault | `statechange-notify.sh` | FAULTED, DEGRADED, REMOVED, UNAVAIL vdevs |
| Data corruption | `data-notify.sh` | Checksum errors, data integrity failures |
| Scrub complete | `scrub_finish-notify.sh` | Scrub results (only notifies if errors found, unless verbose) |
| Resilver complete | `resilver_finish-notify.sh` | Resilver finished |
| TRIM complete | `trim_finish-notify.sh` | TRIM operation finished |

---

# TODO

## Google Drive Backup

- [ ] Create Google API Client ID (console.developers.google.com) and **publish to Production**
- [ ] Configure rclone gdrive remote (`rclone config` with SSH tunnel for browser auth)
- [ ] Replace gdrive backup user script with improved version
- [ ] Schedule gdrive backup: Custom Cron `0 3 * * *` (daily 3 AM)
- [ ] Google Photos: set up `gphotosdl` proxy for original-quality downloads via rclone

## Pending — Hardware

- [ ] **nvme0 (WDC SN550): plan replacement** — 65% worn, 40,413 hours. Replace within 1-2 years.
- [ ] **nvme4 (WD SN560): monitor controller temp** — internal sensor 81°C (NAND throttle ~85°C). Drive temp 59°C is fine.
- [ ] **PCIe RxErr on 01:00.0** — still accumulating correctable errors despite Gen3. WD Red SN700 needs physical reseat or slot swap.

---

# TRaSH Guide Links

- Unraid setup: https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Unraid/
- Hardlinks: https://trash-guides.info/File-and-Folder-Structure/Hardlinks-and-Instant-Moves/
- Radarr quality profiles: https://trash-guides.info/Radarr/radarr-setup-quality-profiles/
- Radarr German profiles: https://trash-guides.info/Radarr/radarr-setup-quality-profiles-german-en/
- Sonarr German profiles: https://trash-guides.info/Sonarr/sonarr-setup-quality-profiles-german-en/
- Radarr naming: https://trash-guides.info/Radarr/Radarr-recommended-naming-scheme/
- Custom formats: https://trash-guides.info/Radarr/Radarr-collection-of-custom-formats/
- qBittorrent: https://trash-guides.info/Downloaders/qBittorrent/Basic-Setup/
- SABnzbd: https://trash-guides.info/Downloaders/SABnzbd/Basic-Setup/
