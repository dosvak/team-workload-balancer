# Ready-made utility apps on the kit: recipes, REST facts and pitfalls

Fourteen small process applications built with `twxkit.py` + `kitlib.py` (standard library only, no base export), each a single
client-side human service dashboard with server-side service flows that call the product REST API as a technical user. Every one
is served as a package (`get_package` prints the download URL), as a generator (`get_tool('build_<acr>.py')`) and as a design topic.
Rebuild with your own snapshot name or environment defaults, or copy the closest one for a new tool: the pattern is always
`kl.new_app(...)` -> business objects -> flows (`app.flow(name, inputs=[('data', BO)], outputs=[('results', BO)], script=JS)`) ->
`kl.csv_flow` -> `app.cshs(name, layout, variables, init)` with `kl.frame(L, title, tabs, extra)`.

| App | Acronym | Target | Data source |
|---|---|---|---|
| SLA Deadline Monitor | SLAMON | 8.6.2, BAW 24-26 | PUT /search/query byTask, PUT /task/{id}?action=assign |
| Team Workload Balancer | TWLB | 8.6.2, BAW 24-26 | PUT /search/query byTask, PUT /task/{id}?action=assign (toUser, toGroup, back) |
| Team Audit | PTAUD | 8.6.2, BAW 24-26 | GET /processApps, /globalTeams, /team/{id}?snapshotId=, /exposed, /user/{login}?includeInternalMemberships=true |
| Process Smoke Test Runner | PSTR | 8.6.2, BAW 24-26 | GET /exposed, POST /process?action=start, GET /process/{id}?parts=header, PUT /process/bulk |
| Instance Timeline | ITLINE | 8.6.2, BAW 24-26 | PUT /search/query byInstance -> server-rendered SVG |
| Orphan Zombie Cleaner | OZCLN | 8.6.2, BAW 24-26 | PUT /search/query byInstance / byTask, GET /processApps, PUT /process/bulk |
| Environment Variable Diff | ENVDIF | BAW 20.0.0.1+ (reports 8.6.2.20001), 24-26, CP4BA | /ops containers, versions, env_vars (CSRF login) |
| Deployment Runbook Generator | DRUNBK | BAW 20.0.0.1+, 24-26, CP4BA | /ops versions, env_vars, team_bindings + GET /exposed |
| Business Data Search | BDSRCH | 8.6.2, BAW 24-26 | GET /searches/tasks/meta/businessDataFields, PUT /search/query with alias columns |
| Notification Manager | NOTIFY | 8.6.2, BAW 24-26 | saved searches as subscriptions (POST / PUT / DELETE /searches/tasks), PUT /search/query, javax.mail |
| Process Performance | PPERF | 8.6.2, BAW 24-26 | PUT /search/query byTask (closed tasks) / byInstance -> durations, throughput, variants |
| Role Inbox | RLINBX | 8.6.2, BAW 24-26 | GET /user/{login} memberships, PUT /search/query, PUT /task/{id}?action=assign (toUser, back) |
| Event Manager Monitor | EVMON | BAW 20.0.0.1+, 24-26, CP4BA | /ops event_manager_tasks, pause / resume, replay, monitor (24+); SQL LSW_UCA |
| Snapshot Lifecycle | SNAPLC | BAW 20.0.0.1+, 24-26, CP4BA | SQL LSW_PROJECT_DEPENDENCY + LSW_PO_VERSIONS; /ops versions, processes/count |

Extended in the second round: SLA Deadline Monitor 1.1 (aging report per team, at-risk watch list) and Environment Variable Diff 1.1
(export of every variable of every snapshot). Workstation tools of the kit (twx_diff, twx_sensitive_scan, baw_healthcheck,
snapshot_promote, coach_a11y, rest_recorder): get_topic('howto-kit-tools').

## The shared pieces (kitlib.py)

