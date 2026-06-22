# ResyBot

**NOTE**

This is the public version of the private repo titled "Job Tool" – to act as a landing page for the public.

It contains only the README of the original repo. If you wish to see the actual code, contact me. 


## 

## Overview

An automated restaurant reservation bot that books tables through the [Resy](https://resy.com) API the instant reservations open, then manages the lifecycle of those bookings — including scheduled cancellations and optional resale via [AppointmentTrader](https://appointmenttrader.com).

---

## How It Works

Restaurants on Resy release reservations at a fixed time each day (typically 10:00 AM). ResyBot wakes up at exactly that moment and books tables across multiple user accounts and party-size configurations concurrently, maximising the chance of securing a coveted slot.

### End-to-End Flow

```
Cloud Scheduler (daily)
    └─► SetupBookVenueTasks (Node.js)
            └─► Cloud Tasks  ──scheduled at bookTime──►  BookVenueWrapper (Node.js)
                                                                └─► BookVenueGO (Go)  ◄── Resy API
                                                                        └─► HandleMadeBooking
                                                                                ├─► Firestore (store reservation)
                                                                                └─► Cloud Tasks (schedule CancelResy)
```

1. **`SetupBookVenueTasks`** — runs daily via Cloud Scheduler. Reads all enabled venues from Firestore and creates a Cloud Task per venue, scheduled to fire at the venue's configured `bookTime`.
2. **`BookVenueWrapper`** — triggered by a Cloud Task at booking time. Fetches venue configuration and enabled user accounts from Firestore, then calls the Go service via authenticated HTTP.
3. **`BookVenueGO`** — the speed-critical Go service. Sleeps until the exact booking second, then fans out goroutines to book every combination of user × table option concurrently against the Resy API.
4. **`HandleMadeBooking`** — runs inline in `BookVenueWrapper` after the Go service responds. Persists each successful reservation to Firestore and queues a future cancellation task.
5. **`CancelResy`** — triggered by a Cloud Task at cancellation time. Checks AppointmentTrader status before cancelling: skips if the reservation sold, removes the listing if it didn't, then cancels via the Resy API.
6. **`UpdateListings`** — periodically re-prices AppointmentTrader listings to stay competitive.

### Resy API Booking (Two-Step)

The Resy API requires two sequential calls to complete a booking:

| Step | Endpoint | Purpose |
|------|----------|---------|
| 1 | `POST /3/details` (`commit: 1`) | Obtains a short-lived `book_token` |
| 2 | `POST /3/book` (form-encoded) | Submits the reservation; returns a `resy_token` |

---

## Architecture

The system is split into two services deployed as **Google Cloud Functions (Gen 2)** in region `us-east4`.

### `go-service` — Booking Engine

| File | Role |
|------|------|
| `BookVenue/BookVenue.go` | Core booking logic — concurrent across users & table options |
| `resyAPI/resyAPI.go` | Typed Resy API client (find slots, get details, book, cancel) |
| `main.go` | HTTP handler (`BookVenueHTTP`) — Cloud Function entry point |
| `types/` | Shared Go types and typed error handling |
| `util/dateTime/` | `SleepUntil` — precision sleep to the exact booking second |
| `scripts/deploy_bookvenue.sh` | Deploys `BookVenueGO` to GCP |

Design priorities: **speed** (zerolog, goroutines, no unnecessary allocations) and **correctness** (typed errors with HTTP status codes, validated inputs).

### `node-service` — Orchestration & Lifecycle

| File | Role |
|------|------|
| `functions/SetupTasks.ts` | Daily setup of per-venue Cloud Tasks |
| `functions/BookVenueWrapper.ts` | Fetches Firestore data, calls Go service, processes results |
| `functions/CancelResy.ts` | Cancels reservations via Resy API (AT-aware) |
| `functions/UpdateListings.ts` | Competitive price updates on AppointmentTrader |
| `shared/FirestoreAPI.ts` | Typed `FirestoreDB` singleton with Zod-validated collections |
| `shared/ResyAPI/` | Node.js Resy API client with Zod response validation |
| `shared/AppointmentTraderAPI/` | Generated AppointmentTrader API client |
| `shared/TaskQueue/` | Cloud Tasks helpers (book queue, cancel queue) |
| `shared/types/schemas.ts` | Zod schemas → TypeScript types (single source of truth) |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Booking engine | **Go 1.23** |
| Orchestration & lifecycle | **TypeScript / Node.js 20** |
| Logging | **zerolog** (structured, high-performance) |
| Schema validation | **Zod** (runtime validation + TypeScript type inference) |
| Database | **Google Cloud Firestore** |
| Task scheduling | **Google Cloud Tasks** |
| Cron trigger | **Google Cloud Scheduler** |
| Compute | **Google Cloud Functions Gen 2** |
| Auth | **google-auth-library** (service-to-service ID tokens) |
| Resale marketplace | **AppointmentTrader API** |

---

## Data Model

| Collection | Key fields |
|------------|-----------|
| `venues` | `id`, `name`, `enabled`, `bookTime`, `daysAhead`, `tableOptions[]` |
| `users` | `user_num`, `email`, `auth_token`, `payment_id`, `enabled` |
| `reservations` | `user_num`, `resData` (Resy token, reservation ID…), `ATmetadata` |

---
