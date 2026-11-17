---
title: "Run Device Management service locally against the dev database"
type: topic
status: active
created: 2026-11-17
updated: 2026-11-17
sources:
  - "[[sources/vcng-local-dev-setup-session-2026-11-17]]"
tags:
  - vcng
  - local-dev
  - device-management
---

# Run Device Management (DM) locally against the dev database

Source: [[sources/vcng-local-dev-setup-session-2026-11-17]]. Verified: DM started and
answered health checks on 2026-11-17.

## Facts

- Service: `pss_cf_vcng_devicemanagement`, port `8092`, context path
  `/api/devicemgmt` (`application.yml`).
- Use Spring profile `local`. Its default DB URL is `...:5433/...?ssl=false`, which
  does **not** connect to the dev DB; override it with the port `5432` +
  `sslmode=require` URL (`application-local.yml`).
- DB host/name/user come from `application-local.yml`
  (`PSS_CF_VOICE_DM_DB_URL`, `PSS_CF_VOICE_DM_DB_USERNAME`,
  `PSS_CF_VOICE_DM_DB_PASSWORD`). The yml default username and password worked
  against dev, so no password override is needed. Do not copy the password into
  the wiki; read it from that file.
- Kafka SASL settings (`PSS_CF_VOICE_KAFKA_VCNG_SASL_JAAS_CONFIG`,
  `PSS_CF_VOICE_KAFKA_CORE_SASL_JAAS_CONFIG`) have no default, so set a dummy value
  (Kafka is disabled in the `local` profile).
- Device logs endpoints need `DEVICE_LOG_MANAGEMENT_ENABLED=true` (default `false`).
- The storage service defaults to `http://localhost:8095/...` in the `local`
  profile; point it at dev with `PSS_CF_VOICE_DM_STORAGE_SERVICE_URL`.
- Auth is validated against the dev auth URL, so a token from the dev login works.

## Steps

1. Pick a JDK that exists on this machine (JDK 25 path below; check with
   `ls ~/Documents/Installed_Software/java/`).
2. Create a script (long pasted terminal commands get garbled, so use a file),
   e.g. `/tmp/run_dm_local.sh`:

```bash
#!/bin/bash
export JAVA_HOME=/Users/anant.chodankar/Documents/Installed_Software/java/jdk-25.0.1.jdk/Contents/Home
export PATH="$JAVA_HOME/bin:$PATH"
export DEVICE_LOG_MANAGEMENT_ENABLED=true
export PSS_CF_VOICE_DM_DB_URL='jdbc:postgresql://instance-365c5b5f-d92a-4621-8600-c58a18abf9c6.database.azuredatabricks.net:5432/dev_ia_pss_fed_vcng_dm?sslmode=require'
export PSS_CF_VOICE_DM_STORAGE_SERVICE_URL=https://dev.psshub.honeywell.com/api/storage
export PSS_CF_VOICE_KAFKA_VCNG_SASL_JAAS_CONFIG=unused
export PSS_CF_VOICE_KAFKA_CORE_SASL_JAAS_CONFIG=unused
cd /Users/anant.chodankar/Documents/Workspace_VCNG7/pss_cf_vcng_devicemanagement
exec mvn -q spring-boot:run -Dspring-boot.run.profiles=local
```

3. Run it in the background: `(bash /tmp/run_dm_local.sh > /tmp/dm-run.log 2>&1 &)`.
4. Wait about 30-60 seconds, then check:
   - `lsof -nP -iTCP:8092 -sTCP:LISTEN`
   - `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8092/api/devicemgmt/actuator/health` (expects `200`).
5. Stop it with `kill $(lsof -nP -tiTCP:8092 -sTCP:LISTEN)`.

## Troubleshooting

- "Unable to determine Dialect without JDBC metadata" plus "The connection attempt
  failed": wrong DB URL (the `ssl=false` / `5433` default). Use the URL above.
- Exit code 143 from `mvn`: the process was stopped with SIGTERM (for example the
  terminal was closed). Not a crash.
- Shell profile prints "No such file or directory" for a `JAVA_HOME` path on
  start-up: harmless here; the script sets its own `JAVA_HOME`.

## Related

- [[run-voiceconsole-mfe-with-local-dm-and-op]]
