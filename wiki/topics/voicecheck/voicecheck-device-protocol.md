---
title: "VoiceCheck device protocol"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - legacy
  - devices
---

# VoiceCheck device protocol

Source: [[sources/voicecheck-codebase-analysis-2026-11-20]] (package `com.vocollect.voicecheck.voice.app`).

## Clients and transport

- **Clients:** the Talkman VoiceCheck task and the Android Inspection app (distributed by VoiceCheck) use the same API. On the Talkman, the operator speaks an access code and the work ID.
- **Transport:** HTTP(S) `POST` of JSON to `/VoiceCheck/services/voiceAppCommand`, served by CXF JAX-RS with Jackson. There is no socket server.
- **Request format:**
  ```json
  {"method": "<command>",
   "params": {"timestamp": "MM-dd-yy HH:mm:ss", "siteName": "", "device": "",
              "operator": "", "locale": "", "sessionId": "", "appVersion": ""}}
  ```
  `timestamp` is in the site's time zone; other parameters depend on the command.
- **Responses:**
  - Success: `{"result": {...}}`.
  - Error: `{"error": {"code": 10xx, "message": "..."}}`.
  - Business errors return HTTP 200, malformed requests 400, and other failures 500.
- **Execution:** each command is a prototype Spring bean and runs in one database transaction.

## Sessions and tokens

- **Sign-on:** `signOn` accepts one of three credentials and returns a `sessionId`:
  - an SSO access token (a JWT checked against the identity provider's keys);
  - the user's server password;
  - the operator's voice access code.
- **Session store:** the server keeps one session per operator in memory, with a sliding timeout of `Voice_Session_Timeout` hours (default 4).
- **One operator per device:** signing on to a device signs off any other operator on that device.
- **Per-assignment token:** from app version 1.7, `getWork` issues a token for the assignment. `postResult`, `createVoiceNote` and photo uploads must present it.

## Commands

| Command | Purpose (main input → output) | Calls per operator |
|---|---|---|
| `getOperators` | → operator IDs and names | per app launch |
| `getConfiguration` | app version → app name, all `Android_*` settings, server version, allowed and working site, vocabulary to train and thresholds | once per sign-on |
| `signOn` | credentials → session ID, break types, licence result, password-expiry warning days | once per shift |
| `getAssignmentTypes` (Android) | → available and assigned work IDs, and plans with localised names | per menu visit |
| `getAssignments` | work ID given as "assigned work", "create assignment", spoken trailing digits or a scanned value; review flag → matching work IDs or plans | 1–3 per assignment |
| `getWork` | work ID (plus a plan ID to create the assignment) → the assignment; the full form (sections, parts, loops, steps with localised prompts, parameters, conditions, prompt data, earlier results, list items); memos; the token | once per start or resume |
| `postResult` | assignment, token, check ID, status (1 incomplete, 2 not applicable, 3 complete), values, notes, section and iteration, time spent → empty result | once per step |
| `createVoiceNote` | note name, base64 WAV (about 30 s), memo flag → voice-note ID | per note or memo |
| `getVoiceNote` | work ID plus note number or name → audio and transcription | on playback |
| `getVoiceNotesForReview` | assignment → notes and memos with their original transcription | per review on the device |
| `postNoteUpdate` | note ID, corrected text | per edited note |
| `endWork` | assignment → releases the operator and the token; may auto-submit | per stop or finish |
| `sendBreakInfo` | break type → nothing is stored (placeholder) | per break |
| `signOff` | → releases in-progress assignments and the licence lease | once per shift |

## Side endpoints used by devices

| Endpoint | Purpose |
|---|---|
| `POST /services/photos` | Multipart photo upload for a photo step |
| `GET /services/imageService/{path}`, `GET /services/resources/{path}` | Step images and other resources (`~` stands for `/` in the image path) |
| `POST /services/templates/...` | Upload a trained speech template |
| `GET /services/templates/...` | Download an operator's templates as a zip for a new device |
| `POST /services/reportProblem` | Upload a zip of device logs |

## Behaviour worth knowing

- **Conditions are evaluated on the device.** Steps carry their conditions and earlier results, and skipped steps come back as "not applicable". The server only stores results and checks references.
- **Section loops:**
  - Iteration 1 is created together with the assignment.
  - A `postResult` for a higher iteration copies the iteration-1 checks.
  - Apps older than 1.7 cannot run looping plans.
- **Speech templates** are stored per operator, locale and word, in recognizer-specific formats, plus an `operator_settings.json` file:
  - `bt2` — Talkman;
  - `SRXAND_2` / `NSRXAND_2` — Android;
  - `SRXIOS_2` / `NSRXIOS_2` — iOS.
- **OpAcuity:** the settings `Android_Bool_useWFP` and `Android_Text_WFPURL` are only passed to the device. The device sends that telemetry itself.

## Related

- [[voicecheck-integrations-and-pipelines]]
- [[voicecheck-performance-and-scaling-constraints]]
- [[voicecheck]]
