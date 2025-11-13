# Calendar Feature Architecture

## Purpose
- Deliver first-class calendar functionality in Thunderbird for Android that aligns with Thunderbird for Desktop (Lightning).
- Keep the feature modular so it can evolve independently and be distributed behind feature flags during early rollout.
- Reuse existing core infrastructure for accounts, sync scheduling, storage, telemetry, and notification surfaces.

## Scope & Parity Goals
- **MVP**: Read-only CalDAV + ICS subscription, agenda and list views, invitation preview from email, account capability detection, basic reminders via Android notifications.
- **Parity Phase**: Writable CalDAV (create/edit/delete events), full ICS import/export, recurring events and exceptions, RSVP workflow from email invitations, shared calendars, offline cache, task list (if prioritized).
- **Future Enhancements**: Matrix of multi-account support, search integration, widgets, cross-device sync settings, synchronization conflict resolution, Thunderbird Account Services integration.

## Module Layout
```
feature:calendar
├── feature:calendar:api           // Public contracts for calendar feature entry points
├── feature:calendar:impl          // UI Compose screens, navigation destinations, DI wiring
├── feature:calendar:event         // Event domain logic
│   ├── feature:calendar:event:api // Event repository + use-cases
│   └── feature:calendar:event:impl// Event interactors and default implementations
└── feature:calendar:sync          // Sync engines and schedulers
    ├── feature:calendar:sync:api  // Sync contract abstractions
    └── feature:calendar:sync:impl // CalDAV, ICS sync workers, WorkManager jobs
```

Supporting modules anticipated during implementation:
- `core:calendar`: Domain models (events, attendees, reminders), shared formatting utilities, calendar-specific preferences, database schema definitions.
- `backend:caldav`: Protocol layer for CalDAV, request/response mappers, DAV authentication integration.
- `backend:ics`: Parser/serializer for iCalendar content, invitation handling helpers.
- `feature:notification:calendar`: Calendar-specific notification builders (optionally in `feature:notification` as sub-module).

## Dependencies & Integration
- **Accounts**: Extend `core:account` capability descriptors so calendars advertise `supportsCalendar`. Provide CalDAV discovery during account setup if server exposes calendar-home-set.
- **Mail**: Hook into MIME processing to surface `text/calendar` parts, route invites to calendar feature intents, and write RSVP responses via existing SMTP stack.
- **Navigation**: Add calendar entry to side-rail and/or bottom navigation through `feature:navigation` APIs.
- **Settings**: Introduce calendar settings screens (sync interval, default reminder) using `feature:settings` extension points.
- **Telemetry**: Add Glean events for sync latency, events created, RSVP responses.

## Data Model & Storage
- Persist events locally using Room (preferred) or existing SQLDelight stack; consider `core:common` abstractions for multi-module DB access.
- Recommended entities: `CalendarAccount`, `Calendar`, `CalendarEvent`, `EventAttendee`, `Reminder`, `SyncState`.
- Provide mappers from CalDAV/ICS payloads to domain models; reuse in unit tests with fixture builders.

## Sync & Protocol Strategy
- **CalDAV**: Start with read-only sync; leverage `OkHttp` + WebDav libraries or implement lightweight PROPFIND/REPORT support. Align scheduler with mail sync (Account sync preferences, WorkManager unique jobs per account/calendar).
- **ICS Subscription**: Support readonly ICS URLs stored per account; schedule fetch and merge operations.
- **Conflict Handling**: For parity phase, adopt last-write-wins with server precedence; surface conflicts via notification or snackbar.
- **Background Execution**: Reuse existing sync infrastructure (`core:sync` patterns) for power-saving compliance and doze handling.

## UI & UX
- Compose screens hosted in `feature:calendar:impl` with previews in `app-ui-catalog`.
- Core surfaces: Agenda (list), Day, 3-Day/Week, Month grid, Event detail.
- Event editor: Step-by-step form with date/time pickers, attendee management, reminders, notes.
- Deep links from mail invites open event preview; allow RSVP actions inline.
- Accessibility: Support TalkBack announcements, high-contrast themes, large text.

## Notifications & Reminders
- Schedule local notifications using `AlarmManager` or `WorkManager` with exact alarms opt-in.
- Provide snooze/dismiss actions, quick RSVP shortcuts for invites.
- Integrate with Android notification channels (`calendar_events`), respect DND and system settings.

## Settings & Feature Flags
- Feature flag gate via `featureflag` infrastructure; allow staged rollout per channel (beta, nightly).
- Settings additions: Default calendar per account, sync interval, default reminder lead time, working hours.
- Provide migration toggles to import Thunderbird desktop settings once remote storage is available.

## Testing Strategy
- Unit tests: Event recurrence expansion, CalDAV parser, ICS serializer, RSVP workflow.
- Integration tests: Sync pipeline using `backend/testing` infrastructure with mock DAV server.
- UI tests: Compose UI automations in `ui-flows/calendar` for agenda navigation, event creation.
- Contract tests: Ensure mail invites open calendar UI using `feature:mail` instrumentation hooks.

## Rollout Milestones
1. Architecture scaffolding: Create modules, register Gradle entries, add empty feature flag, baseline UI placeholder.
2. Read-only agenda: CalDAV subscription + agenda view + invite preview (beta behind flag).
3. Event editing: Writable CalDAV, reminder notifications, ICS import/export.
4. Parity completion: Recurrence, shared calendars, cross-device settings, polished navigation integration.
5. Enhancements: Widgets, search integration, offline conflict resolution.

## Open Questions
- Preferred CalDAV library vs. custom implementation (evaluate licencing compatibility with MPL 2.0).
- Storage layer choice (Room vs. SQLDelight) aligned with project direction.
- Handling of Thunderbird Account Services calendar provisioning (once backend APIs exist).
- Cross-platform telemetry alignment with desktop for combined reporting.

## Next Actions
- Draft detailed technical specification per milestone and seek review.
- Prototype CalDAV client against common servers (Google, Nextcloud, Fastmail) to uncover interoperability issues.
- Determine UX design requirements in collaboration with design team; produce Figma references.
- Align with release management for feature-flag rollout strategy.
