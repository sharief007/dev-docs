---
title: 'Notification System'
weight: 1
type: docs
---

You are asked to design a large-scale **notification system** — the platform service that other product teams call to deliver a message to a user through the right channel: mobile **push** (APNs/FCM), **email**, **SMS**, or **in-app** (WebSocket). When a ride is booked, a payment fails, or a friend comments, some upstream service emits an event and the notification system takes over: it decides *whether* to notify, *which channel(s)* to use, *renders* the content, *delivers* via third-party providers, and *tracks* the outcome.

This is a classic **fan-out + reliable-delivery** problem. The hard parts are not the happy path but the operational realities: honoring user preferences and quiet hours, deduplicating retries so a user isn't paged five times, surviving flaky third-party providers, and doing all of it at millions of notifications per minute without losing or duplicating messages.

## Functional Requirements

1. **Multi-channel delivery:** send a notification via push, email, SMS, or in-app.
2. **Templating:** render notifications from named templates with per-user variables and localization.
3. **User preferences:** respect per-category opt-outs, per-channel choices, and quiet hours / rate caps.
4. **Delivery tracking:** record sent / delivered / failed / opened status per notification.
5. **Retries & dead-lettering:** retry transient provider failures; dead-letter permanent failures.
6. **Deduplication:** never deliver the same logical notification twice within a window.

## Out of Scope

- Composing the *business* decision to notify (that lives in upstream services; they emit events).
- Rich marketing-campaign scheduling / audience segmentation (a separate campaign product).
- The third-party providers themselves (APNs, FCM, SES, Twilio) — treated as external dependencies.

## Non-Functional Requirements

- **Scale:** 500M notifications/day (~5,800/s average, ~20,000/s peak at ×3.5). Bursts from fan-out events (e.g. a sports goal → 10M pushes in minutes).
- **Latency:** transactional notifications (OTP, security) p99 **< 5 s** end-to-end; non-urgent may be batched within minutes.
- **Availability:** **99.95%** for accepting notification requests. Delivery is best-effort but **at-least-once**.
- **Delivery semantics:** **at-least-once** with idempotent dedup so recipients perceive **effectively-once**.
- **Durability:** an accepted notification must never be silently lost — it is persisted before acknowledgment.
- **Ordering:** best-effort per-user ordering; strict global ordering is **not** required.
