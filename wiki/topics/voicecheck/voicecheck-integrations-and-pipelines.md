---
title: "VoiceCheck integrations and pipelines"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - legacy
  - integration
---

# VoiceCheck integrations and pipelines

Source: [[sources/voicecheck-codebase-analysis-2026-11-20]]. Items marked
*(unverified)* were read from configuration or code but not seen running.

## Assignment life cycle

| Code | Status | Entered when |
|---|---|---|
| 1 | Available | created (from the UI, a host or a device) or reopened |
| 2 | InProgress | a device starts it with `getWork` |
| 3 | InReview | a reviewer opens it for review (no operator may be on it) |
| 4 | Complete | closed out (UI, host status call or SOAP), or delivered from ExportPending |
| 5 | PendingReview | `endWork` at 100 % when auto-submit is off |
| 6 | ExportPending | 100 % and submitted; waiting for delivery |

**Creation paths**
- **UI:** a plan plus an optional operator. A blank assignment ID is generated.
- **Device:** `getWork` with a plan ID; the work ID comes from the device.
- **Host (REST or SOAP):** an explicit assignment ID, built either from a plan or from ad-hoc `sectionsToPerform`.

**Numbering:** to get the next number, a row is inserted into `core_assign_num_gen` and then deleted, inside a JVM-synchronised method.

**Checks are created up front:** when an assignment is created, one check is made per plan step, with its prompt data and conditions. The plan is copied by value; no foreign key links an assignment to its plan.

**Other transitions:**
- *Close out* sets Complete and creates no submission.
- *Reopen*, *cancel review* and *clear results* reset state.

## Submission and export

1. **Creating a submission.** A submission is created either by auto-submit at `endWork` (`ALLOW_AUTO_SUBMIT=true`) or by *Review and submit* in the UI.
   - It takes every check that is complete or not applicable and not yet exported.
   - It marks those checks as exported at that moment, so "exported" really means "attached to a submission".
2. **States:**
   - Status: NotSubmitted, Submitted, or WaitingTranscription (only when auto-submitting with untranscribed notes).
   - Result: NotAttempted, Success or Failed, plus a retry count.
3. **PUSH delivery** (`EXPORT_MODE=PUSH`):
   - A loader thread on the load balancer's primary node polls every second.
   - Two workers build the payload and post it to `SERVICE_ENDPOINT`, as XML over REST or as SOAP.
   - On success the submission is marked delivered and ExportPending assignments are closed out.
   - On failure the worker retries every 10 seconds, with no limit.
4. **PULL delivery:** the host calls
   - `GET /services/exports` — IDs waiting to be fetched;
   - `GET /services/exports/{id}` — the XML document;
   - `POST /services/exports/{id}` — acknowledgement.
5. **Interface tables** (`EXPORT_TO_INTERFACE=true`): flattened rows are also written to `int_assignment_results` and its two metadata tables, for hosts or BI tools to read.
6. **The `assignmentResults` payload:**
   - Top level: site, assignment ID, work ID, submitting user, time (UTC), metadata, memo results.
   - Section results contain step results: sequence counter, step ID, user, iteration, metadata, start and end, time spent, status, results, notes.
   - Photos are sent as URLs to the server's photo endpoint, list answers as item codes, and notes as transcription text.

*(unverified)* In `voicecheckJobs.xml` the Export Update Job trigger points at the transcription job. If so, the job that releases WaitingTranscription submissions never fires on schedule; check the Schedules screen.

## Host APIs (base `/VoiceCheck/services`)

REST calls authenticate with Basic authentication or an SSO bearer token, and need the `feature.voicecheck.execute.web.services` feature.

| Area | Endpoints |
|---|---|
| Assignments | `POST /assignments` (XML `<assignment>`); `POST /assignments/{site}/{id}/status` (accepts only `complete`, which closes out); `GET` and `DELETE /assignments/{site}/{id}` (delete only when Available or Complete at 0 %) |
| Exports (PULL) | `GET /exports`, `GET /exports/{id}`, `POST /exports/{id}` |
| Operators | list (zip); get, create, update and delete by name; `POST /operators/import` (zip); `GET /operator/{id}` (JSON plus a templates zip) |
| Authoring | `/voiceform`, `/voiceformVersion` (import also publishes), `/voiceplan`, `/steps`, `/translations` (CSV), `POST /voiceformBuilder` (OCR) |
| SOAP | `AssignmentService`: create, close out, read, delete; WS-Security UsernameToken |

