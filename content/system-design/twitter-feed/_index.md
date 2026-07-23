---
title: 'Twitter Feed / Home Timeline'
weight: 1
type: docs
---

Every time you open Twitter/X, your home timeline loads tweets from hundreds of accounts you follow, ordered to surface the most relevant content first. What looks like a simple list of posts is one of the hardest system-design problems at scale: you must deliver fresh, personalised timelines to **300 million daily active users** in under 100 ms, while ingesting 150 million new tweets per day and dealing with a social graph where some accounts have 100 million followers.

The core challenge is the **fan-out problem**: when a user posts a tweet, do you immediately precompute and push it into every follower's timeline (fast to read, expensive to write), or do you let each user pull the feeds they care about at read time (cheap to write, expensive to read)? Getting this trade-off wrong either collapses under celebrity traffic or kills read latency. The production answer, as this design shows, is a **hybrid** that applies each strategy where it works.

## Functional Requirements

1. **Post tweet:** A logged-in user can create a tweet (up to 280 characters, with optional media attachments).
2. **Home timeline:** A user sees a paginated feed of recent tweets from accounts they follow; initial load returns up to 20 tweets with a cursor for infinite scroll.
3. **Follow / unfollow:** A user can follow or unfollow any account; the timeline reflects the change within seconds.
4. **Likes and retweets:** Users can engage with tweets; engagement counts are visible and eventually consistent.
5. **Delete tweet:** A user can delete their own tweet; it must disappear from all visible timelines within a few seconds.
6. **Tweet edits:** A user may edit a tweet within a time window (e.g. 30 minutes); the edited text propagates to displayed timelines.

## Out of Scope

- Direct messages — a separate real-time messaging subsystem.
- Full-text tweet search and trending topics.
- Twitter Spaces / live audio rooms.
- Ads targeting and insertion into the timeline.
- Notification delivery (covered separately in [Notification Fan-out]({{% relref "/design-concepts/specialized/notification-fanout" %}})).
- User authentication and OAuth session management.
- Rate limiting and abuse prevention (assume an upstream API gateway handles these).

## Non-Functional Requirements

- **Scale:** 300 M DAU; 150 M tweets/day; average user follows ~200 accounts; follower distribution follows a **power law** — most accounts have fewer than 500 followers while top celebrities exceed 100 M.
- **Latency:** Timeline read p99 < 100 ms; tweet post acknowledgement p99 < 200 ms; fan-out delivery within 5 s for regular users.
- **Availability:** 99.99 % for timeline reads; 99.9 % for tweet writes.
- **Consistency:** Timeline is **eventually consistent** — a new tweet may appear in followers' timelines within ~5 s. Chronological/score order within a user's timeline must be stable. Engagement counts (likes, retweets) are approximate and eventually consistent.
- **Durability:** Tweets must not be lost after a successful write acknowledgement. Deleted tweets must not reappear after propagation completes.
- **Fan-out budget:** Write amplification must be bounded; no single tweet (including a celebrity with 100 M followers) should create unbounded write pressure on the storage tier.
