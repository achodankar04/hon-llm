---
title: "VoiceCheck target data model (proposal)"
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
  - database
---

# VoiceCheck target data model (proposal)

These are the proposed PostgreSQL tables per service for
[[voicecheck-microservices-architecture]]. The exact DDL is in the design document
([[sources/voicecheck-microservices-design-2026-11-20]]). It parses with
PostgreSQL's parser but has not been run on a database.

## Conventions

- **Tenancy:** one database per service and one schema per tenant (the organisation UUID without hyphens), migrated by Flyway at startup. Business tables have no `tenant_id` column.
- **Sites:** site-scoped rows carry `site_id uuid` (the Catalyst Fabric site). This replaces `voc_tag` and every `*_tag` join table.
- **IDs:**
  - Aggregate roots use UUIDv7, generated in the application and time-ordered.
  - Child and append-only rows use `bigint` identity.
  - Legacy numeric status codes are kept wherever devices or hosts depend on them.
- **Audit columns** on mutable tables: `created_at`, `created_by`, `updated_at`, `updated_by`, and `version` (optimistic lock).
- **Shared schema** in every service database, with a `tenant_id` column so one relay or scheduler serves all tenants:
  - `outbox_event`: tenant, aggregate, event type, topic, key, payload, headers, published_at;
  - `inbox_event`: event ID and consumer, for de-duplication;
  - `shedlock`.
- **Site copy:** every service keeps `site_ref` (site_id, name, time_zone, site_path), fed by `site.operations`.

## vi-operator-service

| Table | Main columns and constraints |
|---|---|
| `operator` | operator_identifier (unique, case-insensitive), display_name, keycloak_user_id (unique, optional), access_code_hash, access-code change fields, preferred_locale, working_site_id, status, failed_attempts, locked_until, settings `jsonb` |
| `operator_site` | (operator_id, site_id) |
| `device_session` | id (= device session ID), operator, device, site, client type, app version, locale, auth method, lease reference, signed_on/last_seen/ended times, end reason. Partial unique indexes allow one active session per operator and one per device |
| `break_type` | break_number (unique), name, translations `jsonb`, active |
| `break_event` | session, operator, break type, site, start and end (new labour data) |
| `speech_template` | operator, locale, recognizer, word, storage file GUID, size, hash; unique (operator, locale, recognizer, word) |
| `vocabulary_config` | locale (PK), threshold, words `text[]` |
| `mobile_app_release` | platform, version name and code, storage file GUID, hash, status (staged/active/retired); at most one active per platform |
| `device_problem_report` | device, operator, site, storage file GUID, file name, size, uploaded_at |

## vi-design-service

| Table | Main columns and constraints |
|---|---|
| `voice_form` | name (unique, case-insensitive), description, status, latest_version |
| `localized_text`, `localized_text_translation` | Default text with context and optional host key; translation per (text, locale) |
| `list_item_group`, `list_item` | Group code (unique); item code, order and text; unique (group, item code) |
| `prompt_template` | name (unique), description, prompt type, params `jsonb` |
| `image_asset` | Form, folder path, file name, storage file GUID, MIME type, size, hash |
| `step` | Form, step_identifier (unique per form), display identifier, description, prompt type, prompt/help/display/reference text IDs, reference URL, template, list item group, params `jsonb` (replaces `form_args`), image, revision |
| `condition`, `step_condition` | Form, description, left step identifier, operator, operand `jsonb`; step ↔ condition |
| `voice_form_version` | (form, version_no) PK, status published/retired, self-contained content `jsonb`, content hash, published at/by |
| `plan` | plan_identifier (unique), description text, form and bound version, status draft/released/retired, latest release number |
| `plan_section` | Plan, section identifier, description, order, part fields, loop flag, looping prompt text, condition_ids `uuid[]` |
| `plan_step` | Section, step identifier, sequence counter (unique per section), order, mandatory, prompt_data `jsonb`, condition_ids `uuid[]` |
| `plan_release` | Plan, release number, form and version, compiled `jsonb` work package, total steps, etag, released at/by |
| `checklist_import_job` | OCR import state: source file, status, estimated and used tokens, resulting form, error, expiry |
| `transfer_job` | Asynchronous imports and exports: kind, direction, status, input and output files, summary, errors |

