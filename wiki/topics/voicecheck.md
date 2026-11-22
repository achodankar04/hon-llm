---
title: "VoiceCheck"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-ui-exploration-2026-11-20]]"
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
  - "[[sources/voicecheck-microservices-design-2026-11-20]]"
tags:
  - voicecheck
---

# VoiceCheck

Main topic for Honeywell VoiceCheck, a voice-driven inspection application for
vehicles and equipment.

- Inspectors complete checklists by voice on a Talkman device or with the Android
  Inspection app.
- Results are reviewed in a web UI and exported to host systems.
- The version studied is 1.12, built on the Vocollect EPP framework.

Pages live in `wiki/topics/voicecheck/`.

## The application

- [[voicecheck-application-features]] — what each screen does and the end-to-end inspection flow.

## The code

- [[voicecheck-codebase-overview]] — repositories, stack, modules, files on disk, configuration.
- [[voicecheck-device-protocol]] — the JSON command API used by Talkman and Android devices.
- [[voicecheck-integrations-and-pipelines]] — host APIs, assignment life cycle, export, transcription, OCR, reports, jobs.
- [[voicecheck-database-schema]] — the 88 legacy tables, the multi-site model, growth and query hot spots.
- [[epp-platform-framework]] — what the EPP framework provides and how VoiceCheck depends on it.
- [[voicecheck-performance-and-scaling-constraints]] — hot paths and what stops the monolith scaling.

## Modernisation

- [[voicecheck-microservices-architecture]] — proposed split into seven services on Catalyst Fabric.
- [[voicecheck-target-data-model]] — proposed PostgreSQL tables per service and the legacy-table mapping.

## Related

- [[vcng]] — the Catalyst Fabric services whose conventions the proposal follows.

Sources: [[sources/voicecheck-ui-exploration-2026-11-20]],
[[sources/voicecheck-codebase-analysis-2026-11-20]],
[[sources/voicecheck-microservices-design-2026-11-20]].
