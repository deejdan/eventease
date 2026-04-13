# EventEase — Architecture

## Overview

EventEase follows **Hexagonal Architecture** (Ports & Adapters), with influence from Clean Architecture for organizing the core. The goal is a codebase where business rules are isolated, infrastructure is swappable, and every layer has a single, justifiable responsibility.

---

## Core Principle: The Dependency Rule

Dependencies only point **inward**. Outer layers know about inner layers. Inner layers never know about outer layers.

```
http / cli (primary adapters)
    ↓
application (use cases)
    ↓
domain (entities + ports)
    ↑
infrastructure (secondary adapters)
```

Infrastructure implements the ports defined in domain — so the arrow points inward even though it sits on the outside.

---

## Layers

### Domain
The heart of the application. No framework imports. No database imports. Pure TypeScript.

- **Entity** — models a real-world thing, owns and enforces business rules
- **Repository Interface (Port)** — defines the *shape* of persistence without caring how it's implemented

```ts
// modules/invitation/domain.ts
export class Invitation {
  markOpened() {
    if (this.openedAt) return
    this.openedAt = new Date()
  }
  markBounced() {
    this.bouncedAt = new Date()
  }
}

// RsvpResponse and CheckIn are standalone entities — not methods on Invitation.
// Their existence IS the state. Creating them IS the transition.
export class RsvpResponse {
  constructor(
    public invitationId: string,
    public accepted: boolean,
    public noteToOrganizer?: string,
    public noteToHost?: string,
    public dietaryNeedSnapshot?: string,
  ) {}
}

export class CheckIn {
  public checkedInAt: Date
  constructor(public invitationId: string, public actorId: string) {
    this.checkedInAt = new Date()
  }
}

export interface InvitationRepository {
  save(invitation: Invitation): Promise<void>
  findByToken(token: string, tenantId: string): Promise<Invitation | null>
}
```

### Application
Orchestrates domain objects to fulfill one user goal. No business rules live here — it delegates to entities. No framework or database knowledge.

- **Use Case** — fetches via repo, calls entity methods, saves result

```ts
// modules/invitation/application.ts
export class CheckInGuestUseCase {
  constructor(
    private permissions: PermissionManager,
    private invitations: InvitationRepository,
    private checkIns: CheckInRepository,
  ) {}

  async execute(actorId: string, token: string, tenantId: string) {
    const invitation = await this.invitations.findByToken(token, tenantId)
    if (!invitation) throw new InvalidTokenError()

    await this.permissions.assertEvent(actorId, invitation.eventId, EventPermission.CHECK_IN)

    if (await this.checkIns.existsForInvitation(invitation.id)) {
      throw new AlreadyCheckedInError()
    }

    const checkIn = new CheckIn(invitation.id, actorId)
    await this.checkIns.save(checkIn)
    return checkIn
  }
}
```

### Infrastructure (Secondary Adapters)
Implements the ports defined in domain. These are the **exit points** — your app drives them.

- Prisma repositories (database)
- Email adapter
- QR code generator
- Permission manager

```ts
// modules/invitation/infra.ts
export class PrismaInvitationRepository implements InvitationRepository {
  async save(invitation: Invitation) { ... }
  async findByToken(token: string, tenantId: string) { ... }
}
```

### Permission Manager
A **domain service** responsible for all authorization checks. Its interface lives in `shared/ports/` (used by all modules); its Prisma implementation lives in `infrastructure/`. Every use case that touches a protected resource calls it first — before any business logic runs.

```ts
// shared/ports/PermissionManager.ts
export interface PermissionManager {
  assertEvent(actorId: string, eventId: string, permission: number): Promise<void>
  assertTenant(actorId: string, tenantId: string, permission: number): Promise<void>
}
```

`assert*` throws `ForbiddenError` on failure — use cases do not branch on a boolean, they proceed or are interrupted.

Permission constants live in `shared/permissions.ts` and are imported wherever needed:

