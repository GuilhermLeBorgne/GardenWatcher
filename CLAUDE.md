# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

GardenWatcher is at the **architecture/specification stage**. There is no source code, tech stack, build system, linter, or test suite yet. Do not assume any language or framework: the stack is decision point 8, still open. Once it's chosen, update this file with the build/lint/test commands, including how to run a single test.

Write specs and discussion in French to match the README unless asked otherwise.

## Architecture decision workflow

- `docs/architecture/TODO.md` is the ordered checklist of the 16 architecture decisions, which follow their dependencies. It also holds requirements gathered along the way (under each point) and an « Idées pour plus tard » section for features explicitly deferred past V1. Read it first to see what's decided and what's open.
- `docs/architecture/adr/` has one ADR per closed decision (`NNNN-slug.md`, template in `0000-template.md`). When a decision is made: write the ADR, tick the point in `TODO.md` with a link to the ADR, and move any deferred items to « Idées pour plus tard ».
- The owner makes the decisions. Propose a recommendation with trade-offs, then record only what they confirm.

## Key decisions so far (details in the ADRs)

- **Deployment:** a self-hosted server at the user's home. For now it's an old x86 Ubuntu PC without a GPU; a Raspberry Pi (ARM) is the eventual target, so dependencies must run on both. First users are the owner and their partner (two accounts, one shared garden), with a product or paid tier kept possible later. Nothing garden-specific may be hard-coded: it all goes in configuration (ADR-0001).
- **Offline:** garden data lives on the server. Entities and the task calendar keep working without Internet; features that need the Internet degrade gracefully (ADR-0002).
- **Every external service sits behind a swappable interface:** AI, weather, water restrictions, messaging. This is what makes TDD possible (test doubles, recorded responses) and lets a provider be replaced later.
- **AI (Claude API) is used only for:** (A) enriching an entity via web search into a schema-validated species sheet with sources, and (B) daily task planning (ADR-0003).
  - Calculations, alerts and data fetching are deterministic code, not AI.
  - The AI proposes and the user validates. Validation mode is manual, auto, or hybrid (first occurrence manual, then auto).
  - The API key is in a config file, with a budget cap in config. No RAG and no chat in V1.
- **Messaging:** Telegram bot, send-only in V1 (no inbound port, no actions from Telegram), with a persistent outgoing queue (ADR-0004).
- **Weather:** Open-Meteo (non-commercial licence, CC BY 4.0 attribution required), refreshed every 3 hours, with alert thresholds in config (ADR-0005).
- **Water restrictions:** VigiEau. The tool informs and never enforces: the user chooses whether to apply a restriction and enters the forbidden time slots themselves (ADR-0005).
- **Watering modes** (simple / renforcé / expert) are set per entity. Tasks are scheduled within each user's availability slots (TODO points 10–11).

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