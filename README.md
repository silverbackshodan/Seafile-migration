# 📋 Postmortem: Nextcloud → Seafile Migration

> A self-hosted file-sync platform migration. What went well, what broke, and what I'd do differently.

![Status](https://img.shields.io/badge/Status-Resolved-2ea44f)
![Type](https://img.shields.io/badge/Type-Postmortem-blue)
![Severity](https://img.shields.io/badge/Severity-Planned_Migration-orange)
![Downtime](https://img.shields.io/badge/Downtime-~2h-yellow)

> **Note on anonymization:** Domain names and infrastructure identifiers in this writeup have been replaced with placeholders (`<homelab>`, `example.com`). The architecture, debugging steps, and lessons learned are real — production hostnames are not.

---

## TL;DR

Migrated a homelab's primary file-sync platform from **Nextcloud** to **Seafile Pro 13** to solve performance bottlenecks on large libraries and reduce operational overhead. The migration succeeded with **~2 hours of planned downtime** and **zero data loss**, but uncovered three non-obvious failure modes that cost a full afternoon of debugging. This document captures the timeline, root causes, and decisions — both the good ones and the ones I'd reverse.

---

## 📊 Migration at a Glance

| Metric | Value |
|---|---|
| **Source platform** | Nextcloud (Docker, MariaDB backend) |
| **Target platform** | Seafile Pro 13.x (Docker, MySQL backend, NFS-backed storage) |
| **Data migrated** | Multiple libraries, multi-GB total |
| **Planned downtime** | 1 hour |
| **Actual downtime** | ~2 hours |
| **Data loss** | 0 |
| **Rollback events** | 0 (snapshot held in reserve, never used) |
| **Post-migration sync speed** | ~3–5× faster on large libraries (subjective, large-folder scenarios) |

---

## 🎯 Why Migrate?

Nextcloud had served well as a Google-Drive replacement for ~18 months, but three pain points accumulated:

### 1. Sync performance on large libraries
Nextcloud's WebDAV-based sync struggled with libraries containing thousands of small files. Initial sync of a media-asset library could take hours and would frequently stall. Seafile uses a chunk-based protocol with deduplication — a fundamentally different architecture for this workload.

### 2. Storage efficiency
Nextcloud stores files 1:1 on disk with metadata in the DB. Seafile stores content-addressed blocks, so duplicate files across libraries deduplicate automatically. Estimated savings on the dataset: ~15–20%.

### 3. Operational overhead vs. actual usage
Nextcloud is a platform: files, calendar, contacts, mail, office, talk. Actual usage was ~10% of it (files only). Calendar and contacts already lived elsewhere. Every Nextcloud upgrade was a 30-minute event with app-compatibility-matrix anxiety. Seafile does one thing — file sync — and does it without ceremony.

**Decision:** Trade platform breadth for sync speed and operational simplicity. Files only. Anything else stays where it is.

---

## 🗺️ Migration Plan (as designed)

```mermaid
flowchart LR
    A[Snapshot Nextcloud<br/>data + DB] --> B[Stand up Seafile<br/>stack on NFS]
    B --> C[Create libraries<br/>matching folder structure]
    C --> D[Bulk upload via<br/>seaf-import on server]
    D --> E[Verify file counts<br/>+ checksums]
    E --> F[Repoint clients<br/>SeaDrive / mobile]
    F --> G[Keep Nextcloud read-only<br/>for 14 days]
    G --> H[Decommission]

    classDef done fill:#1f4e2c,stroke:#4ade80,color:#fff
    class A,B,C,D,E,F,G,H done
```

**Pre-flight checklist:**
- ✅ Btrfs snapshot of NAS volume (locked, named `pre-seafile-migration`)
- ✅ Local backup of Nextcloud `data/` directory and DB dump
- ✅ Seafile stack tested on a separate subdomain before cutover
- ✅ Client list documented (which devices need re-pairing)
- ✅ 2-hour maintenance window communicated to users

---

## ⏱️ Timeline

| Time | Event |
|---|---|
| `T-1d` | Btrfs snapshot taken; `mysqldump` of Nextcloud DB to NAS + offsite |
| `T-0` | Maintenance window begins. Nextcloud put in maintenance mode. |
| `T+0:05` | Seafile stack deployed (6 containers: db, redis, memcached, seasearch, sdoc-server, notification-server, seafile core) |
| `T+0:20` | **Issue #1 detected** — Seahub fails to start, DB auth errors |
| `T+0:50` | Issue #1 resolved (see RCA below) |
| `T+0:55` | Libraries created, bulk import begins |
| `T+1:30` | Import complete; file count and sample checksums verified |
| `T+1:45` | **Issue #2 detected** — `notification-server` in restart loop |
| `T+1:55` | Issue #2 resolved |
| `T+2:00` | Clients re-paired (SeaDrive + mobile). Service restored. |
| `T+5d` | **Issue #3 detected** — secondary feature non-functional (deferred, see below) |

---

## 🔥 What Went Wrong

### Issue #1: Application-Layer DB Credential Drift After Upgrade Chain

**Symptom:** Seahub container crashed on startup with `Access denied` errors. MySQL root credentials in `.env` were correct — verified by direct DB connection.

**Root cause:** Seafile maintains an **internal application password**, separate from the MySQL root password. This UUID-based credential is stored in the application's config files. During the multi-version upgrade chain, the MySQL container had been recreated with a fresh root password in `.env`, but the application's config files still referenced the original UUID. The container appeared healthy because root auth worked; the application user was the broken layer.

**Detection:** Container logs identified the failed user (the application user, not root) — this pointed at the application-credential layer once I stopped assuming it was a root-password issue.

**Fix:** Aligned application config files to a single known password, restarted the stack. Documented all locations where credentials must agree.

**Time lost:** ~30 min.

---

### Issue #2: notification-server Restart Loop

**Symptom:** Stack stable, web UI working, but `docker ps` showed `notification-server` flapping between `Up` and `Restarting`.

**Root cause:** The container's startup script expected a log directory that the image didn't auto-create. The first line of stderr said exactly that — but in a 6-container stack, I'd been tailing the wrong logs.

**Fix:** Created the missing directory with correct ownership, restarted the container.

**Time lost:** ~10 min (after I started reading the right logs).

---

### Issue #3: Secondary Feature Non-Functional (Deferred)

**Symptom:** A secondary collaboration feature (added in the new major version) failed to function correctly post-migration.

**Root cause:** The new version introduced a feature requiring additional reverse-proxy configuration that wasn't present in the upgrade path.

**Status:** **Deferred.** The affected feature isn't in active use. Two paths forward when it becomes a priority:
1. Apply the additional proxy configuration the new version requires
2. Roll back to the prior major version where the feature works without additional configuration

**Decision pending** until the feature is needed or upstream documentation improves.

---

## ✅ What Went Right

- **The Btrfs snapshot was never used** — but the fact that it existed turned a one-way migration into a two-way decision. This made the whole project psychologically tolerable.
- **Pre-flight import test** on a non-production subdomain caught two unrelated config issues before the maintenance window.
- **Sample-checksum verification** post-import gave me confidence to flip clients over without waiting days.
- **Read-only Nextcloud for 14 days** caught two files I'd forgotten to migrate (a hidden settings folder).

---

## 🎓 Lessons Learned

### 1. App credentials and infra credentials are different layers
When auth fails, the question isn't "is the password right?" — it's "*which* password, *where*, and is it consistent across all the places that reference it?" Some apps maintain their own credentials separate from the database root. Always map them before debugging.

### 2. In a multi-container stack, log discipline matters
`docker compose logs -f` on the whole stack is noisy. When something is flapping, isolate to that container immediately: `docker logs -f <container>`. I lost 10 minutes to noise on Issue #2 that I'd have caught in 30 seconds with the right command.

### 3. "It works" is not the same as "it works for everything"
Issue #3 didn't surface until five days post-migration because that feature wasn't in my verification checklist. Lesson: build a feature checklist for verification, not just a "can I log in and see files" smoke test.

### 4. Trade platform breadth deliberately, not by accident
Nextcloud → Seafile was a deliberate downscope. I lost calendar/contacts/office on purpose because alternatives were already in place. If I hadn't planned that explicitly, the migration would have looked like a regression rather than a focused improvement.

### 5. Snapshots change the risk profile, not just the recovery profile
Knowing rollback was 30 seconds away changed how I made decisions during the window — I committed to fixes faster instead of agonizing.

---

## 📈 Outcome — 30 Days In

- **Sync speed:** Noticeably faster on large libraries (subjective). Initial sync of the largest library went from "go make coffee and come back" to "still faster than coffee."
- **Storage:** ~17% reduction on the deduplicated dataset (NAS-reported).
- **Operational overhead:** One stack to update instead of one stack + app-compatibility matrix. Update time roughly halved.
- **Open issue:** Issue #3 — deferred, not blocking.

**Would I do it again?** Yes — and earlier. The performance delta on large libraries alone justified the migration; the operational simplification was a bonus.

---

## 🛠️ Stack Reference

**Source:**
- Nextcloud (latest stable at time of migration)
- MariaDB
- Docker Compose, Traefik routing, Cloudflare Tunnel ingress

**Target:**
- Seafile Pro 13.x (6-container stack)
- MySQL 8
- Storage: NFS mount → Synology NAS (Btrfs, snapshots enabled)
- Search: SeaSearch (Elasticsearch-based)
- Collaborative editing: SeaDoc Server
- Notifications: Notification Server
- Cache: Redis + Memcached
- Same ingress chain (Traefik → Cloudflare Tunnel)

---

## 👤 Author

**Francesco Ascariz** — [@silverbackshodan](https://github.com/silverbackshodan)
Web Developer & Security Enthusiast

> Production migrations as a learning practice. The interesting part isn't the happy path — it's what you do when the happy path turns out to have four credential files instead of one.
