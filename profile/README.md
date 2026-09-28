# H.E.L.P - Hub for Event Logistic & People

Event and volunteer management platform. Organize events, break them down into missions and time slots, manage user availability, and let people register for slots.

Staging: https://staging-mt-event-app.duckdns.org/ - V1 prototype, frozen: it won't evolve further due to current AWS server constraints. Current development runs locally, and the next deployment target is GCP.

## Repositories

| Repo | Role | Stack | Status |
|---|---|---|---|
| [`event-app`](https://github.com/help-platform-event/event-app) | Frontend + Gateway (public API, events, missions, slots, participation) | React/TypeScript, NestJS, Prisma, MySQL, kafkajs | Local dev (V1 on staging) |
| [`ms-auth-java`](https://github.com/help-platform-event/ms-auth-java) | Auth, users and settings (JWT, refresh tokens, Google OAuth). Has replaced the original NestJS auth service | Spring Boot, JPA/Hibernate, Flyway, MySQL, Kafka | Local dev |
| [`ms-notification-java`](https://github.com/help-platform-event/ms-notification-java) | Notifications: consumes Kafka events, sends emails and in-app notifications (the bell) | Spring Boot, Spring Kafka, JPA, MySQL, Spring Mail | Local dev |

Each repo's README describes its own service: what it does, and how to run and test it on its own. This page covers the whole platform.

## Why several repos

Started as a single monorepo (pnpm + Turborepo). The Java services live in their own repos because their toolchain diverged (Maven vs pnpm) and they need independent CI: each one stays buildable, testable and deployable on its own. `event-app` keeps the TypeScript side together (Front, Gateway, shared contracts). The Java services don't share code with it: each keeps its own copy of the HTTP and Kafka contracts it relies on.

## Architecture

```
Front ──HTTP──► Gateway (NestJS) ──HTTP──► ms-auth-java
                     │                          │
                     │ event.participation.*    │ auth.*
                     ▼                          ▼
                   ─────────────── Kafka ───────────────
                                     │
                                     ▼
                           ms-notification-java ──► email + in-app (polled by the Front's bell via the Gateway)
```

- **Synchronous calls go over HTTP.** The Gateway calls `ms-auth-java` and verifies its JWTs locally. This replaced the original NATS request/reply setup, which is gone, along with the NestJS auth service and its MongoDB.
- **Events go over Kafka, fire-and-forget.** A service publishes what happened after its database transaction commits. Consumers react without calling the publisher back.
- **`ms-notification-java`** keeps its own copy of each user's email and notification preferences, built from those events. It only notifies a user if their preferences allow it. This is where the Kafka concepts NATS doesn't offer come in: consumer groups, offset replay to rebuild state, a compacted topic, retries with a dead-letter topic, and idempotent consumers.

### Kafka topics

| Topic | Published by | When | Consumed by |
|---|---|---|---|
| `auth.user.registered` | ms-auth-java | A user signs up (or first Google login) | ms-notification-java: welcome email, user's email |
| `auth.user.settings-changed` | ms-auth-java | A user updates their settings | ms-notification-java: notification preferences |
| `auth.password.changed` | ms-auth-java | A user changes their password | ms-notification-java: security email |
| `event.participation.requested` | Gateway | A volunteer asks to join a slot | ms-notification-java: email to the organizer |
| `event.participation.decided` | Gateway | The organizer accepts or rejects | ms-notification-java: email to the volunteer |
| `event.participation.cancelled` | Gateway | The volunteer or the organizer cancels a participation | ms-notification-java: email to the other party |

Two rules: **whoever publishes a topic declares it** (3 partitions, messages keyed by user id), and **only what a consumer uses is published**.

## Run the whole platform

**Requirements:** Docker with Docker Compose, Node.js + pnpm, and the three repos cloned side by side:

```
some-folder/
├── event-app/
├── ms-auth-java/
└── ms-notification-java/
```

`event-app`'s `docker-compose.dev.yml` includes the two Java repos' `compose.yaml`, so one command starts everything.

1. Create a `.env` at the root of `event-app` (template: `apps/gateway/.env.example`). The compose file already sets the database, service URLs, Kafka and JWT settings. Add:
   - `VITE_GOOGLE_CLIENT_ID` and `VITE_GEOAPIFY_API_KEY` (passed to the Front at build time);
   - `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` for Google sign-in, and `GEOAPIFY_API_KEY` for geocoding.
2. From `event-app`:

```bash
pnpm stack:up      # build + start everything
pnpm stack:ps      # status
pnpm stack:logs    # follow logs
pnpm stack:down    # stop
pnpm stack:reset   # stop and wipe volumes (fresh databases)
```

| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Gateway API (Swagger at `/api`) | http://localhost:3000 |
| ms-auth-java | http://localhost:8080 |
| ms-notification-java | http://localhost:8085/actuator/health |
| Mailpit (every email sent) | http://localhost:8025 |
| Kafka UI (topics, messages, consumer groups) | http://localhost:8082 |
| phpMyAdmin (Gateway DB) | http://localhost:8083 |
| Adminer (auth DB, server `mysql`) | http://localhost:8081 |
| Kafka | `localhost:9094` from the host, `kafka:29092` from containers |

To see Kafka at work (consumer lag, catch-up, preferences, retries and dead-letter topic, replay), follow the walkthrough in `ms-notification-java`'s README.

## Stack overview

- **Backend (TypeScript):** NestJS, Prisma, MySQL
- **Backend (Java):** Java 21, Spring Boot 4, JPA/Hibernate, Flyway, MySQL, Spring Security
- **Messaging:** Kafka (Spring Kafka, kafkajs)
- **Frontend:** React, TypeScript, TanStack Query, shadcn/ui
- **Testing:** Jest, JUnit 5, Testcontainers (real MySQL + Kafka)
- **Infra:** Docker / Docker Compose (the whole stack in one command), GitHub Actions, GHCR. V1 on AWS EC2; next: GCP (Terraform), with GraalVM native images for the Java services.

## Status

- Done: auth moved to Java, NATS removed, notifications (account and security emails; participation emails and in-app notifications in a bell, filtered by user preferences).
- Next: a real-time discussion chat on each event's page (WebSocket, Java), then GraalVM native images, then deployment to GCP.
