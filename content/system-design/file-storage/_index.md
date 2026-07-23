---
title: 'Dropbox / Google Drive'
weight: 1
type: docs
---

You are asked to design a cloud file-storage and synchronisation service like Dropbox or Google Drive. A user installs a desktop or mobile client, drops files into a watched folder, and those files are uploaded, encrypted, and automatically synchronised to every other device tied to that account — with changes propagating in seconds. Users can share individual files or entire folder trees with teammates, set read-only or read-write permissions, and recover accidentally deleted or overwritten files from version history.

The design is harder than it looks. A naïve "re-upload the whole file on every change" approach burns both bandwidth and storage at scale. The core insight is to treat a file as an **ordered list of content-addressed blocks**: split the file into chunks, hash each chunk with SHA-256, store each unique chunk exactly once across the entire platform, and on edits transfer only the changed chunks. This transforms a 50 GB re-upload into a handful of 4 MB block transfers — while also deduplicating storage cross-user (popular OS files, shared documents, and identical media are stored once regardless of how many users own them).

## Functional Requirements

1. **Upload:** Upload a file of any size (up to 50 GB) from web, desktop, or mobile client.
2. **Download:** Download any owned or shared file from any device.
3. **Sync:** Changes made on one device propagate automatically to all other devices for the same user.
4. **Chunking & deduplication:** Files are split into content-defined blocks; identical blocks are stored exactly once across all users and files (cross-user dedup).
5. **Delta sync:** On a file edit, only the changed blocks are uploaded, not the full file.
6. **Share:** Share a file or folder with another user with read-only or read-write permission.
7. **Versioning:** Retain the last N versions of each file so accidental overwrites are recoverable.
8. **Resumable upload:** An interrupted large-file upload can resume without re-sending already-received blocks.

## Out of Scope

- Real-time co-editing within a document (Google Docs live-cursor / OT). We cover conflict resolution at the file level only.
- Video transcoding, image thumbnailing, and media processing pipelines.
- Full-text search of document contents.
- Virus / malware scanning (a downstream async pipeline, not in the upload critical path).
- Billing, quota enforcement, and subscription management.

## Non-Functional Requirements

- **Scale:** 500 M registered users; 50 M DAU; 20 GB average storage per user → ~10 EB raw data. After a ~2:1 platform-wide dedup ratio the net on-disk footprint is ~5 EB.
- **Upload throughput:** ~2,900 file-upload initiations/s average; ~5,800 block uploads/s average; peak ×3 burst.
- **Latency:** First-block ACK p99 < 1 s; CDN download first-byte p99 < 200 ms; sync notification delivery p99 < 5 s after a remote commit.
- **Availability:** 99.99% SLO for file reads (downloads); 99.9% for writes and sync.
- **Consistency:** Strong read-your-writes per user after a successful commit. Cross-device sync is eventually consistent. Block-existence checks during dedup must be strongly consistent — a false "already exists" answer could silently discard a block and corrupt the file.
- **Durability:** 11-nines object durability (triple replication + erasure coding across Availability Zones, matching S3-class storage).
- **Security:** Files encrypted at rest (AES-256) and in transit (TLS 1.3). Block downloads served via short-lived presigned URLs — never via guessable content-hash paths — so that ACL revocation immediately cuts off access.
