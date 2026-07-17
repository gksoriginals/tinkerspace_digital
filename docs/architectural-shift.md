# Architectural Shift for TinkerSpace Digital

Date: July 17, 2026

## Why this document exists

`tinkerspace_digital` works today as a TinkerSpace-specific kiosk display. It is good at rendering one concrete installation, but it is not yet shaped like a reusable display system that another organization could adopt with configuration alone.

This document captures the architectural shift needed for this project to evolve from:

- a static frontend wired to TinkerHub private APIs

into:

- a modular display platform that still supports the current TinkerSpace deployment

This is not a greenfield product plan. It is a proposed shift in how this existing project should be structured over time.

## Current state

Today the project is primarily:

- a Netlify-hosted static React app
- a read-only display shell for two upstream data sources
- directly coupled to TinkerSpace branding, location, display rules, and data semantics

The current frontend fetches:

- people / check-in data from `REACT_APP_API_BASE_URL`
- calendar display data from `REACT_APP_SPACECALENDAR_API`
- weather from a location hardcoded to TinkerSpace coordinates

The display behavior is also embedded in the frontend:

- maker cards and calendar are the only two first-class views
- some badge behavior depends on exact person-name matching
- copy, visual assets, and certain runtime rules are TinkerSpace-specific

This is a valid implementation for one installation, but it is not a strong long-term shape for reuse.

## Core problem

The current app is structured around the object itself, not the space around it.

In practice that means:

- provider contracts are leaking directly into UI code
- tenant identity is mixed into runtime behavior
- content management lives outside the product boundary
- operational policies are hardcoded into components
- there is no canonical display model that separates source data from presentation

As a result, another organization cannot reasonably adopt this by changing configuration alone. They would need code changes for branding, data integration, location behavior, and likely display modules.

## Design goal

Evolve this project so that:

- the current TinkerSpace deployment keeps working
- the frontend can render from a canonical display model
- organization-specific behavior moves into configuration, adapters, and admin-managed content
- future deployments can reuse the same display runtime without forking UI logic

The system should preserve the real kiosk constraints that matter:

- full-screen display behavior
- polling and resilience for unreliable upstream systems
- module rotation and scheduling
- organization-specific branding
- fallback behavior when sources fail
- support for private internal systems as upstream sources

## Recommended architectural direction

The recommended direction is:

- keep the screen frontend as a lightweight display runtime
- introduce a backend/BFF for configuration, normalization, and admin workflows
- move the current source-specific logic behind adapters
- move organization identity and policy into tenant configuration

This creates a cleaner separation between:

- source systems
- display domain model
- tenant configuration
- screen rendering
- admin and operational control

## Target architecture

### 1. Screen runtime

This remains the frontend application rendered on kiosk screens.

Responsibilities:

- render a canonical display payload
- manage local view transitions and animations
- handle screen-specific concerns like wake lock, fullscreen, and reconnection
- stay resilient when payloads are missing or stale

It should not:

- understand multiple upstream provider contracts directly
- hardcode tenant branding or operational policy
- own primary content management workflows

### 2. Backend / BFF

This becomes the main application boundary for the display platform.

Responsibilities:

- expose a canonical API for the frontend
- normalize upstream provider data into one display model
- store tenant configuration
- store display-native content such as announcements and fallbacks
- manage auth and roles for administrators
- apply per-screen and per-tenant policies
- support caching and recovery from upstream outages

This is the main architectural addition that the current Netlify-only setup does not have.

### 3. Adapter layer

Adapters translate upstream systems into the display platform’s canonical model.

Examples:

- current TinkerHub check-in API adapter
- current SpaceCalendar adapter
- future CSV, Google Calendar, Airtable, Notion, or internal API adapters

This is where source-specific quirks should live.

The frontend should not need to know whether people data came from:

- `/checkin/active`
- an internal admin app
- a CRM
- a spreadsheet

### 4. Tenant configuration

Tenant configuration defines how a given organization’s display behaves.

Examples:

