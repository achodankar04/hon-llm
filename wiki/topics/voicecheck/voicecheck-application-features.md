---
title: "VoiceCheck application features"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-ui-exploration-2026-11-20]]"
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - legacy
---

# VoiceCheck application features

This page describes the VoiceCheck 1.12 web UI as seen on a lab server
([[sources/voicecheck-ui-exploration-2026-11-20]]). For actions that were not tried
in the UI, the behaviour comes from the code
([[sources/voicecheck-codebase-analysis-2026-11-20]]) and is marked as such.

## How the pieces fit

The inspection life cycle runs in this order:
1. **VoiceForm:** a checklist of steps.
2. **Plan:** ordered sections made of those steps.
3. **Assignment:** a plan plus a work ID and an optional operator.
4. The operator completes the assignment by voice on a device.
5. **Pending Review**, then **Review and submit**.
6. **Export** to a host system.

**Navigation:**
- Two top-level areas: **Administration** and **VoiceCheck**.
- VoiceCheck has two tabs: **Inspection** and **VoiceForm Editor**.

**Grids:** every grid offers Manage Filter, Add or Remove Columns, Copy Selection and Printable Version. The actions in the left menu act on the selected rows.

## VoiceForm Editor (building checklists)

| Screen | What it holds | Actions |
|---|---|---|
| VoiceForms | A form is an ordered list of steps; a selector switches between forms | **Steps:** view, create, edit, delete, copy, import, export<br>**Form:** create, edit, delete, export, import, copy, manage versions, manage images, Upload Inspection Checklist |
| Step (create or edit) | Step ID, description, prompt, help message, optional prompt template, prompt type, parameters; tabs for Conditions, Display Options, Dependent Steps | Create, save and create another |
| List Items | List item groups: a code plus ordered items, for example `passfail` = passed, failed | View, create, edit, delete |
| Prompt Templates | Name, description, prompt type, number of steps that use it | View, create, edit, delete |
| Conditions | "Step X *operator* Constant *value*", tied to a form, with the number of steps that use it | View, create, edit, delete |
| Plans | Plan ID, description, VoiceForm and version, total steps, plan sections | Create, view, edit, delete, copy, export, import |
| Translations | Default spoken text, translation and type (step prompt, help message, …) for the chosen language | Add or delete host-provided text, import, export |
| VoiceForm versions | Published versions of one form, with created date | Publish new version, delete, import, export |
| Manage Images | Folder tree of step images | Create folder, upload (drag and drop), delete |

**Prompt types:** Prompt Only, Ready, Notes, Multiple List Selection, Float Value, Value Entry, Long Value Entry, Date, Part Number, Fraction, Photos.

**Step parameters** (the set depends on the prompt type): confirm, scan, OCR, priority prompt, minimum and maximum length, allowed characters (NUMERIC, ALPHA, ALPHA_NUMERIC), result data type (String, Integer), minimum and maximum value, allow negative, date format, decimal places, maximum selection.

**Condition operators:** Equals, Not equals, Greater than, Less than, Greater than or equal, Less than or equal, Starts with, Ends with, Contains (any), Does not contain (any).
- A condition switches a dependent step on.
- In the sample `automotive` form, a "failed" answer on a check such as *Wiper blades* brings up a Notes step that asks for the reason.

**Upload Inspection Checklist** takes a PDF or image and builds a new form through an external OCR service: estimate tokens, confirm, then extract. See [[voicecheck-integrations-and-pipelines]].

**Translation languages:** en_US, de_DE, es_MX, ja_JP, fr_CA, fr_FR, nl_NL, zh_CN, it_IT, pl_PL, es_ES.

## Inspection (running checklists)

