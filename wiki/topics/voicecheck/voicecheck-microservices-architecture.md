---
title: "VoiceCheck microservices architecture (proposal)"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-microservices-design-2026-11-20]]"
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - architecture
---

# VoiceCheck microservices architecture (proposal)

**Status:** proposal v1 of 2026-11-20. Nothing has been agreed or built.

Source: [[sources/voicecheck-microservices-design-2026-11-20]]. The table definitions
are in [[voicecheck-target-data-model]].

## Services

| Service | Owns | Why it is separate |
|---|---|---|
| `vi-device-gateway` | nothing persistent (Redis only) | Device-facing entry point. Keeps the legacy `voiceAppCommand` contract so the Talkman task and Android app work unchanged; adds a v2 REST API with batch results and direct blob uploads. Validates sessions in Redis; enforces idempotency, rate limits and size limits. Highest request rate, stateless. |
| `vi-operator-service` | operators, voice access codes (Argon2id), device sessions, break types and break events, speech templates, vocabulary, mobile app releases, device problem reports, licence leases (through the platform) | Workforce and device runtime; burst load at shift start |
| `vi-design-service` | VoiceForms, steps, conditions, list items, prompt templates, plans, translations, images, published form versions, **compiled plan releases** | Authoring; low write volume; owns the immutable execution packages |
| `vi-execution-service` | assignments, checks and results, submissions, numbering (Postgres sequence), local copies of work packages and operators | The write-hot core: one write per spoken step |
| `vi-voicenote-service` | voice notes, memos, photos, transcription orchestration | Media and an asynchronous, rate-limited, billed external call |
| `vi-results-service` | push/pull result delivery, legacy XML and SOAP host adapters, partitioned result facts for BI, asynchronous PDF reports | Integration and output; keeps slow hosts and heavy reports away from the hot path |
| `vi-admin-service` | typed settings, notifications and asynchronous e-mail, job registry and history, per-user grid preferences | Leftover EPP administration; may shrink if the platform has equivalents |

**Frontend:** one micro-frontend, `pss_cf_mfe_voicecheck`, in the Catalyst Fabric host shell. It replaces about 240 Struts actions and the FreeMarker pages.

## Platform reuse (instead of EPP)

| EPP capability | Target |
|---|---|
| Authentication | Keycloak realm `catalystfabric` |
| Authorisation | Platform permission check per endpoint (`@RequiresPermission`, keys such as `viAssignment:review`) |
| Users and sites | Keycloak users and the platform site service (`site.operations` events) |
| Licensing | Platform licensing service; a Redis lease set is the fallback |
| Files | Storage service (upload request → SAS URL) |
| Server logs | OpenTelemetry → central logging |
| Grid framework and navigation | Micro-frontend components and the host shell |
| Unused EPP parts | Dropped |

## Design principles

1. **One owner per table.** No service reads another service's database.
2. **Local hot path.** `getWork` and `postResult` touch one service and one database. Compiled plans, operator names and sites are copied in through events.
3. **Immutable runtime inputs.** A release is never changed, so it can be cached without invalidation.
4. **Stateless pods.** No sticky sessions and no "primary node".
5. **Asynchronous work through an outbox.** Background work is driven by a transactional outbox and Kafka, with retries, backoff and dead-letter queues. No external call is made inside a database transaction.
6. **Platform conventions throughout.** `/api/<svc>`, `ServiceResponse`, page responses, Flyway, schema per tenant.
7. **Legacy contracts at the edge.** Old device and host formats are translated in the gateway and the results service.

## Boundaries deliberately kept together

| Kept together | Why |
|---|---|
| Plans with forms | Validation needs steps, conditions and versions together, and both change at authoring pace |
| Checks, results, notes, prompt data and metadata with assignments | One aggregate written in one transaction: 7 legacy tables become 1 row per check |
| Translations with design | Every text belongs to a form, list item, plan or break type, and translations are compiled into releases |
| Reports, BI facts, delivery and host adapters | All of it is results leaving the system, with similar slow or bursty external behaviour |
| Transcription with voice notes | Transcription is a state of a voice note |

## Key flows

- **Sign-on:**
  1. The gateway calls the operator service, which checks the hash or token.
  2. In one transaction, it ends any other active session on the device and takes a licence lease.
  3. It emits "session ended" for the replaced operator; the execution service then releases that operator's in-progress assignments.
- **Start work:** the execution service updates the assignment (status, operator, token) and reads its checks by primary-key range. The work package comes from an in-memory cache. In parallel, the gateway fetches memos from the voice-note service.
- **Record a result:** two primary-key reads (assignment first, for lock order), two `UPDATE`s (check, then counters), and an optional outbox insert, in one short transaction.
- **Submit to delivery:**
  1. Execution emits `assignment.submitted`.
  2. The results service batch-fetches the transcriptions. If any are missing, it waits for `voicenote.transcribed`.
  3. It posts to the host outside any transaction, with a circuit breaker and backoff, and emits the delivery outcome.
  4. Execution then moves ExportPending → Complete.