- organization name
- brand assets
- theme and typography
- location and timezone
- weather location
- enabled modules
- rotation rules
- polling behavior
- API connections
- screen-level overrides

This should replace hardcoded organization assumptions in the frontend.

### 5. Admin dashboard

The admin dashboard should exist for organization users to log in and manage display behavior without code changes.

Recommended scope:

- branding and visual configuration
- enabled modules and layout choices
- screen registration and per-screen settings
- upstream connection settings
- announcements and curated content
- fallback content
- scheduling and rotation rules

This dashboard should not try to replace every upstream operational system.

For example, it does not need to become the primary system for:

- member check-in
- people management
- event authoring

Those can remain in external systems and feed the display platform through adapters.

## Canonical data model

The frontend should move toward one internal display contract instead of directly consuming multiple raw source contracts.

A simplified shape could be:

```json
{
  "tenant": {
    "name": "TinkerSpace",
    "timezone": "Asia/Kolkata",
    "theme": {
      "logoUrl": "",
      "accentColor": "",
      "darkModePolicy": "time-based"
    }
  },
  "screen": {
    "id": "screen-lobby-01",
    "layout": "makers-plus-calendar",
    "rotationPolicy": {
      "enabledModules": ["makers", "calendar", "announcements"]
    }
  },
  "modules": {
    "makers": {
      "items": []
    },
    "calendar": {
      "liveEvent": null,
      "upcomingEvents": [],
      "monthEvents": []
    },
    "announcements": {
      "items": []
    }
  },
  "meta": {
    "generatedAt": "",
    "stale": false
  }
}
```

The exact schema can evolve, but the important shift is that the UI should consume one internal model.

## Migration principle

This should be introduced without breaking the current deployment.

Recommended migration principle:

- if `REACT_APP_BACKEND_URL` is present, the frontend consumes the new backend payload
- if it is absent, the frontend can continue working in compatibility mode during migration

Important caveat:

This fallback is acceptable as a migration strategy, but it should not become a permanent excuse for keeping provider-specific logic spread across the UI forever.

The best long-term shape is:

- one frontend display contract
- adapters handling source differences
- backend as the stable integration boundary

## Hosting implications

The current app can continue to be hosted on Netlify as a static frontend.

However, once the project includes:

- admin login
- saved tenant configuration
- adapter execution
- connection secrets
- canonical payload generation

it will require backend hosting as well.

Recommended deployment split:

- Netlify for the kiosk frontend
- separate backend hosting for API, auth, adapters, and admin services
- persistent database for tenant configuration and display-native content

This does not require abandoning the current Netlify deployment. It requires adding the missing server-side layer.

## What should stay true during the shift

The architectural shift should preserve these constraints:

- TinkerSpace must continue to work as a first-class deployment
- the display must remain resilient even when upstream APIs fail
- fallback and cached content should be supported
- private internal systems should remain valid upstream integrations
- configuration should replace hardcoding wherever possible
- the frontend should become simpler, not more coupled

## Proposed phases

### Phase 1: Create a canonical frontend data boundary

- introduce a canonical display payload inside the frontend
- adapt current direct API responses into that shape
- isolate TinkerSpace-specific constants and display policy

### Phase 2: Add backend as an optional integration boundary

- add backend support through `REACT_APP_BACKEND_URL`
- move normalization and tenant configuration server-side
- keep compatibility mode for current direct sources during migration

### Phase 3: Introduce admin-managed configuration

- add organization login
- add tenant configuration management
- add screen settings and display-native content management

### Phase 4: Move source-specific logic fully behind adapters

- retire direct provider-specific UI fetch paths
- make the backend the stable API boundary for the frontend

## What this project becomes after the shift

If the shift is implemented well, this project becomes:

- a reusable display runtime for TinkerHub and beyond
- a platform that can support multiple organizations without frontend forks
- a system where content, policy, and presentation are cleanly separated

Most importantly, it becomes modular without pretending the real operational complexity does not exist.

That is the key design standard for this shift:

- do not remove complexity by hiding it
- move complexity into the correct boundaries
- keep the screen runtime narrow and robust

