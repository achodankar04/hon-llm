---
title: "Run the Voice Console MFE against local DM and OP services"
type: topic
status: active
created: 2026-11-17
updated: 2026-11-17
sources:
  - "[[sources/vcng-local-dev-setup-session-2026-11-17]]"
tags:
  - vcng
  - local-dev
  - voiceconsole
  - frontend
---

# Run the Voice Console MFE against local DM and OP services

Source: [[sources/vcng-local-dev-setup-session-2026-11-17]]. Status: DM override
verified in the served bundle; OP override is implemented the same way but **not
verified** (the local OP service was not started in the session).

## Facts

- Host app `pss_cf_frontend_core` runs on `:3000` and loads the Voice Console MFE
  (`pss_cf_mfe_voiceconsole`, `:3002`) via Module Federation; remote URLs come from
  `pss_cf_frontend_core/public/config/runtime-config.json`.
- The MFE builds service URLs as `<apiUrl>/devicemgmt` and `<apiUrl>/opmgmt`
  (`src/utils/api.ts`, `src/services/api/apisauceClient.ts`), where `apiUrl` comes
  from the host config (the dev API in the dev setup).
- Build-time overrides (added in the session, uncommitted):
  `webpack.config.js` injects `VCNG_LOCAL_DM_URL` and `VCNG_LOCAL_OP_URL` through
  `DefinePlugin`; when set, `/devicemgmt` and `/opmgmt` calls use them instead of the
  dev API. When unset, behaviour is unchanged.
- The values must be the full service base URL including context path:
  - DM: `http://localhost:8092/api/devicemgmt` (context path from DM `application.yml`)
  - OP: `http://localhost:8082/api/opmgmt` (port and context path from OP `application.yml`)
- The variables are read at webpack start, so restart the MFE dev server after
  changing them, then hard-reload the browser.
- The VS Code task "Start VoiceConsole MFE" does not set these variables; starting
  through the task goes back to the dev API.
- A browser-only alternative already exists: in the host page console run
  `localStorage.setItem("vcng_api_base_url_override", "http://localhost:8092/api")`
  (`src/utils/environments.ts`). It replaces the base URL for **all** services, so
  it only suits DM-only testing.

## Steps

1. Start the local DM: [[run-device-management-locally-with-dev-db]].
2. Start the local OP service (inferred, not verified): project
   `pss_cf_vcng_operatormanagement`, port `8082`, context path `/api/opmgmt`.
   Its `local` profile defaults to a DB on `localhost:5433`, so it needs its own
   DB URL override for the dev DB (probably the same host and `sslmode=require`
   with database `dev_ia_pss_fed_vcng_om`; unconfirmed), and the Kafka SASL
   placeholders set as for DM.
3. Start the host (task "Start Local Frontend Host", or
   `npm --prefix pss_cf_frontend_core start`) on `:3000`.
4. Stop any MFE already on `:3002` (`kill $(lsof -nP -tiTCP:3002 -sTCP:LISTEN)`), then start
   it with the overrides:

```bash
cd /Users/anant.chodankar/Documents/Workspace_VCNG7
VCNG_LOCAL_DM_URL=http://localhost:8092/api/devicemgmt \
VCNG_LOCAL_OP_URL=http://localhost:8082/api/opmgmt \
npm --prefix pss_cf_mfe_voiceconsole start
```

   Omit `VCNG_LOCAL_OP_URL` to keep operator calls on dev.
5. Hard-reload `http://localhost:3000`, log in (dev credentials), open the page. In
   the browser Network tab, requests should go to `localhost:8092/...` (and
   `localhost:8082/...` for operators).

## Known limits

- Downloading files from Azure Blob URLs returned by the storage service fails from
  `localhost` because the storage account CORS allows only
  `https://dev.psshub.honeywell.com` (observed in the session).
- `public/config/runtime-config.json` in the host was edited locally for testing
  (`host` and `voiceconsole` remotes, voiceconsole feature-flags URL); revert it
  before committing, as with the MFE override edits.
- Dependency installs: `~/.npm` may contain root-owned files here; use
  `npm ci --cache /tmp/npm-cache` if `npm install` fails with permission errors.

## Related

- [[run-device-management-locally-with-dev-db]]
