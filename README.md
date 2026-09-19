# MessOps — Hostel Mess Management Platform

A microservices-based dining management platform for college hostels, built on 3 principles — **minimise food waste, ensure financial transparency**, and **enable democratic campus feedback**.

Hostel mess systems typically run on three blind spots:
- **The kitchen doesn't know real headcount** — so it either overcooks (waste) or undercooks (shortages) relative to full occupancy.
- **Students have no visibility into mess finances** — fees go in, nobody sees where they go. Mess expenses have zero transparency.
- **There's no real feedback loop** — quality complaints go nowhere actionable.

MessOps is built around closing those three gaps: **accuracy** (headcount prediction), **transparency** (a visible financial ledger), and **voice** (a community feedback board).

## Architecture

Four independent Express services, each with its own MongoDB database, plus a React/Redux frontend. The frontend calls each service directly over its own base URL.

```
                        ┌─────────────────────┐
                        │   React + Redux      │
                        │   (Vite, Tailwind)    │
                        └──────────┬───────────┘
              ┌───────────┬────────┼────────┬───────────┐
              ▼           ▼        ▼        ▼           ▼
        ┌──────────┐ ┌─────────┐ ┌────────────┐ ┌───────────────┐
        │  user-   │ │  mess-  │ │  finance-  │ │  community-   │
        │ service  │ │ service │ │  service   │ │   service     │
        └────┬─────┘ └────┬────┘ └─────┬──────┘ └──────┬────────┘
             │            │            │               │
          MongoDB      MongoDB      MongoDB          MongoDB
             │            │            │
             │      (gRPC: headcount)  │
             └────────────┘            │
             (REST, JWT forwarded) ────┴──── (REST, JWT forwarded)
```

### `user-service`
Identity and authority for the whole system.
- Registration → email verification (short-lived JWT link) → login issues an access token (10d) and a refresh token (30d).
- Password reset via a 6-digit, expiring OTP.
- `bcrypt`-hashed passwords; JWT payload carries `{ id, role }`, verified independently by every other service via a shared signing secret — no per-request call back to this service needed for verification.
- Profile picture upload via Multer (in-memory buffer) → Cloudinary, streamed rather than buffered as a full base64 string.
- Exposes one internal **gRPC** RPC (`GetUserCount`), used by `mess-service` for headcount math — the one internal call in this system currently using gRPC + Protocol Buffers instead of REST.

### `mess-service`
Menu scheduling and attendance tracking, sharing one database — a deliberate coupling, not an oversight (see [Design notes](#design-notes)).
- **Attendance** uses a sparse, "eating-by-default" model: no document is written unless a student deviates from eating, keeping the collection small. Headcount is computed as `total registered − skips`, since "eating" is never stored directly.
- **Menu** management with per-dish images (Cloudinary) and a per-meal rating system, gated by cutoff windows (can't rate a meal you skipped, can't rate before it's served).
- Calls `user-service`'s gRPC endpoint for the total registered-user count used in headcount prediction.

### `finance-service`
The transparency layer.
- Razorpay integration: order creation → Razorpay Checkout → server-side HMAC-SHA256 signature verification (using `crypto.timingSafeEqual` to avoid timing-attack leakage) before a payment is ever marked `Paid`.
- A separate categorized `Expense` ledger; a financial dashboard built on MongoDB aggregation pipelines (income vs. expense vs. category breakdown).
- A Redis-backed **sliding-window-log rate limiter** (ZSET + WATCH/MULTI/EXEC optimistic locking) on the fee-payment-initiation route, to guard against abuse of an endpoint that calls out to Razorpay and writes to Mongo on every hit.
- Calls `user-service` over plain REST (not gRPC) for bulk fee allocation across all students.

### `community-service`
The feedback/voice layer.
- Posts with likes/dislikes, implemented as atomic `$addToSet`/`$pull` updates so concurrent votes can't race each other, plus embedded comments.
- Calls `user-service` over REST to attach author details to a new post.

### Frontend
React + Redux Toolkit + `redux-persist`, React Router v7. `ProtectedRoute` gates routes client-side by role for UX — the actual authorization boundary is each service's backend middleware, which re-checks role independently regardless of what the UI allows.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React, Redux Toolkit, redux-persist, React Router, Tailwind CSS, Vite |
| Backend | Node.js, Express |
| Databases | MongoDB (Mongoose), one per service |
| Auth | JWT, bcrypt, role-based access control |
| Payments | Razorpay |
| Media | Cloudinary, Multer |
| Inter-service | gRPC + Protocol Buffers (user↔mess headcount), REST (all other inter-service calls) |
| Rate limiting | Redis (ioredis), sliding window log algorithm |
