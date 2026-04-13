# EventEase — Product

## What is EventEase?

EventEase is a multi-tenant event management platform designed to streamline the full lifecycle of any event — from planning and guest management, through invitations and RSVPs, to event day check-in and post-event follow-up.

It is built for event organizers who are tired of juggling multiple tools and spreadsheets to manage a single event.

---

## The Problem

Event management today is fragmented and stressful. Organizers typically use one tool to track guests, another to send invitations, a manual process to handle RSVPs, and something ad hoc on event day to manage check-ins. Nothing talks to each other. Data gets out of sync. Things fall through the cracks.

The result: organizers spend more time managing tools than managing their event.

EventEase solves this by being the single system of record for the entire event lifecycle — from the first guest added to the last thank-you sent.

---

## Who Is It For?

### Primary User — The Organizer
The person responsible for planning and executing the event. They create the event, manage the guest list, send invitations, track RSVPs, and monitor check-ins on event day. In a corporate context this might be an events team. In a personal context, a wedding coordinator or the couple themselves.

### Secondary User — The Event Host
The person the event is for — the bride, groom, birthday celebrant, or executive. They may have visibility into the event without being the one managing the operational details.

### Supporting User — Staff / Ushers
On event day, staff use a tablet-friendly check-in dashboard to scan QR codes and verify guests at the door. They have no access to planning or guest management — only check-in.

### End Recipient — The Guest
Guests never log in. They receive an invitation via email or link, access their personal invitation page via a magic link or QR code, submit their RSVP, and on event day use their QR to check in. The same link becomes a post-event memory page after the event concludes.

---

## Core Value Proposition

One product. The entire event lifecycle. Intuitive enough that the stress of running an event comes from the event itself — not the software.

---

## Feature Phases

### v1 — Core (Current Focus)

The goal of v1 is to support a real wedding end-to-end.

- Tenant (organization) signup and onboarding
- Bitwise permission model for tenant and event access, with explicit per-event assignment
- Magic link authentication — no passwords
- Create and manage events with venue support
- Guest list management (manual entry)
- Dietary need capture per guest
- Plus-one support — each plus-one is a named guest with their own invitation
- Invitation creation and delivery via email / magic link
- QR code generation per guest (every attendee has their own QR)
- RSVP flow — guests confirm or decline via their magic link
- RSVP change — guests can update their response
- Event day QR check-in — staff scans, system validates, guest is marked as arrived
- Bounce tracking — organizers can see which invitations failed delivery
- Basic RSVP and attendance tracking for organizers

**Non-goals for v1:**
- Offline check-in
- Walk-in guest creation at the door
- AI-assisted seating
- Sub-event tracking (e.g. rehearsal dinner, bachelor party)
- CSV guest import (deferred due to hidden complexity)
- Complex task management for organizers
- Social auth (deferred to v2)

### v2 — Intelligence

- Social auth (OAuth providers)
- AI-assisted seating arrangement
- Real-time event day dashboard (arrivals, no-shows, walk-ins)
- Seating map view for ushers
- Sub-event attendance tracking
- Offline check-in tolerance
- SMS delivery

### v3 — Post-Event Automation

- Auto-generated attendance report
- AI-written personalized thank you messages
- Feedback survey auto-sent post-event
- AI event summary for organizer
- Digital guest book / memory export

### v4 — SaaS

- Public signup
- Billing / subscription tiers (per tenant)
- White-label options
- Multi-event analytics

---

## Key Domain Concepts

### Tenant
An organization using EventEase. Multi-tenancy is a first-class concern from day one. All data is scoped to a tenant.

### Event
The central container. Everything — guests, invitations, venues — lives inside an event. Events have a lifecycle: `PLANNING → ONGOING → COMPLETED`. A cancelled event can be reinstated to `PLANNING`. `COMPLETED` is the only terminal state.

### Venue
A physical location, scoped to a tenant and reusable across events. Stores name, address, and coordinates for maps. Selected from a dropdown when creating or editing an event.

### Guest
A person in the tenant's address book. Pure identity: name, email (optional), phone (optional). Dietary needs are captured separately. A guest is tenant-scoped — not event-scoped — and can be invited to multiple events via separate Invitation records.

### EventGuest
The roster entry that places a guest on an event's list. Must exist before an invitation can be created. Separates "on the list" from "has been invited" — an organizer can build the full guest list before deciding who to invite and when.

### Invitation
The delivery record for a roster entry. Requires an EventGuest to exist first. Owns the delivery token (QR code and magic link). Delivery state is derived from timestamps (`sentAt`, `openedAt`, `bouncedAt`). RSVP and check-in state are derived from the presence of related records.

### RsvpResponse
Created when a guest responds to their invitation. Records whether they accepted or declined, optional `noteToOrganizer` and `noteToHost` fields, and a dietary need snapshot. A guest can update their response — the record is replaced, not appended.

### CheckIn
Created when a guest is checked in on event day. Records who performed the check-in and when. Terminal — once created, it cannot be deleted. Presence of this record means the guest has arrived.

### InvitationEvent
An append-only delivery audit log. Records every delivery action against an invitation: SENT, OPENED, BOUNCED. Never updated, never deleted. Answers "was this sent?" and "did it bounce?" without conflicting with the derived state model.

### Plus-One
A named guest added by a primary guest at RSVP time. Plus-ones get their own guest record and their own invitation. No anonymous plus-ones — everyone at the door has a name in the system.

### QR Code / Magic Link
Both point to the same URL: `https://eventease.com/i/{token}`. The token is cryptographically opaque. On revoke-and-reissue, a new token is generated and the old one immediately becomes invalid. Tokens expire when the event's start time has passed.

---

## What Must Never Go Wrong

- An invitation token must never be guessable or enumerable
- A checked-in guest cannot be un-checked-in
- A guest from one tenant must never be visible to another tenant
- The OWNER of a tenant cannot be deleted or demoted
- A user must never gain event access without an explicit EventMembership record — no implicit bypass, even for admins

---

## Success Definition

For the creator: EventEase is used successfully for a real wedding, demonstrates real-world architectural skill in portfolio and job interviews, and serves as a living reference for hexagonal architecture, domain modeling, and production system thinking.

For future users: An organizer runs their entire event — from first guest added to last check-in scanned — without leaving EventEase.