**Assignments**
- **Columns:** Assignment ID, Work ID, Status, Operator Name, Start Time, End Time, Percent Complete, Pending Review, Time Spent.
- **Create form:** Assignment ID (or *System Generate*), Work ID, Plan, optional Operator ID.
- **Actions:** view, create, Manage assignments (close out, reopen, cancel review), review and submit, view voice plan, manage exports, assignment report.
- **View Assignment** shows the VoiceForm name and version, status, created date, operator, start and end time, percent complete, pending review and reviewer.
- **Voice plan (one row per step):**
  - Columns: section ID and description, part name and number, step sequence, step ID and description, results, last update, status, operator ID, results modified by and date, export status, time spent, notes.
  - Actions: clear results for selected steps, review and submit this voice plan, manage exports.
- **Assignment report:** downloads as a PDF. It has a header (work ID, assignment number, status, start and end dates, reviewer, report date) and one table per section (step, description, operator ID, status, last update, time).
- **Memos grid** (on the same page):
  - Columns: operator, operator name, transcription, created at, modified by and date, exported, transcribed date.
  - Actions: view, export all, delete.

**Operators**
- **One record for two roles:** "Create Operator/User" makes a single record that is both a UI user and a voice operator.
- **Create form:** Operator ID/User Name, Name, Password and confirmation, Force Change Password, Voice Access Code and confirmation, roles (Administrator, Read-Only), Email, Notes.
- **Columns:** Operator ID, Operator Name, Status (Signed On or Signed Off), Sign On Time, Sign Off Time, Working Site.
- **Actions:** view, create, edit, delete, sign off selected operators.

**Break Types** hold a description and a number. Actions: view, create, edit, delete.

**VoiceNotes** use the same columns as Memos. Actions: view, export all VoiceNotes from the system.

**Exports**
- **Columns:** Assignment ID, Status, Exported Date, Result, Exported By, Retries.
- **Actions:** delete selected export, start exporting, stop exporting, skip transcription for the selected export.

**Mobile App Configuration**
- **App upload:** upload the Android app (`.apk`). Devices download it from a page on the server or by scanning its QR code. Downloading needs administrator or download-mobile-app permission.
- **Vocabulary:** choose which words, taken from list items, devices should train, and set a recognition threshold per language.

## Administration

| Screen | Content |
|---|---|
| Users | User name, name, roles, e-mail, last login time, notes, status |
| Roles | Administrator and Read-Only (seeded) |
| Logs | Server logs and device logs; save as zip |
| Schedules | Export Update Job (interval), Purge Archive Job (daily), VoiceNotes Transcription Job (interval); run selected job; schedule history |
| Sites | Name, description, notes, time zone (the lab server has one Default site) |
| Notifications | Application, process, priority, message, date/time, acknowledged by and when, site; acknowledge |
| Licenses | View or import a licence, view the agreement (the lab server is unlicensed) |
| System Configuration | See below |
| About | Version and build, plus "Gather System Information" |

**System Configuration groups:**
- SMTP.
- Authenticate users against a directory server.
- Post Assignment Results web service: service endpoint, client type REST, export to interface, authentication.
- Transcription server choice (the classic Transcription Server or the Next-Gen SaaS server), with its endpoints.
- OCR endpoint.
- Android settings: OpAcuity URL, use OpAcuity, allow multiple assignment.
- Export settings: allow auto export, export mode PUSH. Changes need a server restart.
- Password expiration: maximum age, notification days.
- Purging and archiving.

**Default transactional purge periods** (archive purge was disabled on the lab server):

| Data | Days |
|---|---|
| Exported memos / all memos | 45 / 90 |
| Reporting data tables | 90 |
| All photos | 90 |
| Exported voice notes / all voice notes | 45 / 90 |
| Complete assignments / all assignments | 7 / 90 |
| All device logs | 7 |
| Job histories | 7 |
| Notifications | 7 |

## Related

- [[voicecheck-device-protocol]] — what the device does during an assignment.
- [[voicecheck-integrations-and-pipelines]] — status transitions, export and transcription behind these screens.
- [[voicecheck]] — topic hub.