```ts
export const EventPermission = {
  VIEW: 1, MANAGE_GUESTS: 2, MANAGE_INVITES: 4,
  CHECK_IN: 8, EXPORT: 16, MANAGE_EVENT: 32,
} as const

export const TenantPermission = {
  VIEW_EVENTS: 1, CREATE_EVENT: 2, ARCHIVE_EVENT: 4,
  MANAGE_MEMBERS: 8, MANAGE_SETTINGS: 16,
} as const
```

See the [Permission Model](#permission-model) section for full details.

### Container (Composition Root)
Not a layer — just the wiring. Instantiates infrastructure adapters and injects them into use cases. Runs once at startup. This is the only place that knows what implements what.

```ts
// modules/invitation/container.ts
export class InvitationContainer {
  private repo = new PrismaInvitationRepository()
  private checkIns = new PrismaCheckInRepository()
  private permissions = new PrismaPermissionManager()
  checkInGuest = new CheckInGuestUseCase(this.permissions, this.repo, this.checkIns)
}
```

### HTTP (Primary Adapter)
The **entry point** — it drives your app. Knows about Fastify, request/response, and validation schemas. Does not know about Prisma, infrastructure, or business rules.

The handler's only auth responsibility is verifying the session token and extracting `userId`. Authorization — "can this user do this?" — is the permission manager's job, not the handler's.

```ts
// modules/invitation/http.ts
app.post('/invitations/check-in', async (request, reply) => {
  const checkIn = await container.checkInGuest.execute(request.userId, ...)
  return reply.send(checkIn)
})
```

---

## Request Flow

```
HTTP Request
  → Route Handler         validates input, extracts fields, provides userId from session
  → Use Case              assertEvent/assertTenant via PermissionManager, then orchestrates
    → Entity              enforces business rules
    → Repository Port     persistence interface
      → Prisma Adapter    fulfills the port (invisible above this)
```

---

## Domain Boundaries

### Core Domains

| Domain | Role | Notes |
|---|---|---|
| Tenant | Supporting | Foundational container for everything else |
| Event | Central container | Everything else lives inside an event |
| Venue | Supporting | Reusable locations, scoped to tenant |
| Guest | Core | Pure identity — the person, tenant-scoped |
| EventGuest | Supporting | Roster entry — links a Guest to an Event |
| Invitation | Load-bearing connector | Owns delivery token and delivery state |
| RsvpResponse | Supporting | Guest's response to an invitation |
| CheckIn | Supporting | Attendance record on event day |
| InvitationEvent | Audit log | Append-only delivery event log (SENT, OPENED, BOUNCED) |
| Notification | Supporting infrastructure | No domain rules — purely a delivery mechanism |

### Ownership Rules

| Concept | Owner | Notes |
|---|---|---|
| Tenant identity | Tenant | Source of truth for org and membership |
| Tenant-level permissions | TenantMembership | Bitwise permissions + isOwner flag |
| Event-level permissions | EventMembership | Bitwise permissions, explicit for everyone including OWNER |
| Event lifecycle | Event | Owns status transitions |
| Guest identity | Guest | Name, email, phone, dietary need |
| Event roster entry | EventGuest | Must exist before an Invitation can be created |
| Delivery token | Invitation | QR code and magic link derive from this |
| Delivery state | Invitation | sentAt, openedAt, bouncedAt |
| RSVP response | RsvpResponse | accepted, noteToOrganizer, noteToHost, dietary snapshot |
| Check-in record | CheckIn | actorId, checkedInAt |
| Delivery audit log | InvitationEvent | Append-only; answers "was this sent?" |
| Plus-one relationship | PlusOne | Separate model — Guest stays pure |
| Dietary need | DietaryNeed | Separate model linked to Guest |

### The Invitation is the Load-Bearing Connector

`Invitation` binds a Guest to an Event and owns the delivery mechanism. Its state is fully derived from related records — there is no status field:

- **Delivery state** — derived from `sentAt`, `openedAt`, `bouncedAt` timestamps on Invitation
- **RSVP state** — derived from the presence of `RsvpResponse` (and its `accepted` field)
- **Check-in state** — derived from the presence of a `CheckIn` record

**Guest owns the person's identity. EventGuest owns the roster entry. Invitation owns the delivery token. RsvpResponse and CheckIn own the guest's actions.**

The lifecycle flows in one direction: Guest → EventGuest → Invitation → RsvpResponse / CheckIn. A guest can exist on a roster without an invitation (added but not yet invited). An invitation cannot exist without a prior EventGuest record — enforced at the schema level via FK.

### Boundary Rules

- **Invitation never reaches into Guest internals.** It holds a `guestId` reference. If guest data is needed alongside invitation data, the use case fetches both and composes them. Entities do not know about each other.
- **Notification has no domain layer.** It is purely infrastructure. No `Notification` entity. No business rules. Use cases call an `EmailPort` interface; the implementation lives in `infrastructure/email.ts`. There is no `notification` module.
- **Domain entities never import from application, infrastructure, or HTTP.** A Prisma import in a domain file is a boundary violation.
- **`tenantId` never appears in domain entity logic.** It is a repository concern. If a domain method receives `tenantId` as a parameter for filtering, the scoping is in the wrong place.
- **The OWNER of a tenant cannot be deleted or demoted by anyone, including themselves.** Enforced in the permission manager.

---

## Permission Model

Authorization operates on two planes. Both use bitwise `permissions Int` fields. No role enum anywhere.

### Tenant Plane

Governed by `TenantMembership.permissions` + `isOwner Boolean`:

| Bit | Constant | Meaning |
|---|---|---|
| 1 | TENANT_VIEW_EVENTS | See event list |
| 2 | TENANT_CREATE_EVENT | Create new events |
| 4 | TENANT_ARCHIVE_EVENT | Archive/restore events |
| 8 | TENANT_MANAGE_MEMBERS | Invite, remove, update member permissions |
| 16 | TENANT_MANAGE_SETTINGS | Edit tenant name and settings |

`isOwner` is a separate boolean — not a bit. Exactly one per tenant, enforced by the permission manager.

Preset compositions (app code only): STAFF=1, ORGANIZER=3, ADMIN=15, OWNER=31+isOwner=true

### Event Plane

Governed by `EventMembership.permissions`:

| Bit | Constant | Meaning |
|---|---|---|
| 1 | VIEW | See event and guest list |
| 2 | MANAGE_GUESTS | Add, edit, remove guests |
| 4 | MANAGE_INVITES | Send, resend, revoke invitations |
| 8 | CHECK_IN | Scan QR, mark attendance |
| 16 | EXPORT | Export guest list and reports |
| 32 | MANAGE_EVENT | Edit event details |

Preset compositions (app code only): VIEWER=1, STAFF=9, ORGANIZER=7, MANAGER=31, FULL=63

### Rules

- **Explicit for everyone** — no implicit bypass, even OWNER/ADMIN must have an EventMembership record
- **On event creation** — permission manager auto-issues FULL (63) EventMembership to the creator
- **Cascading revoke** — archiving a User or removing a TenantMembership must cascade-revoke their EventMemberships (permission manager responsibility)
- **Never check membership records directly in use cases** — always go through the permission manager interface

---

## State Machines

### Invitation State (Derived)

Invitation has no status field. State is fully derived from related records:

| Derived State | Condition |
|---|---|
| Pending | No sentAt |
| Sent | sentAt set, no openedAt, no bouncedAt |
| Opened | openedAt set |
| Bounced | bouncedAt set |
| RSVP'd — accepted | RsvpResponse exists, accepted = true |
| RSVP'd — declined | RsvpResponse exists, accepted = false |
| Checked in | CheckIn record exists (terminal — cannot be deleted) |

- **openedAt** is set by the system when the guest first hits the magic link URL (`/i/{token}`). It is server-side, triggered by the HTTP request — not an email provider event. It is passive link-click tracking, not a guest form submission.
- **bouncedAt** is set by the email provider webhook (ops/infrastructure). Not set by app code.
- A guest can change their RSVP — the existing `RsvpResponse` record is deleted and a new one inserted. There is no update-in-place and no history kept (v2 concern).
- Check-in is terminal — `CheckIn` records are never deleted.

### Event Status

```
PLANNING → ONGOING → COMPLETED (terminal)
         ↘          ↗
          CANCELLED
              ↕
        (reversible to PLANNING)
```

- `COMPLETED` is the only terminal state
- `CANCELLED` can be reinstated back to `PLANNING`

---

## Token Strategy

Every `Invitation` has a single opaque token. Both the QR code and the magic link are representations of the same URL:

```
https://eventease.com/i/{token}
```

- The token is cryptographically random and fully opaque — no sequential or guessable component is ever exposed in the URL
- On reissue (revoke and regenerate), a new opaque token is generated and overwrites the old one. The old QR code and magic link immediately become invalid.
- `tokenVersion` is stored internally on `Invitation` for audit purposes only — it is never exposed in the URL or the QR code
- Token validity is also time-scoped: tokens are disabled once `event.startDate` has passed

---

## RSVP Data

Captured in a separate `RsvpResponse` record when a guest responds:

| Field | Notes |
|---|---|
| `accepted` | Boolean — true for accepted, false for declined |
| `respondedAt` | Timestamp of response (auto-set on creation) |
| `noteToOrganizer` | Optional private note for planners and staff |
| `noteToHost` | Optional guest message for the host |
| `dietaryNeedSnapshot` | Snapshot at RSVP time — what the organizer acts on |

`Guest.dietaryNeed` is the source of truth and pre-populates the RSVP form. When a guest confirms or edits at RSVP time, both `RsvpResponse.dietaryNeedSnapshot` (snapshot) and `Guest.dietaryNeed` (source of truth) are updated. The snapshot ensures catering orders remain stable even if the guest later changes their profile.

A guest can change their mind — the existing `RsvpResponse` is deleted and a new one is inserted. There is no update-in-place, which is why `RsvpResponse` has no `updatedAt`. History of RSVP changes is a v2 concern.

---

## Guest Model Purity

`Guest` is a pure identity model, scoped to a tenant — not to an event. A guest is a person in the tenant's address book and can be invited to multiple events via separate `Invitation` records.

```
name          required
email         optional (required only if sending invitation digitally)
phone         optional (future SMS delivery)
```

Related concerns live in separate models:
- `DietaryNeed` — linked to Guest
- `PlusOne` — captures the relationship between a primary guest and their plus-one
- `Invitation` — owns the delivery token and links Guest to a specific Event
- `RsvpResponse` — owns the guest's response (via Invitation)
- `CheckIn` — owns the attendance record (via Invitation)

**No hard uniqueness constraint on Guest at the DB level.** Email is optional (walk-ins, plus-ones, elderly relatives). The use case layer warns on likely duplicates when email is provided but does not hard-block creation.

---

## Plus-One Model

Every attendee — including plus-ones — has their own named `Guest` record and their own `Invitation` with a unique QR code. No anonymous plus-ones.

- **Directly invited** — has an `Invitation`, no `PlusOne` record
- **Plus-one** — has an `Invitation` + a `PlusOne` record pointing to their host guest

`Guest` is unaware of the plus-one concept. `PlusOne` owns the relationship.

---

## Check-in Rules

- Every person at the venue must have a named `Invitation` and a valid QR code
- Walk-ins are not supported in v1 — staff cannot create guests on the spot
- Plus-ones are declared at RSVP time with names captured then
- Duplicate QR scan (same token scanned twice) returns a warning "Already checked in" but still shows guest details — staff makes the judgment call
- Offline check-in is a v2 concern

---

## Folder Structure

EventEase uses a **Modular Monolith** structure. Each domain concept is a self-contained module with its own internal layers, unified by an `index.ts` as its public interface. The architecture is still fully hexagonal — the dependency rule holds inside every module.

```
src/
├── modules/
│   ├── tenant/
│   │   ├── domain.ts         — Entity + Repository interface (port)
│   │   ├── application.ts    — Use cases
│   │   ├── infra.ts          — Prisma repository (secondary adapter)
│   │   ├── http.ts           — Routes + Fastify schemas (primary adapter)
│   │   ├── container.ts      — Wiring for this module
│   │   └── index.ts          — Public interface (the only thing other modules import)
│   │
│   ├── event/
│   │   ├── domain.ts
│   │   ├── application.ts
│   │   ├── infra.ts
│   │   ├── http.ts
│   │   ├── container.ts
│   │   └── index.ts
│   │
│   ├── venue/
│   │   ├── domain.ts
│   │   ├── application.ts
│   │   ├── infra.ts
│   │   ├── http.ts
│   │   ├── container.ts
│   │   └── index.ts
│   │
│   ├── guest/
│   │   ├── domain.ts
│   │   ├── application.ts
│   │   ├── infra.ts
│   │   ├── http.ts
│   │   ├── container.ts
│   │   └── index.ts
│   │
│   ├── invitation/
│   │   ├── domain.ts
│   │   ├── application.ts
│   │   ├── infra.ts
│   │   ├── http.ts
│   │   ├── container.ts
│   │   └── index.ts
│   │
│   └── auth/
│       ├── application.ts    — RequestMagicLink, VerifyMagicLink use cases
│       ├── infra.ts          — MagicLinkToken Prisma queries (no entity — no business rules)
│       ├── http.ts
│       ├── container.ts
│       └── index.ts
│
├── shared/                   — Cross-cutting types and ports (no business logic)
│   ├── permissions.ts        — EventPermission + TenantPermission constants
│   └── ports/
│       └── PermissionManager.ts  — Interface used by all use cases
│
├── infrastructure/           — Shared infrastructure (not module-specific)
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── client.ts
│   ├── email.ts              — EmailPort implementation
│   ├── qr.ts                 — QR code generator
│   └── permissions.ts        — PrismaPermissionManager implementation
│
├── plugins/                  — Fastify plugins (auth, multitenancy)
│   ├── auth.ts               — Session token verification, injects userId into request
│   └── multitenancy.ts       — Injects tenantId into request context
│
├── app.ts                    — Registers all module routes + plugins
└── server.ts                 — Boots the server
```

> **Note:** This structure reflects the initial known features. It is an example, not a prescription.

### The index.ts Contract

`index.ts` is the **only** public surface of a module. Other modules never reach into a module's internals.

```ts
// modules/guest/index.ts — public contract
export { GuestContainer } from './container'
export type { AddGuestInput } from './application'
export type { Guest } from './domain'  // only if other modules genuinely need the type
```

```ts
// modules/invitation/application.ts — consuming guest from another module
import type { Guest } from '../guest'        // ✅ through the public interface
import type { Guest } from '../guest/domain' // ❌ reaching into internals — violation
```

The mental model is an npm package — you interact only with what's exported from the package root, never with internal files.

### index.ts as a Health Metric

The size of `index.ts` signals the health of the module boundary:

- **Small `index.ts`** — module is well-encapsulated, boundaries are right
- **Growing `index.ts`** — the module may be too large, or boundaries are leaking
- **Many shared types** — domain concepts may need rethinking

A deep module has a narrow interface. `index.ts` makes the interface width visible.

---

## Rules for Adding Files or Folders

Before adding a new file or folder, justify it against one of these:

| Trigger | Action |
|---|---|
| New domain concept (e.g. `Venue`) | Add a new module folder under `modules/` with its own `domain.ts`, `application.ts`, `infra.ts`, `http.ts`, `container.ts`, `index.ts` |
| New infrastructure provider (e.g. swap email) | Replace the file in `infrastructure/` — nothing else changes |
| New entry point (e.g. CLI) | Add `cli.ts` inside the relevant module alongside `http.ts` — same use cases, new primary adapter |
| A module file exceeds ~200 lines and has distinct concerns | Split into subfiles within the same module folder (e.g. `application.create.ts`, `application.checkin.ts`) |
| A concept is needed by 3+ modules | Consider promoting it to `shared/` — but only types and ports, never logic |
| A concept spans multiple modules | Consider a domain service inside the owning module — do not leak logic into application layer |

**Do not create a new module for something that isn't a genuine domain concept.** Start flat within a module, split when it hurts.

---

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Architecture | Hexagonal Modular Monolith | Domain isolation, swappable infrastructure, feature co-location |
| Multi-tenancy | Shared DB, `tenantId` on all entities | Simple, Prisma-friendly, cost-effective |
| Tenant scoping | Repository layer only | Domain never thinks about tenants |
| Auth (users) | Magic link → session token | No passwords stored; consistent with guest auth mental model |
| Auth (guests) | QR code / magic link | Guests should not need accounts |
| ORM | Prisma | Seamless SQLite → Postgres swap |
| HTTP framework | Fastify | Lightweight, plugin system, schema validation |
| DI | Manual (no library) | Simple, explicit, sufficient for current scale |
| Frontend | SPA (React or Vue) | App-like UX, API is already agent-ready |
| Folder structure | Modular Monolith (`modules/` per domain concept) | Feature co-location, explicit public interfaces via `index.ts`, clear path to extraction |
| Permission model | Bitwise permissions on both planes | Flexible without schema changes; presets in app code only |
| Explicit event access | Everyone needs EventMembership, no implicit bypass | Accidental bypass (even for admins) is a correctness risk; explicit is auditable |
| Invitation state | Derived from related records, no status field | Single source of truth; a status field was a denormalization of what the records already said |
| Roster vs delivery separation | EventGuest + Invitation | Organizer can build a guest list before deciding to send invitations; conflating the two forces an invitation record just to add someone to a list |
| RSVP data | Separate RsvpResponse model | Keeps Invitation as a pure connector; RSVP is its own lifecycle |
| Check-in | Separate CheckIn model | Explicit record with actor; presence = checked in, terminal |
| Guest uniqueness | No hard DB constraint | Email is optional; soft duplicate warning in use case |
| Plus-one | Separate PlusOne model | Guest stays pure; every attendee has their own Invitation |
| Dietary need | Separate DietaryNeed model | Guest stays pure; snapshot stored on RsvpResponse at RSVP time |

---

## Glossary

| Term | Definition |
|---|---|
| Entity | A class that models a real-world thing and owns its business rules |
| Repository Interface | A TypeScript interface (port) defining the shape of persistence |
| Use Case | An application-layer class that orchestrates one user goal |
| Port | An interface in domain that infrastructure must implement |
| Primary Adapter | Entry point that drives the app (HTTP, CLI) |
| Secondary Adapter | Exit point that the app drives (Prisma, email, QR) |
| Composition Root | The single place where all dependencies are wired together |
| Modular Monolith | An architecture where each domain concept is a self-contained module with its own internal layers, deployed as a single unit but with explicit module boundaries enforced via `index.ts` |
| Load-bearing Connector | An entity so central that most other domain concepts depend on it — in EventEase, this is Invitation |
| Permission Manager | A domain service that handles all authorization checks across both tenant and event planes |
| Bitwise Permission | An integer field where each bit represents a capability. Combined with bitwise OR to compose access presets. |
| Snapshot Field | A field that captures the value of another model's data at a point in time, so that future changes to the source do not affect historical records |
