# Knowledge Base Activity Log

Append-only record of ingests, queries, filed answers, and lint passes.

## [2026-11-13] setup | Knowledge base initialized
- Input: Karpathy's personal knowledge-base pattern
- Pages created/updated: [[README]], [[AGENTS]], [[index]], [[log]], [[karpathy-personal-knowledge-bases]]
- Notes: Created the initial raw-source, wiki, index, and activity-log workflow.

## [2026-11-17] ingest | VCNG local dev setup (DM + Voice Console MFE)
- Input: Copilot session observations (no file in `raw/`)
- Pages created/updated: [[vcng-local-dev-setup-session-2026-11-17]], [[run-device-management-locally-with-dev-db]], [[run-voiceconsole-mfe-with-local-dm-and-op]], [[index]]
- Notes: Local OP service run is inferred, not verified.

## [2026-11-17] restructure | Group VCNG how-tos under a main topic
- Input: user request
- Pages created/updated: [[vcng]], [[run-device-management-locally-with-dev-db]], [[run-voiceconsole-mfe-with-local-dm-and-op]], [[index]]
- Notes: The two how-tos moved to `wiki/topics/vcng/`; `wiki/topics/vcng.md` is the main topic page.

## [2026-11-21] ingest | VoiceCheck application, codebase and microservices design
- Input: Copilot sessions of 2026-11-20 (UI walk-through, code reading, design proposal); no file in `raw/`
- Pages created/updated: [[voicecheck-ui-exploration-2026-11-20]], [[voicecheck-codebase-analysis-2026-11-20]], [[voicecheck-microservices-design-2026-11-20]], [[voicecheck]], [[voicecheck-application-features]], [[voicecheck-codebase-overview]], [[voicecheck-device-protocol]], [[voicecheck-integrations-and-pipelines]], [[voicecheck-database-schema]], [[epp-platform-framework]], [[voicecheck-performance-and-scaling-constraints]], [[voicecheck-microservices-architecture]], [[voicecheck-target-data-model]], [[index]]
- Notes: This repository is public, so at the user's request security findings, the lab server address, internal repository URLs and all credentials were left out. Items marked (unverified) were inferred from code; the architecture is a proposal.
