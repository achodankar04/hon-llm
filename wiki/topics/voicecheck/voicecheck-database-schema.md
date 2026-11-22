---
title: "VoiceCheck database schema"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - legacy
  - database
---

# VoiceCheck database schema

Source: [[sources/voicecheck-codebase-analysis-2026-11-20]] (Hibernate mappings and upgrade scripts).

## How the schema is managed

- **Definition.** The schema is defined only by Hibernate `hbm.xml` mappings: 39 in EPP and 28 in VoiceCheck.
  - The create DDL is generated at build time by a Hibernate schema-export task and is not in the repositories.
  - Upgrades are hand-written SQL per database vendor and version.
- **Vendors.** SQL Server 2016–2022 and Oracle 19c, with upgrade folders for older versions. HSQLDB is used for tests.
- **IDs.** The `native` generator gives an identity column per table on SQL Server, and one shared sequence (`voc_primary_key_sequence`) on Oracle.
  - Seed rows use negative IDs.
  - `core_operators` shares its primary key with `voc_user`.
- **Prefixes and naming.**
  - `voc_` — EPP
  - `core_` — runtime data
  - `form_` — design and localisation
  - `plan_` — plans
  - `int_` — interface and reporting
  - `check_` — check-to-condition joins

  EPP's older tables use snake_case column names; VoiceCheck uses camelCase.
- **Columns and conventions.**
  - Most entities have a `version` column for optimistic locking, and most VoiceCheck tables have `createdDate`.
  - Checks and voice notes also have `modifiedByUserId` and `modifiedDate`.
  - There are no generic audit columns, no soft delete, and no views, procedures or triggers.
- **Caching and storage.**
  - There is no Hibernate second-level cache.
  - Published form versions, audio, photos and images are files on disk, not LOBs. The only LOB is the licence.
- **Isolation.** On SQL Server, read-only service methods and device commands run with READ UNCOMMITTED isolation.

## Multi-site model

- **Site tags.** Each `voc_site` row has one `voc_tag` row (`tag_type=1`, `taggable_id` = site ID). Two special tags exist: -463 SYSTEM and -927 ALL.
- **Join tables.** Each site-scoped ("taggable") entity has its own join table with columns `tagged_id` and `tag_id`:
  - `core_assignment_tag`
  - `core_voice_note_tag`
  - `voc_notification_tag`
  - `voc_report_tag`
  - `voc_user_tag`

  Operators get their sites from their user's tags.
- **Writing.** Saving an entity attaches the tag of the current site.
- **Reading.** Reads are filtered by rewriting HQL at runtime (`QueryUtil.addQuerySiteRestriction`) to join the tag table. There are no Hibernate filters.
- **Global data.** These are not site-scoped: all `form_*` and `plan_*` tables, break types, templates and system properties. Child rows are scoped through their assignment.

## Table catalogue (88 tables)

### EPP: security and users

| Table | Purpose and main columns |
|---|---|
| `voc_user` | User: name (unique), fullName, password plus two history columns, email, last_login, login_location, notes, enabled, allSitesAccess, currentSite, failed-login counters, changePwd, lastPasswordChangeDate |
| `voc_user_to_role`, `voc_user_to_site`, `voc_user_tag` | User ↔ role, user ↔ site, user ↔ site tag |
| `voc_user_properties` | Name/value pairs per user |
| `voc_role`, `voc_role_to_feature` | Role (name, description, is_admin; seeded with Administrator and Read-Only) and its features |
| `voc_feature`, `voc_feature_group` | About 100 features (name, read_only, group). VoiceCheck adds 75 features and 2 groups |
| `voc_ldapconfiguration`, `voc_networkcredentials` | Directory settings and network credential settings (the latter unused by VoiceCheck) |

### EPP: sites, jobs, notifications, licence, properties

| Table | Purpose and main columns |
|---|---|
| `voc_site`, `voc_site_properties`, `voc_tag` | Site (name, description, notes, timeZone, shiftStartTime, Wi-Fi fields), per-site properties, tags |
| `voc_jobs`, `voc_job_history` | Job (name, type interval/daily, class, status, interval, cron, last_run, running, last_ping) and run history (start, finish, result, details) |
| `voc_notification`, `voc_notificationdetail`, `voc_notification_tag` | Notification (host, application, process, errorNumber, message, priority, acknowledged user and time, created time) with key/value details |
| `voc_epp_license` | The imported licence (the only LOB) |
| `voc_system_properties` | Name/value configuration (no unique constraint on name) |

