# Knowledge Base Index

This catalog is maintained after every ingest, filed query, and lint pass.

## Getting started

- [[README]] — How to operate this vault.
- [[AGENTS]] — The operating schema for the LLM maintainer.

## Sources

- [[karpathy-personal-knowledge-bases]] — Persistent LLM-maintained wiki pattern that inspired this vault.
- [[vcng-local-dev-setup-session-2026-11-17]] — Observed local DM/MFE setup session (evidence for the local-dev how-tos).
- [[voicecheck-ui-exploration-2026-11-20]] — Read-only walk-through of a lab VoiceCheck 1.12 server.
- [[voicecheck-codebase-analysis-2026-11-20]] — Code reading of the VoiceCheck and EPP repositories.
- [[voicecheck-microservices-design-2026-11-20]] — Proposal document for splitting VoiceCheck into microservices.

## Topics

- [[vcng]] — VCNG main topic (Device Management, Operator Management, Voice Console MFE).
  - [[run-device-management-locally-with-dev-db]] — Start the DM service locally against the dev database.
  - [[run-voiceconsole-mfe-with-local-dm-and-op]] — Run the Voice Console MFE against local DM and OP services.
- [[voicecheck]] — VoiceCheck main topic (legacy voice inspection app: features, code, database, modernisation).
  - [[voicecheck-application-features]] — What each screen does and the inspection life cycle.
  - [[voicecheck-codebase-overview]] — Repositories, stack, packages, files on disk, configuration.
  - [[voicecheck-device-protocol]] — JSON command API used by Talkman and Android devices.
  - [[voicecheck-integrations-and-pipelines]] — Host APIs, assignment states, export, transcription, OCR, reports, jobs.
  - [[voicecheck-database-schema]] — The 88 legacy tables, multi-site model, growth and hot spots.
  - [[epp-platform-framework]] — What the EPP framework provides and how VoiceCheck depends on it.
  - [[voicecheck-performance-and-scaling-constraints]] — Hot paths and what stops the monolith scaling.
  - [[voicecheck-microservices-architecture]] — Proposed seven-service design on Catalyst Fabric.
  - [[voicecheck-target-data-model]] — Proposed tables per service and where each legacy table goes.

## Entities

_No entities created yet._

## Projects

_No projects created yet._

## Decisions

_No decisions recorded yet._

## Questions and comparisons

_No filed questions or comparisons yet._