## vi-execution-service

| Table | Main columns and constraints |
|---|---|
| `assignment_number_seq` | Sequence, seeded above the legacy maximum |
| `form_version` | Copy of published versions (used to compile ad-hoc host packages) |
| `work_package` | Plan release or ad-hoc package: plan fields, form name and version, compiled `jsonb`, total steps, etag (unique for ad-hoc packages), `is_current` (one per plan) |
| `operator_ref` | Copy of operator identifier, name and status |
| `assignment` | site_id, assignment_identifier (unique), work_id (unique), work_id_rev (spoken-digit lookup), work_package_id, plan and form fields, status (legacy 1–6), operator, device, work_token, reviewer, source, start and end, **counters** (total, completed, unsubmitted, time spent), generated `percent_complete`, metadata `jsonb`. Indexes: (site_id, status, created_at), (operator_id, status), (site_id, work_id_rev `varchar_pattern_ops`), (status, ended_at) |
| `assignment_check` | PK (assignment_id, check_no); section identifier, iteration (0 = no loop), description, part fields, sequence counter, order, step identifier and description, prompt type, mandatory, status (legacy 1–3), result_values `text[]`, notes / prompt_data / metadata `jsonb`, operator, times, time spent, submission, modified by/at. Unique (assignment, section, iteration, sequence counter) prevents duplicate loop clones. `fillfactor 80` |
| `submission` | Assignment, submission number, submitted-by type and value, check count, state (open/delivered/cancelled) |
| `assignment_event` | Append-only audit of life-cycle transitions with actor and details |

## vi-voicenote-service

| Table | Main columns and constraints |
|---|---|
| `voice_note` | site, assignment (soft reference), check number, operator, kind (legacy 1–3), note name (unique per assignment), locale, audio file GUID, duration, size, transcription status, original and edited transcription (1,000), transcribed_at, export state, modified by/at. Partial index on pending statuses |
| `photo` | site, assignment, check number, operator, file name (unique per assignment), storage file GUID, MIME type, size, created_at (new: legacy photos had no row) |
| `transcription_attempt` | Note, provider, purpose (initial or correction), attempt number, timings, HTTP status, outcome, tokens used, provider request ID, error |

## vi-results-service

| Table | Main columns and constraints |
|---|---|
| `integration_endpoint` | Name, kind (REST XML, SOAP, REST JSON, webhook), URL, auth type, secret **reference** (Key Vault), timeout, maximum attempts, backoff `jsonb`, delivery mode (push/pull), write result facts, enabled, paused |
| `submission_delivery` | submission_id (PK), assignment, site, identifiers, endpoint, mode, status (waiting transcription, ready, in flight, delivered, failed, pull pending, pulled, cancelled, dead), pending voice-note IDs, payload `jsonb`, attempts, next_attempt_at, last error. Partial indexes for due and pull-pending rows |
| `delivery_attempt` | Submission, attempt number, timing, HTTP status, outcome, error excerpt |
| `result_fact` | Replaces `int_assignment_results` and both metadata tables: one flattened row per result, with assignment and check metadata stored once as `jsonb`; `consumer_state` keeps the legacy hand-off flag. Range-partitioned by `submitted_month`, purged by dropping partitions |
| `report_job` | Kind (assignment PDF), assignment, locale, status, output file, error, requested by/at, expiry |
| `inbound_host_request` | Idempotency key (unique), operation, site, assignment identifier, status, response |

## vi-admin-service