### EPP: UI framework, reports, localisation, plug-ins, unused

| Table | Purpose |
|---|---|
| `voc_view`, `voc_column`, `voc_data_type` | Grid definitions (VoiceCheck has 15 views and 107 columns) |
| `voc_user_column`, `voc_filter`, `voc_filter_criterion`, `voc_timewindow` | Per-user grid columns, saved filters, time windows |
| `voc_homepage`, `voc_homepage_summaries`, `voc_homepage_access_summaries`, `voc_summaries`, `voc_summaries_extra_params`, `voc_charts`, `voc_chart_settings` | Home-page summaries and charts (VoiceCheck defines none) |
| `voc_reports`, `voc_report_parameter`, `voc_report_tag`, `voc_reportType`, `voc_reportType_parameter` | EPP scheduled reports (unused by VoiceCheck) |
| `voc_system_translations`, `voc_data_translations` | Enum and system translations; translations of user data |
| `voc_plugin_modules`, `voc_plugin_modules_auth`, `voc_plugin_components`, `voc_plugin_components_auth` | Navigation modules. VoiceCheck is module 201; its `moduleVersion` records the schema version |
| `voc_device`, `voc_device_tag`, `voc_operators`, `voc_operator_tag` | VoiceLink/VoiceConsole base tables, unused by VoiceCheck |

### VoiceCheck: form design

| Table | Main columns |
|---|---|
| `form_voice_forms` | name, createdDate. Only the live form is stored here; versions are files |
| `form_steps` | voiceFormId, stepIdentifier, description, prompttype, the IDs of the prompt/help/display/refLinkLabel texts, promptTemplateId, imageURL, stepVersion, displayIdentifier, refLinkURL |
| `form_args` | Name/value parameters of a step or a template, optional listItemGroupId |
| `form_conditions`, `form_step_conditions` | Condition (jsonString, voiceFormId, description, leftStep, rightStep) and step ↔ condition |
| `form_list_item_groups`, `form_list_items` | Group (groupCode unique, description); item (itemCode, itemOrder, group, itemTextId) |
| `form_prompt_templates` | name (unique), description, promptType |
| `form_localizable_text`, `form_localizations`, `form_host_provided_text` | Default text; text per language; host-provided texts |

### VoiceCheck: plans

| Table | Main columns |
|---|---|
| `plan_plans` | voiceFormId, voiceFormVer, identifier (unique), description |
| `plan_sections` | planId, identifier, description, sectionOrder, voiceFormVer, loop, loopingPrompt |
| `plan_steps` | planSectionId, stepId (or verStepIdentifier when bound to a version), sequenceCounter, stepOrder, mandatory |
| `plan_step_prompt_data` | planStepId, promptKey, value |
| `form_section_conditions`, `form_section_condition_ids`, `form_plan_step_conditions`, `form_plan_step_condition_ids` | Conditions on sections and plan steps, linked either by FK or by an ID inside a version file |

### VoiceCheck: execution

| Table | Main columns |
|---|---|
| `core_operators` | ID equal to `voc_user.id`; status (signed on/off), voice access code, signOnTime, signOffTime, device, workingSiteName |
| `core_assignments` | assignmentIdentifier (unique), workId (unique), speakableWorkId, voiceFormId and voiceFormVer, operatorId, status (1–6), startTime, endTime, reviewUserId, planIdentifier and planDescription (plan copied by value), token. Computed fields: totalChecks, completeChecks, notSubmitted, timeSpent |
| `core_assignment_tag`, `core_assignment_metadata` | Site tag; metadata key/value |
| `core_assignment_checks` | One row per plan step per loop iteration: assignmentId, stepId or verStepIdentifier, sequenceCounter, mandatory, sectionIteration, categoryId and categoryDescription (the section), part fields, loopingPrompt, submissionId, checkOrder, status (1 incomplete, 2 not applicable, 3 complete), operatorId, start and end, exportStatus, modifiedBy and modifiedDate, timeSpent |
| `core_assignment_check_results`, `core_assignment_check_notes` | Result values per check (codes, values, photo or audio file names) and dynamic notes |
| `core_prompt_data`, `core_assignment_check_metadata` | Prompt data and metadata key/value per check |
| `check_section_conditions`, `check_section_condition_ids` | Conditions copied onto checks |
| `core_assignment_submit` | Submission: assignmentId, status, userId, submissionResult, retryAttempt |
| `core_voice_notes`, `core_voice_note_tag` | Voice notes and memos: assignmentId (no FK), audioFileName (unique), operatorId, locale, original and edited transcription (1,000 characters each), transcription status, export status, recordingType (1 note, 2 memo, 3 memo reviewed) |
| `core_assign_num_gen` | Assignment number generator (normally empty) |
| `core_break_types` | breakNumber, ID of the name text |