- **Plan release:** the design service validates and compiles the plan, then publishes it to a compacted topic. Execution stores it as a work package. New assignments use the new release; running assignments keep theirs.

## Event catalogue

| Topic | Key | Producer → consumers |
|---|---|---|
| `vi.design.form-version.published`, `vi.design.plan-release.published` (compacted) | ID | design → execution |
| `vi.design.plan.retired` | planId | design → execution |
| `vi.operator.changed` (compacted) | operatorId | operator → execution, voicenote, results |
| `vi.operator.session` | operatorId | operator → execution, gateway |
| `vi.exec.assignment.changed`, `.submitted`, `.removed` | assignmentId | execution → results, voicenote |
| `vi.voicenote.transcription.work` (with retry topics and a DLQ) | voiceNoteId | voicenote → its own workers |
| `vi.voicenote.transcribed` | assignmentId | voicenote → results |
| `vi.results.delivery.outcome` | assignmentId | results → execution, admin |
| `vi.admin.config.changed` (compacted), `vi.admin.job.command` | key | admin → all |
| `vi.platform.notification.raised`, `vi.platform.job.run` | key | all → admin |
| `site.operations` | siteId | platform → all |

Topics are keyed by aggregate ID, not by tenant, so load spreads across partitions while order is kept per aggregate. Tenant, site and correlation IDs travel in headers.

## Performance fixes

| Legacy constraint ([[voicecheck-performance-and-scaling-constraints]]) | Fix |
|---|---|
| `getWork` N+1 | Immutable work package compiled at release time, cached per package and locale |
| `postResult` read-modify-write across 3 tables | Primary-key reads and two updates; results stored on the check row; `fillfactor 80` for HOT updates |
| Formula sub-queries | Stored counters on the assignment plus a generated `percent_complete` column |
| In-memory grids with 10 s polling | Keyset paging, indexed server-side filters, capped counts, `updatedSince` delta polling (SSE later) |
| Tag-join site filter | `site_id` column with `(site_id, status, created_at)` indexes |
| `like '%digits'` | Reversed work-ID column with a `varchar_pattern_ops` index |
| In-JVM state and local files | Redis, the platform licensing service, the storage service |
| Primary-node export loop | `FOR UPDATE SKIP LOCKED` workers on any pod |
| Transcription on every node | Kafka partition ownership |
| Uncached properties, synchronous e-mail | Cached settings with change events; asynchronous e-mail |
| Synchronous OCR and PDF | Asynchronous jobs |
| Heavy purge | Partition drops and indexed batch deletes |
| `int_*` growth | Metadata stored once per row as `jsonb`, monthly partitions |
| JVM locks | Postgres sequence, bulkheads, cached permission decisions |

**Capacity targets (assumptions to validate):**
- 10,000 concurrent operators and about 2,000 `postResult` calls per second at peak.
- `postResult` p95 under 40 ms; `getWork` p95 under 150 ms cold and 60 ms warm.
- 1,000 sign-ons within 5 minutes.

**Scaling:**
- Gateway and execution: HPA with a minimum of 2 pods.
- Workers: KEDA on Kafka lag.
- Database: dedicated compute for execution; PgBouncer in front of the per-tenant connection pools.

## Migration and roadmap

**Migration:** per-tenant ETL.
1. Map legacy sites to Catalyst Fabric site UUIDs.
2. Move users to Keycloak; convert voice access codes to the new hash format.
3. Load forms and plans, and turn version files into rows, then compile one release per plan.
4. Collapse checks and their child tables.
5. Upload files to the storage service.
6. Reconcile counts and checksums.

At cutover per customer, devices keep their legacy URL paths through an ingress rewrite.

**Roadmap:**

| Phase | Scope |
|---|---|
| 0 | Foundations and recorded golden tests of the device protocol |
| 1 | Authoring (design service) |
| 2 | Runtime core: operator, execution and gateway services, running with unmodified devices |
| 3 | Media and transcription |
| 4 | Results and admin services |
| 5 | Migration and pilot customer |
| 6 | Optimisation: protocol v2, SSE, partition tuning |

## Open questions

- Real load figures: tenants, operators, assignments per day, steps per form.
- Whether the Talkman task and Android app can send a device token before sign-on.
- Whether Redis and a licensing lease API are available on the platform.
- Whether the Databricks-hosted Postgres supports the partitioning and indexing the design needs.
- Whether to converge operators with VCNG operator management later.
- How BI consumers currently read the `int_*` tables.
- Whether an on-premises edition must continue.

## Related

- [[voicecheck-target-data-model]]
- [[epp-platform-framework]]
- [[vcng]]
- [[voicecheck]]
