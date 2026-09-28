# H.E.L.P - Hub for Event Logistic & People

Event and volunteer management platform. Organize events, break them down into missions and time slots, manage user availability, and let people register for slots.

Staging: https://staging-mt-event-app.duckdns.org/ - V1 prototype, frozen: it won't evolve further due to current AWS server constraints. Current development runs locally, and the next deployment target is GCP.

## Repositories

| Repo | Role | Stack | Status |
|---|---|---|---|
| [`event-app`](https://github.com/help-platform-event/event-app) | Frontend + Gateway (public API, events, missions, slots, participation) | React/TypeScript, NestJS, Prisma, MySQL, kafkajs | Local dev (V1 on staging) |
| [`ms-auth-java`](https://github.com/help-platform-event/ms-auth-java) | Auth, users and settings (JWT, refresh tokens, Google OAuth). Has replaced the original NestJS auth service | Spring Boot, JPA/Hibernate, Flyway, MySQL, Kafka | Local dev |
| `ms-notification-java` | Notifications: consumes Kafka events, sends emails (in-app next) | Spring Boot, Spring Kafka, JPA, MySQL, Spring Mail | In progress - private for now |

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
                           ms-notification-java ──► email (in-app next)
```

- **Synchronous calls go over HTTP.** The Gateway calls `ms-auth-java` and verifies its JWTs locally. This replaced the original NATS request/reply setup, which is gone, along with the NestJS auth service and its MongoDB.
- **Events go over Kafka, fire-and-forget.** A service publishes what happened (a user registered, changed their settings, asked to join a slot, was accepted...) after its database transaction commits. Consumers react without calling the publisher back.
- **`ms-notification-java`** keeps its own copy of each user's email and notification preferences, built from those events. It only notifies a user if their preferences allow it. This is where the Kafka concepts NATS doesn't offer come in: consumer groups, offset replay to rebuild state, a compacted topic, retries with a dead-letter topic, and idempotent consumers.

## Stack overview

- **Backend (TypeScript):** NestJS, Prisma, MySQL
- **Backend (Java):** Java 21, Spring Boot 4, JPA/Hibernate, Flyway, MySQL, Spring Security
- **Messaging:** Kafka (Spring Kafka, kafkajs)
- **Frontend:** React, TypeScript, TanStack Query, shadcn/ui
- **Testing:** Jest, JUnit 5, Testcontainers (real MySQL + Kafka)
- **Infra:** Docker / Docker Compose (the whole stack in one command), GitHub Actions, GHCR. V1 on AWS EC2; next: GCP (Terraform), with GraalVM native images for the Java services.