### VoiceCheck: interface (reporting) tables

| Table | Main columns |
|---|---|
| `int_assignment_results` | One flattened row per result, note or voice note, per check, per submission: site, assignment, work ID, plan, form, section, step, timings, status, result and note codes and values, submitted by, updated by, resultState (-1 until a consumer takes the row) |
| `int_assignment_metadata`, `int_assignment_check_metadata` | Assignment and check metadata, copied once for every result row |

## Relationships

- **Form aggregate:**
  - A form owns its steps and conditions (cascade delete).
  - A step has args, localisable texts, and an optional template and list item group.
  - Conditions and steps are linked many-to-many.
- **Plan aggregate:**
  - A plan points to a form (plus version) and owns sections → plan steps → prompt data (cascade delete).
  - Plan steps refer to form steps by ID, or by identifier when the plan is bound to a version.
- **Assignment aggregate (site-scoped):**
  - An assignment points to a form (plus version), an optional operator and an optional reviewer.
  - It owns checks and metadata (cascade delete). Checks own results, notes, prompt data, metadata and conditions.
  - It also has submissions.
  - Checks point to **form** steps, not plan steps.
  - Voice notes are linked to assignments without an FK, and checks reference notes by file name.
- **Cross-domain links a split must break:**
  - operator, reviewer and modifier columns → `voc_user`;
  - every `*_tag` table → `voc_tag`;
  - assignment → form;
  - check → step and condition;
  - break type → localisable text;
  - plan → form.

## Growth and query hot spots

- **Fastest-growing tables:**
  - `core_assignment_checks` (assignments × steps × iterations) and its child tables;
  - `int_assignment_results` with its metadata tables, which grow multiplicatively;
  - then assignments, submissions, voice notes, notifications and job history.
- **Widest tables:** `int_assignment_results` (28 columns) and `core_assignment_checks` (about 28).
- **Assignment formulas:** four correlated sub-queries run for every assignment row loaded, and the device "available work" query filters on them.
- **Spoken work-ID lookup:** it uses `like '%digits'`, which no index can serve.
- **Missing indexes:**
  - assignments on status, operatorId and createdDate;
  - checks on submissionId and operatorId;
  - submissions on status and assignmentId;
  - voice notes on createdDate and exportStatus;
  - `int_assignment_results.resultSubmittedDate`;
  - every `*_tag` table on `tag_id`;
  - notifications.
- **Purge queries** filter on unindexed dates and statuses, and use anti-joins over all assignments.

## Schema drift to check on real databases

- The SQL Server 1.7 upgrade script created `verStepIdentifier` as `nvarchar(1)`.
- `voc_system_properties.value` is 4,000 characters on SQL Server from 1.12, but 2,000 in the mapping.
- `sequenceCounter` is 255 characters on upgraded databases, but 188 in the mapping.
- Column types are mixed depending on the version that added them: `numeric(19,0)` or `bigint`, `datetime` or `datetime2`, `tinyint` or `bit`.
- `voc_user.name` is case-sensitive only after the SQL Server 2017/2019/2022 1.12 scripts.
- There is no Oracle 1.12 upgrade script.
- *(unverified)* The `IX_orig_trans` index covers a 1,000-character column, which exceeds SQL Server's index key size limit, so saving long transcriptions may fail.

## Related

- [[voicecheck-target-data-model]]
- [[epp-platform-framework]]
- [[voicecheck-performance-and-scaling-constraints]]
- [[voicecheck]]
