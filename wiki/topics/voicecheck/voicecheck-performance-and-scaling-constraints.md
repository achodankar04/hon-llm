---
title: "VoiceCheck performance and scaling constraints"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - legacy
  - performance
---

# VoiceCheck performance and scaling constraints

Source: [[sources/voicecheck-codebase-analysis-2026-11-20]]. Costs were derived by
reading the code; nothing was measured. *(unverified)* marks items that also depend
on configuration not seen.

## Hot paths

| Path | What one call costs today |
|---|---|
| `postResult` (once per spoken step) | One read-write transaction with about 6–9 `SELECT`s (operator; assignment with 4 formula sub-queries; check; its two condition sets; step), then deletes and inserts of result and note rows, a versioned update and an explicit flush |
| `getWork` (once per start or resume) | Loads every host text, then for each check lazily loads the step, args, localisations and two condition sets (N+1 queries) and copies the static version map; clears and reloads the session; writes the assignment |
| `getAssignmentTypes`, assignment grid | Filter and sort on the formula columns, so correlated sub-queries run for every row |
| Voice plan grid | Loads the whole assignment graph and all its notes into memory |
| Assignment creation | Inserts one check per plan step, plus prompt-data and condition rows |

## UI grids

- Each request loads the full result set (up to 5,000 rows), then pages, sorts some columns, filters and builds tooltips in memory.
- Every open grid re-runs its query every 10 seconds.
- The UI polls for unacknowledged notifications every 60 seconds.

## Synchronous work on request threads

- OCR token estimate and extraction.
- E-mail for critical notifications, sent inside the caller's transaction.
- A fresh fetch of the identity provider's signing keys on every SSO token check.
- A Keycloak token fetch when its cache is cold.
- The PDF report, including recompiling its templates.
- REST calls with Basic authentication pay a BCrypt (cost 12) check on every call.

## Background work

| Worker | How it runs today |
|---|---|
| Export | Runs only on the load balancer's primary node and polls every second. The HTTP call to the host is made inside the database transaction. Retries every 10 s with no limit, no backoff and no dead-letter handling. |
| Transcription | Threads on every node poll every second with an in-memory pending set, so in a cluster the same note can be processed (and billed) twice. Read timeout is 60 s. |
| Quartz | In memory with 5 threads. Purge and transcription jobs run on every node, and cluster coordination uses database ping/poll loops that occupy Quartz threads. |

## In-process locks and threads

| Mechanism | Detail |
|---|---|
| `Semaphore(25)` | Static and blocking; limits concurrent host assignment calls |
| `synchronized nextAutoAssignmentId` | Per JVM |
| Form-version cache | Static map with a global lock and per-form semaphores |
| File depot | `buildDirPath` is synchronised and called on every file write |
| Vocabulary cache, Sphinx recogniser, bearer-token cache | Synchronised |
| Voice sessions | Unsynchronised static map |
| Export | 1 loader + 2 workers |
| Transcription | Loader, saver and sender threads |
| Role-map and version change listeners | Poll the database every second (clustered mode) |

## What prevents horizontal scaling

1. Voice sessions live in one JVM, so requests must route to the same node per operator.
2. Licence leases are counted per JVM.
3. The form-version cache is static and reloaded by polling, and parsing a version writes to the database. *(unverified)* On a single node the cache may be empty after a restart.
4. The vocabulary cache is per node and never invalidated.
5. Every file (audio, photos, templates, forms, images, APK, checklists, vocabulary, problem logs) is on local disk.
6. The export worker runs only on the primary node.
7. Transcription runs on every node.
8. Quartz uses an in-memory job store.
9. UI HTTP sessions hold detached entities, OpenSessionInView holds a connection, and checklist-upload state is kept in the session.
10. `server.name` is baked into speaker IDs and photo URLs.
11. Jasper writes into the web-app folder and uses Windows paths.

## Reliability defects found (non-security)

- **Duplicate loop checks:** two concurrent `postResult` calls for a new loop iteration can create duplicate checks, because there is no unique key.
- **Version-number race:** two concurrent publishes of a form version can pick the same version number.
- **No transaction timeout on device commands:** the attribute `-timeout_15` is parsed as a rollback rule, not a timeout.
- **Notification noise:** every device business error creates a notification row, including routine outcomes such as "no assignment available".
- **Transcription starvation:** notes stuck in the transcription pending set can starve the first 100 rows.
- *(unverified)* **Export update job never scheduled:** its trigger is mis-wired in `voicecheckJobs.xml`.
- **Non-atomic host create:** a plan-based create from a host runs in two transactions. *(unverified)* It may also flush a change to the plan's form version.

## Related

- [[voicecheck-microservices-architecture]] — how the proposed design removes these constraints.
- [[voicecheck-database-schema]]
- [[voicecheck-device-protocol]]
- [[voicecheck]]
