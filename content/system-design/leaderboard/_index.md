---
title: 'Leaderboard'
weight: 1
type: docs
---

Every competitive game — from mobile titles like PUBG Mobile and Clash of Clans to web-scale esports tournaments — needs a leaderboard: a ranked list of millions of players that updates in real time. When a match ends, a player's score is written, and within seconds the global top-100 board and the "what is my rank?" query must both reflect the change. The problem sounds simple — "sort by score" — but at scale it becomes anything but: a SQL `SELECT COUNT(*) WHERE score > ?` on a 50-million-row table takes hundreds of milliseconds under write load, which is impractical as a hot read path with a 10 ms latency target.

The interesting design space lives at the intersection of the right data structure (Redis sorted sets), the right write path (synchronous vs. Kafka-buffered), and two genuinely hard sub-problems: **tie-breaking** (when thousands of players share the same integer score) and **sharding** (how to compute a global rank when no single Redis instance can hold all 50 million members and serve 250,000 reads per second).

## Functional Requirements

1. **Update score**: Accept a score event for a user; maintain their best (or cumulative) score on the board.
2. **Global top-K**: Return the top *K* players (e.g. top 100) with their rank, username, and score.
3. **My rank**: Given a user ID, return their current rank and score.
4. **Nearby ranks**: Given a user ID, return the *N* players immediately above and below them.
5. **Segmented leaderboards**: Maintain separate ranked boards per country, per friend-group, and per time window (daily, weekly, all-time).

## Out of Scope

- Score validation and anti-cheat pipelines (scores arrive pre-computed from game servers).
- Authentication and authorisation (handled by an upstream API gateway).
- Rich player profiles, avatars, or social graphs beyond name and country.
- Push notifications when a player's rank changes (can layer on top via a separate notification service).

## Non-Functional Requirements

- **Scale:** 50 M active players; **5,000 score writes/s** sustained; **50,000 rank reads/s** sustained.
- **Peak factor:** 5× during live tournaments → **25,000 writes/s**, **250,000 reads/s**.
- **Latency:** top-K and my-rank reads p99 **< 10 ms**; score write acknowledgment p99 **< 50 ms**.
- **Availability:** 99.9% for reads; 99.5% for writes (a brief write outage is recoverable by replaying the event log).
- **Consistency:** near-real-time — rank updates must be visible within a few seconds of a score event; minor staleness is acceptable in friend-group boards.
- **Durability:** no score loss across node failures; durable persistence required before acknowledgment.
- **Memory budget:** the global sorted set for 50 M players must fit within a Redis cluster (target ≤ 60 GB total per board including replicas).
