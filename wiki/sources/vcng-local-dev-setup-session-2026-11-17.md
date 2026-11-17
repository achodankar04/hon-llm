---
title: "VCNG local dev setup session (2026-11-17)"
type: source
status: active
created: 2026-11-17
updated: 2026-11-17
sources: []
tags:
  - vcng
  - local-dev
---

# VCNG local dev setup session (2026-11-17)

## Origin

Hands-on Copilot session in the `Workspace_VCNG7` workspace. Commands were run and
their results observed on 2026-11-17 (macOS, JDK 25, Maven 3.9.11 via Homebrew).
Not a document in `raw/`; the evidence is the repo files listed below plus the
observed run results.

## Repo files the how-tos rely on

- `pss_cf_vcng_devicemanagement/src/main/resources/application.yml` — port `8092`,
  context path `/api/devicemgmt`, feature flag `DEVICE_LOG_MANAGEMENT_ENABLED`,
  default storage service URL.
- `pss_cf_vcng_devicemanagement/src/main/resources/application-local.yml` — `local`
  profile: DB defaults, Kafka SASL placeholders without defaults, auth URL.
- `pss_cf_vcng_operatormanagement/src/main/resources/application.yml` — port `8082`,
  context path `/api/opmgmt`.
- `pss_cf_vcng_operatormanagement/src/main/resources/application-local.yml` — `local`
  profile defaults (DB on `localhost:5433`, auth URL).
- `pss_cf_mfe_voiceconsole/webpack.config.js`, `src/utils/api.ts`,
  `src/services/api/apisauceClient.ts` — build-time local service URL overrides
  (added in this session, uncommitted).
- `pss_cf_mfe_voiceconsole/src/utils/environments.ts` — pre-existing browser
  `localStorage` override `vcng_api_base_url_override`.

## Observed results

- DM with the `local` profile and the dev DB URL (`sslmode=require`, port 5432)
  started: "Tomcat started on port 8092 ... context path '/api/devicemgmt'"; health
  check on `/api/devicemgmt/actuator/health` returned HTTP 200.
- DM with the yml default DB URL (`ssl=false`, port 5433) failed at startup with
  "The connection attempt failed" / "Unable to determine Dialect without JDBC metadata".
- Voiceconsole MFE restarted with `VCNG_LOCAL_DM_URL` set: the served `main` bundle
  contained the local URL.
- Not observed: a logged-in browser session calling the local DM; the local
  Operator Management service starting at all.

## Limitations

- The logged-in UI flow was not verified (login needs user credentials).
- Operator Management local run is inferred from its config files, not executed.
