---
title: "VoiceCheck codebase overview"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - legacy
  - codebase
---

# VoiceCheck codebase overview

Source: [[sources/voicecheck-codebase-analysis-2026-11-20]].

## Repositories

| Folder | Content |
|---|---|
| `maintenance_inspection_server/main` | The VoiceCheck web application (version 1.12) |
| `maintenance_inspection_server/VoiceCheckInstaller` | Windows installer, database operations (`db_operations.json`), database driver settings |
| `enterprise_product_platform/epp_main` | The EPP framework (see [[epp-platform-framework]]) |

- **Code layout:** business logic sits in `*Root` classes under `src/`. `src-custom/` holds thin subclasses plus all the Spring XML, Struts XML and Hibernate `hbm.xml` mappings. EPP uses the same pattern.
- **Size (approximate):**
  - VoiceCheck: about 67K lines of production Java in 547 files.
  - EPP: about 91K lines in 706 files.
  - Over 1,100 JavaScript files, plus FreeMarker pages.

## Stack

| Layer | Technology |
|---|---|
| Web UI | Struts 2.5.30 actions, FreeMarker with SiteMesh, jQuery and SlickGrid tables, DWR 3 (client-side validation only) |
| Services | Spring 5.3.27 with XML configuration; transactions assigned by method-name rules (`TransactionProxyFactoryBean`) |
| Persistence | Hibernate 5.6.5 with XML mappings only, c3p0 connection pool, no second-level cache, OpenSessionInView |
| APIs | Apache CXF 3.5.1 under `/services/*`: JAX-RS (REST with Jackson) and JAX-WS (SOAP) |
| Security | Spring Security 5.8 (see [[epp-platform-framework]]) |
| Jobs | Quartz 1.5.2 with an in-memory job store |
| Reports | JasperReports 6.3.1 |
| Runtime | Tomcat installed on Windows. SQL Server 2016–2022 or Oracle 19c in production; HSQLDB for tests |

## Packages (`com.vocollect.voicecheck.*`)

| Package | Role |
|---|---|
| `model`, `model/form` | Entities: forms, steps, plans, assignments, checks, voice notes and so on |
| `dao` | DAO interfaces that run named queries (`<Entity>_<method>`) |
| `service`, `service/impl` | Managers holding the business logic |
| `voice/app` | Device command beans (see [[voicecheck-device-protocol]]) |
| `web/action` | Struts actions for the UI |
| `web/rest`, `web/service`, `web/soap` | REST resources, shared web-service logic, SOAP endpoints |
| `reports` | Data for the assignment PDF report |
| `scheduling`, `threading` | Quartz jobs, worker threads for export and transcription |
| `licensing` | VoiceCheck licence lessor and licensee |
| `com.vocollect.transcription.*` | Transcription client threads and models |

## Main dependencies between managers

- **Device commands** use the operator, sign-off, assignment, assignment check, plan, voice form, host text, voice note, file depot, break type and list item managers, plus the vocabulary cache and system properties.
- **`AssignmentManager`** uses `PlanManager`, the submission services, the assignment-number DAO and notifications. `PlanManager` in turn uses the voice form, condition and host text managers.
- **`SubmissionServicesManager`** uses `AssignmentResultsExportManager` (which uses the result-interface and step managers) and the voice-note manager. Transcription is linked to export only through the state of voice notes.
- **`VoiceFormManager`** uses the step, list item group, condition, prompt template, host text and localisable text managers, and the file depot.
- **`VoiceFormVersionManager`** keeps a static cache of published versions. Checks, plan steps and validation read from that cache.
- **`OperatorManager`** uses the EPP user, role, tag and site managers, and the assignment manager.

## Files on disk

Everything sits under the folder set by the `FILE_BASE_DIR` system property.

| Path | Content |
|---|---|
| `VoiceForms/<form>/<form>_<n>.json` | Published form versions (not in the database) |
| `Transcription/VoiceNotes/<site>/` | Voice-note and memo audio (WAV) |
| `Photos/` | Photo results; the file name is stored as the check result |
| `VoiceAppResources/Images` | Step images |
| `templates/<operatorId>/<locale>/` | Speech-recognition templates |
| `vocab.json` | Training vocabulary and thresholds |
| `apps/InspectionApp.apk` | The Android app |
| `checklists/` | OCR checklist uploads, cleaned up after one day |
| `ProblemLogs/<device>/` | Device problem logs |

## Configuration

Runtime settings are name/value rows in `voc_system_properties`, edited in System Configuration. The main key groups:

| Area | Keys |
|---|---|
| Device | `Voice_Session_Timeout`, `Android_*`, `SSO_*`, `Company Name` |
| Assignment and export | `ALLOW_AUTO_SUBMIT`, `EXPORT_MODE`, `SERVICE_ENDPOINT`, `EXPORT_WEB_SERVICES_*`, `EXPORT_TO_INTERFACE` |
| Transcription and OCR | `TRANSCRIPTION_MODE`, `TRANSCRIPTION_SERVICE_URLS`, `TRANSCRIPTION_TS_ENDPOINT`, `TRANSCRIPTION_KEYCLOAK_*`, `OCR_ENDPOINT` |
| Versions and clustering | `LAST_PUBLISHED_VOICE_FORM_VERSION_TIME`, `APP_CLUSTERED`, `APP_LB_ENABLED`, `FILE_BASE_DIR` |
| Purge | `<Model>_<Status>_Purge`, for example `Assignment_Complete_Purge` |
| E-mail | `SMTP_*` |

- **Bootstrap files:** `database.properties` and `server.properties` (`server.name`, port, HTTPS, context root, `server.lb.primary.node`).
- **Unused keys:** the installer seeds `VOICE_CHECK_PORT` but the code never reads it, and `LAST_ASSIGNMENT_ID` is legacy.

## Licensing

- A device sign-on takes a lease for the operator and device from the "VoiceCheck" feature of the installed licence.
- Sign-off releases the lease; otherwise it lapses 12 hours after sign-on.
- Leases are held in memory on each server.
- The licence file itself is described in [[epp-platform-framework]].

## Related

- [[voicecheck-database-schema]]
- [[voicecheck-device-protocol]]
- [[voicecheck-integrations-and-pipelines]]
- [[voicecheck-performance-and-scaling-constraints]]
- [[voicecheck]]
