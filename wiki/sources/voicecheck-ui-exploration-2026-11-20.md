---
title: "VoiceCheck UI exploration (2026-11-20)"
type: source
status: active
created: 2026-11-21
updated: 2026-11-21
sources: []
tags:
  - voicecheck
  - legacy
---

# VoiceCheck UI exploration (2026-11-20)

## Origin

The Copilot agent walked through a lab VoiceCheck server in the VS Code integrated
browser on 2026-11-20, after the user logged in as an administrator. The walk-through
was read-only. The About page reported Honeywell VoiceCheck V1.12 (build
202611210535) and Vocollect Integration Framework 2.0.

There is no file in `raw/`; the evidence is the pages listed below. The server
address and login details are deliberately not recorded.

## What was looked at

- **Inspection:**
  - Assignments (with the Memos grid), Operators, Break Types, Translations, VoiceNotes, Exports.
  - Mobile App Configuration (APK upload, training vocabulary).
- **VoiceForm Editor:**
  - VoiceForms and their steps, List Items, Prompt Templates, Conditions, Plans, Translations.
  - VoiceForm versions and Manage Images.
- **Administration:** Users, Roles, Logs, Schedules, Sites, Notifications, Licenses,
  System Configuration, About.
- **Forms opened but not submitted:** Create Operator/User, Create Assignment, Create
  VoiceForm, Create Step, Create Plan, Edit Training Vocabulary.
- **Detail views:** View Assignment, View Voice Plan, and the Assignment report
  (downloads as a PDF).

## Key observations

- **Create Step prompt types:** Prompt Only, Ready, Notes, Multiple List Selection,
  Float Value, Value Entry, Long Value Entry, Date, Part Number, Fraction, Photos.
- **Condition operators:** Equals, Not equals, Greater than, Less than, Greater than
  or equal, Less than or equal, Starts with, Ends with, Contains (any), Does not
  contain (any).
- **Translation languages:** en_US, de_DE, es_MX, ja_JP, fr_CA, fr_FR, nl_NL, zh_CN,
  it_IT, pl_PL, es_ES.
- **Schedules:** three jobs — Export Update Job (interval), Purge Archive Job (daily),
  VoiceNotes Transcription Job (interval).
- **Assignment report PDF:** a header (work ID, assignment number, status, start and
  end dates, reviewer, report date), then one table per plan section with step,
  description, operator ID, status, last update and time.
- **Sample data on the server:**
  - 10 assignments, all Available.
  - 3 operators and 1 site.
  - 2 forms: `automotive` with 31 steps, and `test form`.
  - 2 plans, with 30 and 8 steps.
  - 12 conditions.
  - 3 list item groups: `passfail`, `readyfail`, `taillight`.

## Limitations

- One lab server with sample data; no production data and no licence installed.
- Actions that change data were not exercised: save, submit, delete, publish,
  import, and start/stop exporting. Their effects are known only from the code (see
  [[sources/voicecheck-codebase-analysis-2026-11-20]]).
- The help pages were not read.
