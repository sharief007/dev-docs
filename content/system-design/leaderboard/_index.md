---
title: ''
weight: 1
type: docs
toc: true
sidebar:
  open: true
prev: 
next: high-level-design
params:
  editURL: 
---

Imagine you are tasked with designing a real-time leaderboard system for a massive multiplayer online game. This system needs to track player scores from millions of matches, provide players with their live ranking instantly, and be highly available and scalable to handle peak tournament traffic.

The user gets a point when they win a match. We can go with a simple point system in which each user has a score associated with them. Each time the user wins a match, we should add a point to their total score. Each month, a new tournament kicks off which starts a new leaderboard.

**Data Volume:**

* Up to **25 million Monthly Active Users (MAU)** participating in a tournament.
* An average of **5 million Daily Active Users (DAU)**, with each user playing approximately 10 matches per day.
* Each monthly leaderboard will contain entries for all 25 million MAU.

**Functional Requirements:**

The system must support the following primary queries for any given monthly tournament:

1.  **Top N Players:**
    * Return the **top 10 players** (showing `user_id`, `score`, and `rank`) on the current leaderboard.
2.  **Specific User Rank:**
    * For a given `user_id`, return that user's current **rank and total score**.
3.  **Surrounding Ranks (Player Context View):**
    * *(Bonus Requirement)* For a given `user_id`, return a list of players who are **four places above and four places below** that user, including the user themselves (a total of 9 players). The response should include each player's `user_id`, `score`, and `rank`.
4.  **Tie-Breaking:**
    * If two players have the same score, they are considered to have the same rank. A mechanism to break ties (e.g., using `win_timestamp` to favor the player who reached the score first) can be considered.

**Non-Functional Requirements:**

1.  **Latency:**
    * **Real-Time Updates:** The end-to-end latency from a "Match Win" event to the updated rank being queryable must be minimal (seconds, not minutes). A batched or delayed history of results is not acceptable.
    * **Read Performance:** Queries for the top 10, user rank, and surrounding ranks must be served with very low latency (sub-200ms).
2.  **Correctness:**
    * The score for each user must be an accurate sum of their wins. The system must not lose or double-count win events.
3.  **Scalability:**
    * The system must be horizontally scalable to handle the current and future growth of the user base and match volume without performance degradation.
4.  **Reliability & Resilience:**
    * **High Availability:** The leaderboard service must be highly available. Failure of a single component should not lead to system downtime or data loss.
    * **Data Integrity:** The system must be resilient to transient failures in downstream components (e.g., the leaderboard data store). Score updates should not be lost if a dependency is temporarily unavailable.