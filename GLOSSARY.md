# DeviceDesk

DeviceDesk is the team's tool for booking desks and meeting rooms in its office. This repo also holds the agent-run delivery process that builds it, so it has two vocabularies: Booking (the product) and Delivery (how the product gets built and shipped).

## Booking

### Places

**Office**:
The team's premises shown on the Floor Plan, made up of Areas and Meeting Rooms.
_Avoid_: building, floor, site

**Area**:
A shared space in the Office that holds Desks. An Area itself is never booked.
_Avoid_: room, shared room, zone

**Desk**:
A single workplace in an Area, booked for a whole Work Day.
_Avoid_: seat, station, workstation

**Meeting Room**:
A room that is booked in Slots.
_Avoid_: room (on its own), conference room, boardroom

**Space**:
A Desk or a Meeting Room: anything that can be booked or blocked.
_Avoid_: resource, asset, bookable

**Capacity**:
How many people a Meeting Room seats. It's shown to people but not enforced.

**Smoke-test Desk**:
A hidden Desk used only by release checks. Members never see it.
_Avoid_: test desk, canary desk

**Layout**:
The data that describes the Office: its Areas and Spaces, and where each one appears on the Floor Plan.
_Avoid_: map config, seating chart

**Floor Plan**:
The sketch of one Area, or of the Meeting Rooms, for one date, showing each Space's State.
_Avoid_: map (on its own), seat map

**Office Overview**:
A small picture of the whole Office, used to choose which Floor Plan to show.
_Avoid_: minimap, index

**State**:
How a Space looks on the Floor Plan for one date: free, mine, taken, partly booked (Meeting Rooms only) or blocked.

### Time

**Work Day**:
A date from Sunday to Thursday, in Asia/Dhaka.
_Avoid_: business day, weekday

**Slot**:
A 30-minute interval between 09:00 and 18:00 Asia/Dhaka. It's the unit of a Room Booking.
_Avoid_: period, block (Block means something else)

**Booking Window**:
The dates that can be booked: from today in Asia/Dhaka to 14 calendar days ahead, inclusive.
_Avoid_: horizon, lead time

### Bookings

**Booking**:
A Member's hold on a Space: either a Desk Booking or a Room Booking.
_Avoid_: reservation, appointment

**Desk Booking**:
A Booking of one Desk for one Work Day.

**Room Booking**:
A Booking of one Meeting Room for one or more consecutive Slots on one Work Day, at most two hours in total.
_Avoid_: meeting (the Booking is not the meeting)

**Owner**:
The Member a Booking belongs to.
_Avoid_: booker, organizer

**Cancellation**:
Ending a Booking before it ends. The cancelled Booking stays in history and records whether its Owner or an Admin cancelled it.
_Avoid_: deletion, release

**Block**:
An Admin's declaration that a Space, or the whole Office, can't be booked for a range of dates. A Block cancels any Bookings it overlaps.
_Avoid_: closure, hold, maintenance

### People

**Member**:
A person from the company's Entra tenant who has signed in, identified by their Entra object ID. Members book Spaces and cancel their own Bookings.
_Avoid_: user, employee, account

**Admin**:
A Member whose email is on the admin list. Admins can cancel any Booking and create Blocks. They can't book for someone else.
_Avoid_: manager, superuser

**Smoke Principal**:
A non-human identity, used only by the release check. It can book and cancel the Smoke-test Desk and nothing else.
_Avoid_: test user, service account, bot

## Delivery

**Ticket**:
A GitHub issue describing one piece of work small enough to implement and verify in one attempt, listing the Tickets that block it.
_Avoid_: task, story, card

**Proposal**:
A suggested change filed by an agent (a Persona, the product owner, triage or the reproduction agent). It becomes a Ticket only after a human approves it.
_Avoid_: suggestion, feature request

**Attempt**:
One fresh agent session that tries to get a Ticket merged. A Ticket gets at most three.
_Avoid_: run, try

**Evidence**:
What `devicedesk-ctl` produces to show a change works: screenshots, a state diff and, where relevant, a trace. Every pull request has to carry it.
_Avoid_: proof, artifacts

**Feature Map**:
The verification skill's per-feature reference: user paths, CLI commands, selectors and gotchas for each feature.
_Avoid_: feature docs, test plan

**Release Gate**:
The check that runs after every deploy and decides whether the new version stays or is rolled back.
_Avoid_: smoke test (it's only one step of the gate), health check

**Baseline**:
The metrics recorded from the last version that passed the Release Gate. The next version is compared against it.

**Correction**:
A fix a human had to make to the agents' output, logged with the layer the fix went into.
_Avoid_: intervention, override

**Persona**:
A simulated stakeholder agent that walks through the deployed app and files Proposals backed by Evidence.
_Avoid_: tester, user agent

**Phase Transition**:
A point where the driver pauses until a human chooses which Proposals become Tickets.
_Avoid_: milestone, checkpoint
