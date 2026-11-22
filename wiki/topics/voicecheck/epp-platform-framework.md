---
title: "EPP platform framework"
type: topic
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - legacy
  - epp
---

# EPP platform framework

Source: [[sources/voicecheck-codebase-analysis-2026-11-20]].

EPP (Enterprise Product Platform) is the Vocollect framework that VoiceCheck is built on. Class and plug-in names in EPP suggest VoiceLink and VoiceConsole use it as well *(inferred)*.

## Code layout

- Logic lives in `*Root` classes under `src/com/vocollect/epp/*`.
- `src-custom/` holds the thin subclasses plus all Spring, Struts and `hbm.xml` configuration.
- Main libraries: Spring 5.3, Spring Security 5.8 (with OAuth2 client, JOSE and LDAP), Hibernate 5.6, ehcache 2.10, c3p0, Struts 2.5 (SiteMesh, JSON, Spring, Jasper and DWR plug-ins), DWR 3, Quartz 1.5.2, JasperReports 6.3, CXF 3.5 with Swagger, Nimbus JOSE JWT, jasypt, JavaMail, jcaptcha, JFreeChart.

## Capabilities

| Capability | How it works | Used by VoiceCheck |
|---|---|---|
| Authentication | Form login against database users (BCrypt), LDAP bind, and OIDC single sign-on (authorisation code with PKCE, mapped to an existing user). A JWT validation helper serves non-browser clients. Repeated failures trigger a captcha and a temporary account lock. Password rules: at least 8 characters with mixed case, a digit and a special character; the last two passwords cannot be reused; expiry is set by `Password_Age`. Sessions time out after 240 minutes. | Heavily: UI login, REST Basic or bearer auth, device sign-on with a server password or SSO token |
| Authorisation | A role maps to features, and features belong to feature groups. `app-security*.xml` maps URLs to features, and a voter checks every action. "Read-Only" is just a seeded role whose features are all read-only. Templates hide what a user may not do. | Heavily: VoiceCheck adds 75 features and about 200 URL mappings |
| Users, operators, devices | `User` (roles, sites, tags, properties) is also the Spring Security principal. `OperatorCommon` and `DeviceCommon` are base models. | VoiceCheck's `Operator` extends `User` one-to-one (shared primary key); the base operator and device models are unused |
| Sites and tags | Each site has a tag. Site-scoped entities have tag join tables, and HQL is rewritten per request according to the current site context (see [[voicecheck-database-schema]]). | Heavily: assignments, voice notes, notifications and users |
| Scheduling | Quartz with an in-memory store and 5 threads, synchronised with the `voc_jobs` and `voc_job_history` tables. In clustered mode, nodes coordinate through database ping and poll columns. An option can restrict jobs to a "primary" node. | 3 jobs: purge, transcription, export update |
| Notifications | `Notification` rows with details and site tags, acknowledged in the UI. CRITICAL notifications are e-mailed to administrators of the site. The UI checks for unacknowledged notifications every 60 s. | Yes, through a wrapper that raises system notifications |
| Licensing | A signed XML licence (products, platforms, features with unit counts, expiry) is stored in the database. Leases are counted in memory per server. A daily job warns before expiry. | A lease is taken at device sign-on and released at sign-off |
| System and site properties | Name/value rows read through a static utility, with no cache. Site properties also exist. | Yes, through about 34 VoiceCheck properties (site properties are unused) |
| Localisation | Resource bundles per locale; `SystemTranslation` for enum codes (also used to sort by translated values); `DataTranslation` for user data, cached and refreshed every 60 s. | VoiceCheck ships bundles in 11 locales |
| UI table framework | `View` and `Column` metadata; per-user columns, filters and time windows; a data-provider bean called by reflection. The whole result set is loaded and then paged, sorted and filtered in memory. Grids refresh every 5 s by default (10 s in VoiceCheck). | Heavily: 15 views, 107 columns |
| Reports | `Report` and `ReportType` models plus a Jasper wrapper. The report scheduler is a stub. | Not used; VoiceCheck renders its PDF with the Struts Jasper plug-in |
| Logs | Lists server log files on the local node, lists device log folders, and downloads them as a zip. | Yes (admin screens) |
| Plug-in modules | `voc_plugin_modules` and `voc_plugin_components` rows build the top and side navigation. | VoiceCheck registers module 201 (`/core`, `/formeditor`) |
| Search | Named search queries for users, roles and sites, capped at 25 results per category. | Reused for the VoiceCheck search page |
| EAP and network credentials | Wi-Fi 802.1X settings for provisioning devices. | Not used |
| Clustering | No session replication and no distributed cache. Nodes coordinate only through the database: job ping/poll, a role-map change check every second, and translation refresh. Sticky sessions are required. | Same limits apply |

## Persistence patterns

- **Generic DAO.** DAO interface methods named `find*`, `list*`, `count*`, `update*` and so on run the named query `<Entity>_<method>` from the mappings.
- **Generic manager.** Covers get, getAll, save, delete and query-by-example.
- **Transactions.** Methods named `add*`, `save*`, `delete*`, `update*`, `create*`, `execute*`, `open*` and `close*` run read-write. Everything else runs read-only. OpenSessionInView keeps the session open for `*.action`, DWR and `/services/*` requests.
- **Saving.** `save()` flushes after every save.
- **Edit forms.** Edit forms keep detached entities in the HTTP session, so the version column can be checked on submit.

## How VoiceCheck depends on EPP

| EPP capability | Coupling | Likely replacement on Catalyst Fabric |
|---|---|---|
| Authentication | Heavy | Keycloak (LDAP federation, password policy, brute-force protection, SSO) |
| Authorisation (roles, features, URL map) | Heavy | Platform permission keys checked by each service |
| Users and roles | Heavy | Identity service; operators become a VoiceCheck-owned entity |
| Sites, tags, site context | Heavy | Platform site service plus a `site_id` column |
| Generic DAO, managers, transactions | Heavy | Spring Data JPA in each service |
| UI table framework, Struts, FreeMarker | Heavy | Micro-frontend grids over paged APIs |
| System properties | Heavy | Configuration service, ConfigMaps and secrets |
| Notifications and e-mail | Medium to heavy | Events plus an asynchronous notification service |
| Scheduling | Medium | Per-service jobs with distributed locks (ShedLock) or Kubernetes CronJobs |
| Licensing | Medium | Shared licensing service with a central lease store |
| Localisation | Medium | Frontend i18n plus a data-translation API |
| Logs, navigation, search, home pages | Light | Central logging; the host shell |
| EPP reports, EAP, site properties, charts | None | — |

**Version mismatch:** VoiceCheck 1.12 references EPP beans and classes that exist only on EPP feature branches, such as the user, site and role REST beans. Use the matching EPP branch when porting logic.

## Related

- [[voicecheck-codebase-overview]]
- [[voicecheck-database-schema]]
- [[voicecheck-microservices-architecture]]
- [[voicecheck]]