| Table | Main columns and constraints |
|---|---|
| `setting_definition` | key (PK), owner service, data type, default, validation, is_secret, scope (tenant/site), UI group and order, legacy key name |
| `setting_value` | (key, site_id) PK; a nil UUID means tenant-wide; secrets stored as references only |
| `notification` | site (null = system), source service, process, priority, message key and arguments, details `jsonb`, error code, dedup key (unique while unacknowledged), occurrence count, first/last occurrence, acknowledged by/at |
| `email_dispatch` | Notification, recipients, status, attempts, next attempt, last error |
| `job_definition`, `job_run` | Job registry (owner service, cron, enabled) and run history |
| `user_view_preference` | (user, view key) PK; grid columns, filters and time windows as `jsonb` |

## Where the legacy tables go

| Legacy tables | Target |
|---|---|
| `voc_user*`, `voc_role*`, `voc_feature*`, `voc_ldapconfiguration` | Keycloak and platform permissions; user properties → `user_view_preference` |
| `voc_site`, `voc_site_properties`, `voc_tag`, all `*_tag` tables | Platform site service, `site_ref` copies, `site_id` columns |
| `voc_jobs`, `voc_job_history` | `job_definition`, `job_run`, plus ShedLock in each service |
| `voc_notification`, `voc_notificationdetail` | `notification` (details as `jsonb`), `email_dispatch` |
| `voc_epp_license` | Platform licensing |
| `voc_system_properties` | `setting_definition` / `setting_value`; export settings → `integration_endpoint`; secrets → Key Vault |
| `voc_view`, `voc_column`, `voc_data_type`, `voc_user_column`, `voc_filter*`, `voc_timewindow`, `voc_homepage*`, `voc_summaries*` | Micro-frontend code and `user_view_preference` |
| `voc_charts*`, `voc_reports*`, `voc_reportType*`, `voc_system_translations`, `voc_data_translations`, `voc_plugin_*`, `voc_device*`, `voc_operators*`, `voc_networkcredentials` | Dropped (unused, or replaced by frontend i18n and the host shell) |
| `form_voice_forms`, `form_steps` + `form_args`, `form_conditions` + `form_step_conditions`, `form_list_item_groups`, `form_list_items`, `form_prompt_templates` | `voice_form`, `step` (params `jsonb`), `condition` + `step_condition`, `list_item_group`, `list_item`, `prompt_template` |
| `form_localizable_text`, `form_localizations`, `form_host_provided_text` | `localized_text`, `localized_text_translation` |
| `plan_plans`, `plan_sections`, `plan_steps`, `plan_step_prompt_data`, `form_section_condition*`, `form_plan_step_condition*` | `plan`, `plan_section`, `plan_step` (prompt data and condition IDs inline) |
| VoiceForms JSON files | `voice_form_version` → copied to `form_version` in execution |
| (new) | `plan_release` → copied to `work_package` in execution |
| `core_operators`, `core_break_types` | `operator` (+ Keycloak link), `break_type` |
| `core_assignments`, `core_assignment_metadata` | `assignment` (metadata `jsonb`, stored counters) |
| `core_assignment_checks`, `core_assignment_check_results`, `core_assignment_check_notes`, `core_prompt_data`, `core_assignment_check_metadata`, `check_section_conditions`, `check_section_condition_ids` | `assignment_check` (7 → 1; conditions resolved from the work package) |
| `core_assignment_submit` | `submission` + `submission_delivery` |
| `core_assign_num_gen` | `assignment_number_seq` |
| `core_voice_notes`, Photos folder | `voice_note`, `photo` + storage service |
| `int_assignment_results`, `int_assignment_metadata`, `int_assignment_check_metadata` | `result_fact` (partitioned) + lakehouse feed |
| templates folder, `vocab.json`, APK, ProblemLogs, VoiceAppResources/Images, checklists | `speech_template`, `vocabulary_config`, `mobile_app_release`, `device_problem_report`, `image_asset`, `checklist_import_job` |

## Related

- [[voicecheck-database-schema]] — the legacy schema.
- [[voicecheck-microservices-architecture]]
- [[voicecheck]]
