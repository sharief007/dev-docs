---
title: 'Instagram / Photo Sharing'
weight: 1
type: docs
---

Instagram is a mobile-first photo sharing platform used by over **two billion people monthly**. A user captures a photo, applies a filter, writes a caption, and posts it; the platform processes and stores the image, then surfaces it in the feeds of every follower within seconds. The challenge is simultaneously operating a **media ingest pipeline** (object storage, async processing, global CDN distribution) and a **social graph engine** (follower relationships, ranked feeds, likes, comments, notifications) at a scale where a single viral post can trigger tens of millions of fan-out writes.

This design focuses on the photo-sharing core: **upload → process → store → deliver to follower feeds**. It sits among the most infrastructure-intensive system-design categories because it couples high write-QPS social graph updates with petabyte-scale media storage and multi-terabit CDN egress — two very different scaling axes that must be solved simultaneously.

## Functional Requirements

1. **Upload photo:** A user uploads a photo with a caption and optional tags. The photo is stored and published to their profile and followers' feeds.
2. **Display feed:** A user sees a personalised (or chronological) feed of photos from people they follow, paginated and refreshable.
3. **Follow / unfollow:** Users can follow or unfollow any other user; the feed updates accordingly.
4. **Like / comment:** Users can like or comment on any photo; both counts are visible to all viewers in near real time.
5. **Profile page:** View any user's photo grid in reverse-chronological order.
6. **Explore / trending:** Discover popular or algorithmically curated content beyond the follow graph.
7. **Notifications:** Receive alerts when someone likes or comments on your photo, or starts following you.

## Out of Scope

- Stories, Reels, and live video streaming (video transcoding is a separate, deeper problem).
- Direct messages and group chats.
- Shopping, product tagging, and in-app purchases.
- Ads delivery and monetisation pipeline.
- Advanced content moderation and policy enforcement (assume a separate ML safety pipeline).
- Account creation, login, OAuth, and 2FA (assume an upstream identity/auth service handles these).

## Non-Functional Requirements

- **Scale:** 500M DAU; **100M photos uploaded per day** (~1,160 writes/s average, 3× peak ~3,480/s); **2.5B feed opens per day** (~28,900 reads/s average, 3× peak ~86,700/s).
- **Latency:** Feed load p99 **< 200 ms** (metadata + CDN URL list, not photo bytes); photo thumbnail delivery p99 **< 100 ms** (CDN-served from edge); upload acknowledgement p99 **< 2 s** (async processing continues after ack).
- **Availability:** **99.9%** for feed reads and photo delivery; upload can tolerate 99.5% (client-retryable).
- **Consistency:** Feed is **eventually consistent** — a new post appearing in all followers' feeds within 30 s is acceptable. Like and comment counts may lag by seconds. Follow/unfollow must be **read-your-writes consistent** (the user sees the effect of their own action immediately on their device).
- **Durability:** Original uploaded photos must never be lost (**11 nines** via object storage with cross-region replication). Processed thumbnails and display variants are regenerable from the original.
- **Media scale:** Petabyte-class storage; all media files live in object storage, not databases. CDN egress is the dominant cost driver.
