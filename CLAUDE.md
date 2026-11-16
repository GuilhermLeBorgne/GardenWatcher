# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

GardenWatcher is at the **architecture/specification stage**. There is no source code, build system, linter, or test suite yet.

The stack is chosen (ADR-0007):
- **Backend:** Python 3.12+, FastAPI, Pydantic, SQLite (SQLAlchemy 2 and Alembic), APScheduler, httpx, pytest (with respx and time-machine), ruff, uv.
- **Frontend:** TypeScript with no framework, Tailwind and Leaflet (TODO point 14).

Once the project is initialized, update this file with the run/test/lint commands, including how to run a single test.

Write specs and discussion in French to match the README unless asked otherwise.

## Code organisation and language (ADR-0008)

**Process and modules:**
- Everything runs as a single process (API and scheduler), with one web server worker so scheduled jobs don't run twice.
- Modules: `garden`, `knowledge`, `tasks`, `planning`, `weather`, `water`, `notifications`, `users`, `ai`, `core`.
- Each module has four layers:
  - `domain/`: pure business rules with no I/O. Most TDD happens here.
  - `services/`: use cases.
  - `adapters/`: database and HTTP clients.
  - `api/`: FastAPI routes.
- A module never touches another module's tables; it goes through that module's services.
- There's no internal event bus in V1: scheduled jobs chain the steps.

**Language:**
- **Code is in English.**
- **The UI, Telegram messages, AI-generated content and documentation are in French.**
- User-facing strings live in dedicated files, never inline, so that internationalisation stays cheap later.

**Glossary (French domain term → name in code):**

