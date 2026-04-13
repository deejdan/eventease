# EventEase

A multi-tenant event management platform built to streamline the full lifecycle of any event — from planning and guest management, through invitations and RSVPs, to event day check-in and post-event follow-up.

Built for real use. Designed for architectural clarity.

---

## The Problem

Event management today is fragmented and stressful. Organizers use one tool to track guests, another to send invitations, a manual process to handle RSVPs, and something ad hoc on event day to manage check-ins. Nothing talks to each other. Data gets out of sync. Things fall through the cracks.

EventEase is the single system of record for the entire event lifecycle — from the first guest added to the last check-in scanned.

---

## Features (v1)

- **Tenant onboarding** — organization signup with bitwise permission model for team access
- **Magic link auth** — no passwords; users and guests both authenticate via link
- **Event management** — create and manage events with full lifecycle tracking
- **Venue management** — reusable, tenant-scoped venues with full address and coordinates
- **Guest list management** — manual guest entry with dietary need capture
- **Plus-one support** — every plus-one is a named guest with their own invitation and QR code
- **Invitations** — create and deliver invitations via email and magic link
- **QR code generation** — every attendee gets a unique, cryptographically opaque QR code
- **RSVP flow** — guests confirm or decline via their magic link; can change their mind
- **Bounce tracking** — organizers see which invitations failed delivery
- **Event day check-in** — staff scans QR codes to validate and check in guests

---

## Tech Stack

| Layer           | Choice               | Reason                                         |
| --------------- | -------------------- | ---------------------------------------------- |
| Runtime         | Node.js              | —                                              |
| Language        | TypeScript           | Type safety, portfolio clarity                 |
| HTTP Framework  | Fastify              | Lightweight, fast, excellent plugin system     |
| ORM             | Prisma               | Seamless SQLite (dev) → PostgreSQL (prod) swap |
| Database (dev)  | SQLite               | Zero setup, in-process                         |
| Database (prod) | PostgreSQL           | Scalable, reliable                             |
| Auth (users)    | Magic link           | No passwords stored                            |
| Auth (guests)   | Magic link / QR code | Guests should not need accounts                |

---

## Architecture

EventEase follows **Hexagonal Architecture** (Ports & Adapters) organized as a **Modular Monolith**. Each domain concept is a self-contained module with its own internal layers, unified by an `index.ts` as its public interface.

```
src/
├── modules/
│   ├── tenant/
│   │   ├── domain.ts         — Entity + Repository interface (port)
│   │   ├── application.ts    — Use cases
│   │   ├── infra.ts          — Prisma repository (secondary adapter)
│   │   ├── http.ts           — Routes + Fastify schemas (primary adapter)
│   │   ├── container.ts      — Wiring for this module
│   │   └── index.ts          — Public interface
│   ├── event/
│   ├── venue/
│   ├── guest/
│   ├── invitation/
│   └── auth/
│
├── shared/                   — Cross-cutting types and ports only
│   ├── permissions.ts        — EventPermission + TenantPermission constants
│   └── ports/
│       └── PermissionManager.ts
├── infrastructure/           — Shared infrastructure (Prisma client, email, QR, permissions)
├── plugins/                  — Fastify plugins (auth, multitenancy)
├── app.ts                    — Registers all routes and plugins
└── server.ts                 — Boots the server
```

### Core Principles

- **Dependency rule** — dependencies point inward. Domain never imports from infrastructure or HTTP.
- **Tenant scoping at the repository layer** — domain entities never think about tenants.
- **Invitation as the load-bearing connector** — owns the delivery token; RSVP and check-in state live in their own models.
- **Module boundaries via `index.ts`** — modules never reach into each other's internals.
- **Permission manager** — all authorization checks go through a single domain service; no implicit access, even for admins.

### Key Domain Concepts

| Concept      | Role                                                                          |
| ------------ | ----------------------------------------------------------------------------- |
| Tenant       | Organization using the platform. All data is scoped to a tenant.              |
| Event        | Central container. Everything lives inside an event.                          |
| Venue        | Reusable, tenant-scoped location with address and coordinates.                |
| Guest        | Pure identity — name, email, phone. Tenant-scoped, reusable across events.    |
| EventGuest   | Roster entry — places a guest on an event's list before any invite is sent.   |
| Invitation   | Delivery record for a roster entry. Owns token and delivery timestamps.       |
| RsvpResponse | Guest's response — accepted/declined, noteToOrganizer, noteToHost, dietary snapshot. |
| CheckIn      | Attendance record — who checked in the guest and when. Terminal.              |
| PlusOne      | Links a plus-one guest to their host. Guest stays unaware of this concept.    |

### Invitation State (Derived)

Invitation has no status field. State is derived from related records:

| State | Condition |
|---|---|
| Pending | No sentAt |
| Sent | sentAt set, no openedAt, no bouncedAt |
| Opened | openedAt set |
| Bounced | bouncedAt set |
| RSVP'd | RsvpResponse exists |
| Checked in | CheckIn record exists (terminal) |

### Event State Machine

```
PLANNING → ONGOING → COMPLETED (terminal)
         ↘          ↗
          CANCELLED (reversible to PLANNING)
```

---

## Roadmap

### v1 — Core (current)

Support a real wedding end-to-end.

### v2 — Intelligence

Social auth, AI-assisted seating, real-time event day dashboard, sub-event tracking, offline check-in, SMS delivery.

### v3 — Post-Event Automation

Attendance reports, AI-written thank you messages, feedback surveys, digital guest book.

### v4 — SaaS

Public signup, billing tiers, white-label options, multi-event analytics.

---

## Non-Negotiable Invariants

- A checked-in guest can never be un-checked-in
- An invitation token must always be unique and cryptographically opaque
- A guest from one tenant must never be visible to another tenant
- The OWNER of a tenant cannot be deleted or demoted by anyone, including themselves
- Every attendee at the venue must have a named invitation and a valid QR code — no anonymous attendees
- No user gains event access without an explicit EventMembership record — no implicit bypass

---

## Project Goals

This project is built with three goals in mind:

1. **Personal** — used successfully for the creator's own wedding
2. **Portfolio** — a living reference for hexagonal architecture, domain modeling, multi-tenancy, and production system thinking
3. **Commercial** — a foundation with a clear path toward a SaaS product

---

## Development Setup

> Coming soon — Fastify scaffold, Prisma schema, and local SQLite setup.