**The `<assignment>` payload**
- Header fields: site name, assignment ID, work ID, metadata map, operator ID, form name and version.
- Then either a plan name or `sectionsToPerform`:
  - a section has an ID, description, part fields, loop flag, looping prompt and conditions;
  - each step has a sequence counter, step ID, mandatory flag, prompt data, metadata and conditions.
- Validation checks required fields, unique section IDs and sequence counters, and that the steps exist in the form version.

## Transcription

**Background processing**
- A loader thread polls new and edited notes and memos every second (100 at a time) into an in-memory queue.
- `TranscriptionJob` restarts the threads every 15 minutes.
- Audio longer than 35 s or shorter than 0.5 s is skipped.
- Notes edited by a reviewer are sent again together with their audio.

**Modes**

| Mode | Setting | How it works |
|---|---|---|
| Classic | `TRANSCRIPTION_MODE=true` | One thread per URL in `TRANSCRIPTION_SERVICE_URLS` posts XML containing base64 audio and the locale, with Basic authentication. With no URL configured, it falls back to a local Sphinx recogniser. |
| Next-Gen SaaS | `TRANSCRIPTION_MODE=false` | One thread posts JSON to `TRANSCRIPTION_TS_ENDPOINT` with a Keycloak bearer token. HTTP 402 (no tokens left) raises a critical notification. |

*(unverified)* The asynchronous path (HTTP 202 followed by a callback) does not work, because no callback URL is ever set. Notes answered this way stay pending until the server restarts.

## OCR: "VoiceForm Builder Agent"

- `OCR_ENDPOINT` provides `/estimate-tokens`, `/extract` and `/token-balance`, authenticated with the same Keycloak settings as transcription.
- Upload Inspection Checklist and `POST /voiceformBuilder` call it synchronously.
- The response contains VoiceForm JSON, which is saved as a new, unpublished form.

## VoiceForm versions

- **Publishing** writes `VoiceForms/<form>/<form>_<n>.json` (n = highest version + 1) from the live form and reloads an in-memory cache.
- **Deleting** a version is refused while any plan or assignment uses it.
- **Binding:** plans, sections, plan steps, assignments and checks store `voiceFormVer`.
  - Null means the live form in the database.
  - A number means that published file, with steps looked up by identifier.
- **JSON content:**
  - top level: `formVersion`, the form name, user translations;
  - steps: identifier, description, prompt type, prompt/help/display/reference texts with their localisations, args with list item groups, image, conditions, template, version;
  - conditions.
- **Not self-contained:** reading a version re-links its list item groups and prompt templates to the current database rows.
- **Clustering:** in clustered mode a thread polls `LAST_PUBLISHED_VOICE_FORM_VERSION_TIME` every second and reloads the files.
- *(unverified)* On a single node the cache may stay empty after a restart until something is published.

## Import and export formats

| Item | Format |
|---|---|
| VoiceForm | zip of JSON (legacy `.frm` XML is also accepted) |
| Form version | JSON |
| Plan | zip of JSON |
| Steps | `Steps.json` |
| Operators | zip |
| Translations | CSV |
| Voice notes | zip of XML plus WAV |

There is no Excel format.

## Reports

- The only report is the assignment PDF: `AssignmentReport.jrxml`, with sub-reports for sections, steps, results, photos, voice notes and memos.
- `AssignmentReportProcessor` fills it from Java beans.
- It is generated synchronously, and the templates are recompiled on every request.

## Scheduled jobs and purge

| Job | Schedule |
|---|---|
| `VoiceCheckPurgeArchiveJob` | daily at 02:00 |
| `TranscriptionJob` | every 15 minutes |
| `ExportUpdateJob` | every 15 minutes |

- The jobs run on Quartz in memory and are mirrored in `voc_jobs` for the Schedules screen.
- Purge rules are system properties named `<Model>_<Status>_Purge`; the defaults are listed in [[voicecheck-application-features]].
- Assignments are deleted in batches of 200, followed by their voice notes and audio files.
- Photos are deleted when no completed photo check references them.
- Interface-table rows are deleted by submitted date.

## Mobile app

- The APK upload checks the package and its version. An older version is staged until an administrator approves it.
- The file is stored as `apps/InspectionApp.apk`.
- Devices download it from an authenticated page or its QR code. `InspectionAppConfig.json` gives the device the server address and operator ID.

## Related

- [[voicecheck-device-protocol]]
- [[voicecheck-database-schema]]
- [[voicecheck-performance-and-scaling-constraints]]
- [[voicecheck]]
