---
title: 'WhatsApp / Chat System'
weight: 1
type: docs
---

WhatsApp serves over 2 billion users who exchange text, media, and calls daily. The system's defining challenge is not storage — a text message is a few hundred bytes — but **connection management and real-time routing**: maintaining persistent channels to billions of mobile devices over unreliable cellular networks, routing each message to the correct recipient within a second, and fanning a single group message out to hundreds of members without compounding latency at every hop. Layered on top is the requirement that the server *itself* cannot read any message content, making end-to-end encryption a first-class architectural constraint rather than an add-on.

This case study covers 1-on-1 messaging with three-state delivery receipts, group chat with server-side fan-out, media sharing, and online/offline presence. A dedicated [encryption deep dive](encryption-deep-dive) explains the **Signal protocol** (X3DH key exchange + Double Ratchet) at the precision level expected in a senior system design interview, because the encryption architecture changes several fundamental design decisions — from the data model (ciphertext columns, per-device payloads) to the multi-device sync story.

## Functional Requirements

1. **1-on-1 messaging:** Send and receive text messages between two users. Delivery is reliable (at-least-once, deduplicated at the receiver) and in-order per conversation.
2. **Delivery receipts:** Three visible states — **Sent** (server persisted and acknowledged), **Delivered** (message reached the recipient's device), **Read** (recipient opened the conversation thread).
3. **Group chat:** Create groups of up to 1,000 members. A message sent to the group is delivered to every current member. Members can be added and removed by admins.
4. **Media sharing:** Send images, video clips, audio recordings, and documents. Large files are stored in object storage; the message record holds only a URL and integrity checksum.
5. **Presence / last-seen:** Display whether a contact is currently online; if offline, show the timestamp of their last activity.
6. **End-to-end encryption (E2EE):** All messages are encrypted on the sender's device before transmission. The server stores and forwards only ciphertext and never holds keys sufficient to decrypt message content.

## Out of Scope

- Voice and video calls (WebRTC signalling and media relay; entirely separate subsystem).
- Stories / status updates (ephemeral media pipeline with different retention semantics).
- In-app payments.
- Message search across history (requires on-device full-text indexing; incompatible with server-side E2EE).
- Spam and abuse detection (must operate on metadata and client-reported signals only, due to E2EE).
- User registration and phone-number verification.
- Business messaging API and broadcast lists.

## Non-Functional Requirements

- **Scale:** 2B registered users; 100M daily active users (DAU); ~50B messages/day; ~15M concurrent WebSocket connections at peak.
- **Latency:** Online-to-online message delivery (same region) p99 < 500 ms; p50 < 100 ms. Presence update visible to active subscribers < 1 s. Media upload start p99 < 200 ms.
- **Availability:** 99.99% for the message send/deliver path (~52 min downtime/year). Presence is best-effort and may serve stale data under partial degradation.
- **Consistency:** Per-conversation **FIFO ordering** guaranteed. Cross-conversation ordering is not required. Read-your-own-writes: a sender sees their own message immediately after the Sent receipt arrives.
- **Durability:** Messages durably stored server-side for 30 days (sufficient for device recovery and re-sync). Beyond 30 days, client devices own their history. With E2EE, the server cannot reconstruct decrypted history for any retention period.
- **Security:** End-to-end encrypted using the Signal protocol (X3DH + Double Ratchet). Private keys are generated on-device and never transmitted. The server is zero-knowledge with respect to message plaintext.