| Français | Code |
|---|---|
| Jardin | `Garden` |
| Entité | `Entity` |
| Relation entre entités | `EntityLink` |
| Statut | `status` |
| Fiche de connaissances | `KnowledgeSheet` |
| Surcharge (valeur ajustée sur l'entité) | `override` |
| Brouillon / Validée | `draft` / `validated` |
| Tâche | `Task` |
| Tâche proposée | `proposed` (task status) |
| Mode de validation (manuel, auto, hybride) | `ValidationMode` (`manual`, `auto`, `hybrid`) |
| Mode d'arrosage (simple, renforcé, expert) | `WateringMode` (`simple`, `enhanced`, `expert`) |
| Créneau de disponibilité | `AvailabilitySlot` |
| Restriction d'eau | `WaterRestriction` |
| Arrêté (préfectoral) | `Decree` |
| Source d'eau | `WaterSource` |
| Alerte / Seuil | `Alert` / `threshold` |
| Période sensible pour la faune (nidification) | `WildlifeSensitivePeriod` (`nesting`) |
| Utilisateur | `User` |

Update the glossary whenever a new domain concept appears.

## Architecture decision workflow

- `docs/architecture/TODO.md` is the ordered checklist of the 16 architecture decisions, which follow their dependencies. It also holds requirements gathered along the way (under each point) and an « Idées pour plus tard » section for features explicitly deferred past V1. Read it first to see what's decided and what's open.
- `docs/architecture/adr/` has one ADR per closed decision (`NNNN-slug.md`, template in `0000-template.md`). When a decision is made: write the ADR, tick the point in `TODO.md` with a link to the ADR, and move any deferred items to « Idées pour plus tard ».
- The owner makes the decisions. Propose a recommendation with trade-offs, then record only what they confirm.

## Key decisions so far (details in the ADRs)

- **Deployment:** a self-hosted server at the user's home. For now it's an old x86 Ubuntu PC without a GPU; a Raspberry Pi (ARM) is the eventual target, so dependencies must run on both. First users are the owner and their partner (two accounts, one shared garden), with a product or paid tier kept possible later. Nothing garden-specific may be hard-coded: it all goes in configuration (ADR-0001).
- **Offline:** garden data lives on the server. Entities and the task calendar keep working without Internet; features that need the Internet degrade gracefully (ADR-0002).
- **Every external service sits behind a swappable interface:** AI, weather, water restrictions, messaging, TAXREF. In code, each is a `typing.Protocol` with a real implementation and a test double. This is what makes TDD possible and lets a provider be replaced later.
- **Time is an injected dependency** (a clock), never read directly, so scheduling and alert tests can freeze it.
- **AI (Claude API) is used only for:** (A) enriching an entity via web search into a schema-validated species sheet with sources, and (B) daily task planning (ADR-0003).
  - Calculations, alerts and data fetching are deterministic code, not AI.
  - The AI proposes and the user validates. Validation mode is manual, auto, or hybrid (first occurrence manual, then auto).
  - The API key is in a config file, with a budget cap in config. No RAG and no chat in V1.
- **Messaging:** Telegram bot, send-only in V1 (no inbound port, no actions from Telegram), with a persistent outgoing queue (ADR-0004).
- **Weather:** Open-Meteo (non-commercial licence, CC BY 4.0 attribution required), refreshed every 3 hours, with alert thresholds in config (ADR-0005).
- **Water restrictions:** VigiEau. The tool informs and never enforces: the user chooses whether to apply a restriction and enters the forbidden time slots themselves (ADR-0005).
- **Knowledge base (ADR-0006):**
  - Knowledge sheets are shared per species and form the offline knowledge base. Each entity can override any value of its sheet.
  - The AI's web search is limited to a configurable list of trusted domains. Species identity comes from TAXREF (INPN), without AI.
  - AI-generated sheets stay as drafts until the user validates them, and an entity only enters planning once its sheet is validated.
  - Planning respects wildlife-sensitive periods (e.g. no hedge pruning during nesting) by warning, never blocking.
- **Planning engine (ADR-0009):**
  - **Daily pipeline over a 7-day horizon:** needs (from rules, AI and manual tasks) → constraints → slot placement → validation mode → notification.
  - **AI is only an advisor:** rules keep planning when it's unavailable.
  - **The engine never alters a task the user validated or edited:** it proposes a change instead.
  - **Every task records its source and justification.**
  - **Watering modes are set per entity:** simple or enhanced in V1, expert later. Simple-mode watering is grouped into one daily notification; enhanced mode sends one notification per entity.
  - **Hybrid validation:** the hybrid "task type" is action + entity. The validation mode is set per entity, with a garden default.
  - **Tasks are unassigned in V1:** they're placed where at least one user is available, and notifications go to one shared Telegram group.

## Product concept (from README.md)

A garden-management tool that maintains a knowledge base of the user's garden and feeds it to an AI agent that decides which maintenance actions to take, respecting seasons and local fauna/flora while making the best use of the user's available time. The UI is meant to be simple, in the style of Claude Desktop.

Core domain model:
- **Entité**: anything in the garden: a specific tree, a grove, a hedge, a potted plant, or a piece of ground or structure (greenhouse, raised bed, vegetable square, compost heap). Each one has a user-chosen unique identifier plus details (species, variety, age, …).
- **Relations** between entities (the user can link several entities together).
- **Statuts** of an entity: `En cours de création`, `Plantée`, `Malade`; the rest are still to be defined.
- **Tâches**: scheduled maintenance. The tool suggests and plans, it doesn't impose. Users can create, edit (e.g. move the date), delete, and mark tasks done.

User flow: create entity → enter ID and details → validate (optionally link to other entities) → optionally have the tool look up the entity online and add what it finds → the tool then handles planning, notifications, things worth remembering, and weather alerts.

Requested features:
- A map placing entities and showing their statuses.
- Short- and medium-term weather data (precipitation, humidity, …) used to work out the best windows for watering, planting, and harvesting.
- An offline knowledge base.
- Notifications for tasks due and weather alerts (frost, wind, heat, drought).
- A task calendar.

The README leaves two questions open: which information sources are reliable (TODO point 5), and the full list of entity statuses (point 11). RAG is ruled out for V1 (ADR-0003).


# TDD Development Rules

## Core Principles
1. **Tests first**: Any new feature must have failing tests first
2. **Minimal implementation**: Only write code just enough to pass tests
3. **Continuous refactoring**: Consider refactoring after each green light

## Development Flow
1. Receive requirement → Write tests first
2. Confirm test fails (Red)
3. Write minimal code to pass tests (Green)
4. Refactor (keep Green)
5. Repeat

## Forbidden
- ❌ Don't write feature code without tests
- ❌ Don't delete or skip existing tests
- ❌ Don't write "tests later" code