* `kit-common.js` server file: `kitSearch(org, columns, conditions, size, sort)` (PUT /search/query, throws on a non-200 with the
  body), `kitIso(text)` (ISO 8601 -> ms, the engine's Rhino has no reliable `Date.parse` of ISO text), `kitDate(ms)`, `kitHours(ms)`,
  `kitCsv(rows, cols)`, `kitEsc`, `kitEnc`, `kitHeader(title, user, subtitle)`, `kitOps(method, path, body)` (CSRF login once per
  script run through `opsLoginPath`, then `BPMCSRFToken` on every `/ops` call under `opsBasePath`).
* `new_app(name, acronym, snapshot, description, prefix, subtitle)`: kit-rest.js + kit-common.js + `appTitle` + the `<prefix> Init`
  flow (header HTML, user name). `KITENV_<name>` in the build environment overrides any environment variable default (twxkit `App.env`,
  so every kit generator honours it) - lab builds with real hosts and users under their own snapshot name.
* `csv_flow(app, prefix, rows_bo, csv_bo, cols)`, `frame(L, title, tabs, extra)` (header, status line, tab section, CSV modal,
  message modal), `ON_RESULT` / `ON_CSV` event expressions, `STD_VARS`, `ops_env(app)`.
* `kitSql(sql, maxRows)` (kit-common.js): read-only SELECT on the product database through the engine's JNDI datasource
  (`dbJndiName`, default `jdbc/TeamWorksDB`; `kl.sql_env(app)`), for facts no REST API exposes - verified on 20.0.0.1 (Db2) and 26.
* `kit_flow_test.py <app>.ids.json "<flow>" '{"data": {...}}'` runs any flow over REST before the dashboard is played
  (`POST /service/{id}?action=start&branchId=&params=` - the JSON is keyed by the input parameter name `data`).

## Search facts (PUT /search/query, verified on 8.6.2)

* Task columns that work: taskId, taskSubject, taskStatus, taskPriority, taskDueDate, taskReceivedDate, taskClosedDate, taskIsAtRisk,
  taskAtRiskTime, taskActivityName, assignedToUser, assignedToRole, assignedToRoleDisplayName, taskReceivedFrom, instanceId, instanceName,
  bpdName, bpdId, instanceStatus, instanceCreateDate, instanceModifyDate, instanceDueDate, instanceProcessApp, instanceSnapshot,
  instanceSnapshotId. **Not** `taskAssignedTo` (CWTBG0037E) although the rows contain it as `{type, who}`.
* `taskStatus|Equals|New_or_Received` = open tasks; `instanceStatus|Equals|Active|Completed|Failed|Terminated|Suspended`;
  `instanceProcessApp|Equals|<acronym>`; repeated `condition` = AND; `sort` ascending only; `size` <= 500.
* `organization=byInstance` still returns one row per task: de-duplicate by `instanceId`. `instanceSnapshotId` comes without the
  `2064.` prefix that `GET /processApps` uses.
* `GET /search/meta/task` / `/instance` answer CWTBG0039E on 8.6.2: the column list is documented, not discoverable.
* Dates are ISO 8601 UTC (`2026-09-04T13:11:32Z`); team tasks have `assignedToUser: null` and the team in `assignedToRoleDisplayName`.

## Team and user facts (verified on 8.6.2)

* `GET /globalTeams` -> teamId, teamName, processAppName (no snapshot). `GET /team/{teamId}` and `GET /team?name=` need `snapshotId`
  (CWTBG0011E) -> name, type (`StandardMembers` | service), members [{name, type User | Group}].
* `GET /user/{login}?includeInternalMemberships=true` -> memberships: plain groups plus `<Team>_T_<team uuid>.<snapshot uuid>` (team
  members) and `<Team>_S_...` (team managers).
* `GET /exposed` lists what the calling user may open, without the exposing team.
* `PUT /task/{id}?action=assign&toUser=<login>` | `&toGroup=<group>` | `&back=true` | `&toMe=true`; an unknown login answers 404
  `AssignActionFetchErrorException`.
* User preferences (`PUT /user/{login}?action=setPreference`) accept only the product's own keys (CWTBG0541E) - not a place to
  persist application data. Shared saved search definitions are (`POST /searches/tasks`: name up to 64 characters, `fields` as plain
  column names, `shared: true` visible to everyone, owner = the caller; `PUT` renames, `DELETE` removes, `GET /tasks?searchId=` runs).

## Instance facts (verified on 8.6.2)

* `POST /process?action=start&bpdId=&branchId=[&snapshotId=]&params=<json>&parts=header` -> `data.piid`, `executionState`; the
  `startURL` of the `GET /exposed` item already carries bpdId + branchId.
* `GET /process/{id}?parts=header` -> executionState (Active, Completed, Failed, Terminated, Suspended), creationTime,
  lastModificationTime, dueDate, snapshotID, processAppAcronym.
* `PUT /process/bulk?instanceIds=1,2,3&action=terminate|delete|suspend|resume|retry` (delete needs terminated / finished instances).
* `java.lang.Thread.sleep(ms)` works in a server-side script (polling loops); keep the total under the HTTP timeout of the caller.

## /ops facts (verified on BAW 26 traditional and on the 20.0.0.1 lab that reports 8.6.2.20001 - the original 8.6.2 has no /ops)

* Login: `POST /bpm/system/login` (basic auth, body `{"refresh_groups": true, "requested_lifetime": 7200}`) -> `csrf_token`; every
  `/ops/std/bpm` call carries `BPMCSRFToken: <token>`, otherwise CWTBG0651E. CP4BA: `/bas/bpm/system/login`, `/bas/ops/std/bpm`.
* `GET containers?type=PA` -> containers [{container, container_name, id, archived, toolkit}]; `GET containers/{acr}/versions` ->
  versions [{version, id, active, tip, branch_name, creation_date, target_environment, installable}]; `GET containers/{acr}/versions/{v}`
  -> the same facts; `optional_parts` of `containers/{acr}` accepts only `branches, versions` (no dependencies).
* `GET containers/{acr}/versions/{v}/env_vars` -> pairs [{name, value, envVarRef}]; `POST` the same path sets values.
* `GET containers/{acr}/versions/{v}/team_bindings` -> team_bindings [{name, participant_id, user_members[], group_members[],
  manager_name}].
* `GET installedApps/teams` needs `filter` and `snapshotId`.

## Product database facts (verified on 20.0.0.1 / Db2, read only through the engine)

* Every repository object is versioned in `LSW_PO_VERSIONS` (PO_ID, PO_VERSION_ID, BRANCH_ID, START_SEQ_NUM, END_SEQ_NUM); the snapshot
  a version belongs to is found by `v.BRANCH_ID = s.BRANCH_ID AND v.START_SEQ_NUM <= s.SEQ_NUM AND v.END_SEQ_NUM > s.SEQ_NUM` against
  `LSW_SNAPSHOT` (SNAPSHOT_ID, NAME, ACRONYM, SEQ_NUM, BRANCH_ID, PROJECT_ID, IS_ACTIVE, IS_DEFAULT, IS_ARCHIVED) - there is no foreign key.
* Toolkit dependencies: `LSW_PROJECT_DEPENDENCY` (NAME, TARGET_SNAPSHOT_ID, IS_SYSTEM, RANK, VERSION_ID) joined that way; the target
  snapshot's project is the toolkit (`LSW_PROJECT.SHORT_NAME`, `IS_TOOLKIT`). No REST API lists them.
* Undercover agents: `LSW_UCA` (NAME, SCHED_TYPE 1 = scheduled / 2 = event, IS_ENABLED, FREQ_TYPE, DAY_LIST, HOUR_LIST, MINUTE_LIST as
  two-digit lists, 41-47 = Sunday-Saturday, IMPLEMENTATION_TYPE), parameters in `LSW_UCA_PARM`.
* Event manager queue: `LSW_EM_TASK` (TASK_ID, DESCRIPTION, SCHEDULED_TIME, TASK_STATUS, REPEAT_STRING, LAST_EXECUTION_TIME) - the same
  rows `/ops event_manager_tasks` returns. Tasks: `LSW_TASK` (RCVD_DATETIME, CLOSE_DATETIME, ACTIVITY_NAME, STATUS, USER_ID, GROUP_ID);
  instances: `LSW_BPD_INSTANCE` (CREATE_DATETIME, CLOSE_DATETIME, EXECUTION_STATUS, SNAPSHOT_ID, BPD_NAME).
* The engine has no delegation / absence feature and no `/user/{login}/delegate` endpoint; a "task delegation" tool has nothing to call.
* `GET /performance/query` and `/performance/instance/{id}` need tracking groups (CWTBG0019E "Couldn't find the tracking group") -
  durations come from the closed tasks (taskReceivedDate -> taskClosedDate) and the instances instead.

## Event manager facts (verified on 20.0.0.1 and 26)

* `GET /ops/std/bpm/event_manager_tasks?states=&process_id=&size=` - states exactly `acquired, blackedout, executing, on_hold,
  scheduled`; rows id, description, state, job_queue, scheduled_time. `DELETE ...?em_task_ids=`, `POST .../{id}/replay`.
* `POST /ops/std/bpm/event_manager/pause` | `/resume` (every node); `GET /ops/std/bpm/event_manager/monitor` (schedulers) exists on
  24+ only (404 on 20.0.0.1). `DELETE /ops/std/bpm/uca/event_manager_tasks/?container=&version=&undercoverAgentName=&scheduled_after=`
  removes the queued runs of one scheduled UCA. `GET /ops/std/bpm/processes/count?containers=<acr>[&versions=<name>]&states=running`;
  instances on the unnamed tip are not counted under a version name.
* `GET /ops/std/bpm/installedApps/teams?filter=<name or %>&snapshotId=` lists the teams of a snapshot (ids), not the exposure of items.

## Build and deployment pitfalls (all verified)

* A config option the view does not declare (for example `options={...}` passed to a kit control) imports and fails at playback with
  `generatecoachng ... 500` (`CWLLG2145E`, `NullPointerException at CoachDataConfigGenerator.generate`) - twxkit refuses it at build time.
* A re-imported snapshot whose objects keep the previous versionIds behaves like the previous snapshot; twxkit derives versionIds from the
  content. A server restart does not help.
* Environment variable defaults are placeholders in the served packages: the flows answer `401` until `serverBaseURL`,
  `restAuthUser`, `restAuthPassword` are set on the snapshot. 8.6.2 has no REST call for that: set them in Process Admin, or build a lab
  variant with `KITENV_*` under a new snapshot name (a snapshot caches `tw.env`).
* A generator that inserts its own folder first on `sys.path` imports a stale `twxkit.py` lying next to it; check `twxkit.__file__`.
* A `pc_import.py` run killed after the wizard submitted still completes on the server: check the snapshot list before re-importing
  the same snapshot name.
* Direct flow runs: `POST /service/{id}?action=start` wants `params={"data": {...}}` keyed by the input parameter name (CWTBG0543E
  "variable does not exist" otherwise).
* `/bpm/system/login` answers 201; `POST /searches/tasks` answers 201; `DELETE /searches/tasks/{id}` answers 204 - check 2xx, not 200.
* The environment variable export returns stored values in clear, technical-user passwords included: administrators only.
* Two `pc_import.py` runs at the same time on one console work (separate browser sessions) but double the wizard time; the queue runner
  serialises them.

* **Never write `tw.local` in a coach event expression** (button click, table row selection, modal primary button, Service Call
  result): the browser has no `tw` object, so the button throws `ReferenceError: tw is not defined` and does nothing - on every version.
  Pass the service input instead: `${SvcAssign}.execute({taskIds: ids.join(","), toUser: ${AssignUser}.getText(), toGroup: ""})`;
  values from control getters (`getText()`, `getSelectedRecords()`), values of an earlier service result through a hidden Output Text
  set in that service's On result (`result` is the output). twxkit refuses `tw.local / tw.env / tw.system / tw.object` in any `event*`
  option at build time. Found in nine served kit apps whose write buttons had only been proven over REST; fixed in SLA Deadline Monitor
  1.1.1, Team Workload Balancer, Process Smoke Test Runner, Orphan Zombie Cleaner, Deployment Runbook Generator, Role Inbox, Event
  Manager Monitor, Notification Manager and Business Data Search 1.0.1. The `execute(input)` pattern is verified in the browser on
  CP4BA 24.0.1 and 8.6.2 (Team Workload Balancer: person row, reassignment through the modal and return to the team, task owner
  checked over REST); the other fixed apps build under the guard and play back without JavaScript errors - their write buttons have
  not been driven end to end yet.

## Verification summary

Every app was imported on the lab, every flow run over REST with `kit_flow_test.py` (real data: tasks reassigned and returned,
instances started / terminated / deleted, e-mails delivered to an SMTP sink) and the dashboard played back with `dash_test.py`
(no JavaScript error, no failed HTTP call). The `/ops` apps were verified on BAW 26 and, through the other-server path, on the
20.0.0.1 lab. Runtime facts learnt on the way: the task search index lags a few seconds behind an assignment; instances started from
the designer run on the unnamed tip, which `GET /processApps` never lists; `PUT /process/bulk` answers 200 for terminate and delete of
a completed instance; `GET /searches/tasks` lists shared definitions of every owner.

## CP4BA 24 (verified on CP4BA 24.0.1 Workflow Authoring and a connected Process Server)

* Build for the Cloud Pak: `TWXKIT_TARGET=cp4ba TWXKIT_MEMBERS=<ldap users> KITENV_restAuthUser=<ldap user> KITENV_restAuthPassword=...
  python3 build_<app>.py --snapshot <v>` (System Data `8.6.0.0_TC`, serverBaseURL `https://localhost:9443/bas`); the served
  `-CP4BA.twx` packages are built that way with placeholder credentials. All sixteen kit packages import on the Studio with 0
  `assetsValidation` errors.
* Every kit dashboard was played back on the Studio (header from the Init flow, every tab, every non-writing button with its status line -
  `kit_smoke.py`): tasks, instances, teams, exposed items, saved searches, `/ops` containers / snapshots / environment variables, event
  manager tasks and the product database queries (`kitSql` through the engine datasource) all answer through the `/bas` loopback.
* Process Server: the snapshot created on the Studio after the import (`designer_snapshot.py`, the imported one renders unstyled) was
  installed on the Process Server, `serverBaseURL` set to `https://localhost:9443/baw-<instance>` with
  `POST /ops/std/bpm/containers/<acr>/versions/<v>/env_vars {"pairs": [...]}` (live), and the same smoke test passed there for all
  sixteen apps (two old builds still showed the event-expression error fixed above).
