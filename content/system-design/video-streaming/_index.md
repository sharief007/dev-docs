---
title: 'YouTube / Video Streaming'
weight: 1
type: docs
---

YouTube processes over **500 hours of video uploaded every minute** and delivers more than **1 billion hours of watchtime every day**. Behind that scale sits a multi-stage pipeline: a creator uploads a raw video file, the system chunks it, transcodes it into a ladder of resolutions and bitrates, stores thousands of immutable segments in object storage, and delivers them globally through a massive CDN. On playback, the client player dynamically switches quality level per segment based on available bandwidth — seamlessly, with no rebuffering on a good connection.

The challenge is that video has two completely different traffic patterns: **upload and transcoding** (write-heavy, compute-intensive, latency-tolerant — a video can take minutes to process) vs. **streaming and delivery** (read-extremely-heavy, latency-sensitive, dominated by CDN egress at tens of terabits per second). A good design separates these planes entirely and optimises each independently. The transcoding pipeline is the engineering heart of the upload path; adaptive bitrate streaming is the engineering heart of the playback path.

## Functional Requirements

1. **Upload:** Users upload video files of arbitrary length (seconds to hours). Uploads must be resumable after interruption (large files over unstable connections).
2. **Transcode:** The system automatically transcodes each uploaded video into a rendition ladder (240p, 360p, 480p, 720p, 1080p, 4K) and packages the output for adaptive streaming.
3. **Stream:** Users play videos with adaptive bitrate (ABR) streaming via HLS or DASH. The player automatically selects the best quality rendition for the current bandwidth, switching per segment.
4. **Global delivery:** Video segments are served from CDN edge nodes close to the viewer; playback must start quickly with no initial buffering.
5. **View counts:** The system tracks approximate view counts per video.
6. **Likes and comments:** Users can like videos and post or read comments at scale.
7. **Search and feed:** Users can search videos by title or tag and browse a subscription or home feed.

## Out of Scope

- Live streaming (fundamentally different latency model and packaging pipeline).
- Recommendation engine and personalised feed ML (separate ML platform; see {{% relref "/design-concepts/ml/recommendation-systems" %}}).
- Ad serving and monetization.
- Content moderation, copyright fingerprinting, and CSAM scanning (separate async pipelines).
- Caption and subtitle generation.
- User authentication and billing (upstream API gateway handles identity).

## Non-Functional Requirements

- **Scale:** 2 billion logged-in users; **500 hours of video uploaded per minute**; **1 billion hours of video watched per day**; ~180,000 new videos published per day.
- **Streaming latency:** Playback start (time-to-first-segment) **< 3 s** globally; segment delivery p99 **< 500 ms** at the CDN edge.
- **Upload:** Support files up to 128 GB; resumable chunked upload protocol; transcoding begins as soon as the upload is complete.
- **Availability:** **99.99%** for video playback (a dead stream is immediately visible to users); **99.9%** for upload ingestion (a failed upload can be retried).
- **Consistency:** Video metadata is **eventually consistent** (a new video may take minutes to appear in search). View and like counts are **approximate** (within a small margin is acceptable). Segment data is **immutable** once written — no consistency challenge once published.
- **Durability:** **11 nines** for stored video segments, achieved via multi-region object storage replication.
- **Storage growth:** ~400 TB/day of new segments across all renditions; ~150 PB/year.

## Terminology

| Term | Meaning |
|---|---|
| **Rendition** | A specific quality variant of a video — e.g. 720p at 2.5 Mbps |
| **Segment / chunk** | A short, independently decodable piece of a rendition (typically 2–10 s) |
| **GOP** | Group of Pictures — an IDR keyframe followed by its dependent P/B-frames |
| **HLS** | HTTP Live Streaming — Apple's adaptive streaming protocol using `.m3u8` manifests |
| **DASH** | Dynamic Adaptive Streaming over HTTP — the MPEG open standard using `.mpd` manifests |
| **ABR** | Adaptive Bitrate — the player algorithm that selects a rendition per segment based on bandwidth |
| **Manifest / Playlist** | A text file listing available renditions and segment URLs |
| **CDN** | Content Delivery Network — distributed edge cache serving segments close to viewers |
| **Transcoding** | Re-encoding a source video to a target codec, resolution, and bitrate |
