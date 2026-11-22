---
title: "VoiceCheck codebase analysis (2026-11-20)"
type: source
status: active
created: 2026-11-21
updated: 2026-11-21
sources: []
tags:
  - voicecheck
  - legacy
  - codebase
---

# VoiceCheck codebase analysis (2026-11-20)

## Origin

On 2026-11-20 the Copilot agent and research sub-agents read two local repositories
in the `Workspace_VoiceCheck` workspace, using file searches and file reads. Nothing
was built or run. They also read the Catalyst Fabric services in `Workspace_VCNG7`
for target-platform conventions.

| Repository folder | What it is | Commit read |
|---|---|---|
| `maintenance_inspection_server/main` | VoiceCheck server, version 1.12 | 280db743 |
| `maintenance_inspection_server/VoiceCheckInstaller` | Installer and database operations | same repository |
| `enterprise_product_platform/epp_main` | The EPP framework VoiceCheck is built on | e4d3e7ca |

## Main evidence files

- **Hibernate mappings:** `src-custom/com/vocollect/{epp,voicecheck}/**/*.hbm.xml`
  (39 in EPP, 28 in VoiceCheck). The session factory is in
  `applicationContext-epp-hibernate.xml`.
- **Spring and Struts configuration:** `src-custom/applicationContext-voicecheck-*.xml`,
  `applicationContext-epp-*.xml`, `struts.xml`, `app-security-voicecheck.xml`.
- **Scheduled jobs:** `voicecheckJobs.xml`, `quartz.properties`.
- **Database upgrade scripts:**
  `installer/externalResources/Database/VoiceCheck/<Vendor>/Upgrade/<version>/Vocollect_script.sql`,
  covering SQL Server 2008–2022 and Oracle 10g–19c.
- **Seed data:** `data/install/install-data-voicecheck.xml` and the EPP `data/dbunit/*.xml` files.
- **Device protocol:** package `com.vocollect.voicecheck.voice.app`, namely
  `VoiceAppRequestServiceImplRoot` plus one command bean per method.
- **REST and SOAP:** packages `com.vocollect.voicecheck.web.rest`, `web.service` and
  `web.soap`, configured in `applicationContext-voicecheck-restServices.xml`.
- **Reports:** `web/jasper/*.jrxml`.

## Limitations

- **EPP version mismatch.** The EPP checkout is older than the EPP version VoiceCheck
  1.12 builds against. Some beans and methods VoiceCheck references exist only on EPP
  feature branches:
  - the user, site and role REST beans;
  - `isClusteredOrLBEnabled`;
  - the transcription property getters.
- **No generated create scripts.** They are not in the repositories. Table
  definitions come from the mappings plus the upgrade scripts, and column sizes on
  upgraded databases can differ.
- **Inferred behaviour.** Items marked *(unverified)* in the topic pages were inferred
  from code and not confirmed by running the application.
- **Approximate counts.** Line and file counts are approximate.
