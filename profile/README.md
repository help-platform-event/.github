# H.E.L.P - Hub for Event Logistic & People

Event and volunteer management platform. Organize events, break them down into missions and time slots, manage user availability, and let people register for slots.

Staging: https://staging-mt-event-app.duckdns.org/ - V1 prototype, won't evolve further due to current AWS server constraints.

## Repositories

| Repo | Role | Stack | Status |
|---|---|---|---|
| [`event-app`](https://github.com/help-platform-event/event-app) | Frontend + Gateway + ms-auth (NestJS) | NestJS, React/TypeScript, Prisma, Mongoose, NATS | Deployed (staging, AWS EC2) |
| `ms-auth-java` | Auth service rewrite - Java/Spring Boot | Spring Boot, JPA/Hibernate, MySQL, Kafka | Local dev - private for now |

## Why two repos

Started as a single monorepo (pnpm + Turborepo). `ms-auth-java` was split out into its own repo once the toolchain diverged (Maven vs pnpm) - no shared TypeScript contracts, independent CI needs. Each service stays deployable and testable on its own.

## Architecture

Services communicate over NATS (request/reply) in the current production setup. The Java rewrite of `ms-auth` explores Kafka as an alternative messaging layer, as a comparative learning exercise (event-driven patterns, offset management, replay - concepts NATS doesn't cover).

## Stack overview

- **Backend:** NestJS, TypeScript, Prisma, Mongoose, NATS
- **Backend (Java, in progress):** Spring Boot, JPA/Hibernate, Kafka
- **Frontend:** React, TypeScript, TanStack Query, shadcn/ui
- **Infra:** Docker, GitHub Actions, AWS EC2, GHCR
