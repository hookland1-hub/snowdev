# Handoff — ServiceNow Now SDK, full‑potential authenticated setup (single self‑contained bootstrap)

**Purpose.** This is the **one document** an operator (or an AI agent on a brand‑new machine, with no
prior context) reads to stand up a **100 % working ServiceNow development session** using the Now SDK
+ Fluent at full capability: connected to an instance, with the official ServiceNow skill and the
Superpowers skills installed, ready to build, install, round‑trip and verify real applications.

It is an **initial setup** guide — generic, project‑agnostic. It does **not** carry any specific
user‑story / business detail. Placeholders:

- `<scope>` → `x_<vendor>_<app>` (vendor prefix + app code), all lowercase.
- `<scopeId>` → a stable 32‑char hex sys_id for the scope.
- `<AppName>` → human application name.
- `<instance>` → `https://<name>.service-now.com`.
- `<alias>` → a short name for a saved auth credential.

Everything below was **verified against `@servicenow/sdk` 4.7.2** and **Claude Code 2.1.x** on
Windows. Where a fact is version‑dependent, the doc says to re‑check with `--help` / `explain`.

---

## PART A — Configure the AI session (do this FIRST, once per machine)

The AI session's ServiceNow superpowers come from **Claude Code plugins** (which provide *skills*),
plus the **Now SDK** CLI. Three installs, in order.

### A.1 Base tooling

```
OS:        Windows (PowerShell) or any OS with a POSIX shell
Node.js:   recent LTS or newer        (validated alongside SDK 4.7.2)
npm:       bundled with Node
Claude Code: 2.1.x or newer (the `claude` CLI must be on PATH)
Optional:  ripgrep (rg), git
```

### A.2 Install the Now SDK

Install globally so `now-sdk` is on PATH (it is also pulled per‑project as a devDependency):

```powershell
npm install -g @servicenow/sdk           # provides the `now-sdk` binary
now-sdk --version                        # expect 4.7.x (>= 4.6.0 required for `explain`)
```

### A.3 Install the Claude Code skills (THE part that makes the agent capable)

Skills are delivered as plugins from **git marketplaces**. Add the two marketplaces, then install the
two plugins. Verified command surface (`claude plugin --help`):

```powershell
# 1) Add the marketplaces (GitHub repos):
claude plugin marketplace add anthropics/claude-plugins-official
claude plugin marketplace add obra/superpowers-marketplace

# 2) Install the plugins (syntax: <plugin>@<marketplace>):
claude plugin install servicenow-sdk@claude-plugins-official    # → skill: fluent:now-sdk-explain
claude plugin install superpowers@superpowers-marketplace        # → skills: superpowers:*

# 3) Verify:
claude plugin list                       # both should show Status: enabled
```

What each gives the agent:

| Plugin | Marketplace (GitHub) | Skill(s) it surfaces | Why it matters |
|---|---|---|---|
| `servicenow-sdk` | `anthropics/claude-plugins-official` | `fluent:now-sdk-explain` | Pulls SDK docs on demand (`now-sdk explain`) — API types, conventions, project structure. The agent **must** use this before writing Fluent. |
| `superpowers` | `obra/superpowers-marketplace` | `superpowers:brainstorming`, `:test-driven-development`, `:systematic-debugging`, `:writing-plans`, `:executing-plans`, `:verification-before-completion`, `:requesting-code-review`, … | The disciplined workflow (design → plan → TDD → verify) the agent follows. |

Notes:
- Plugins install at **user scope** by default (`-s project` / `-s local` to scope to a repo).
- Restart the Claude Code session after install so skills load.
- `claude plugin update <plugin>` to upgrade; `claude plugin marketplace update` to refresh sources.
- The agent invokes a skill via its **Skill tool** (e.g. `fluent:now-sdk-explain`); a human types
  `/<skill>` — never read skill files directly.

**Session is "configured" when:** `now-sdk --version` works **and** `claude plugin list` shows both
`servicenow-sdk` and `superpowers` enabled.

---

## PART B — Authenticate & scaffold the project

### B.1 Authenticate to an instance (verified flags: `now-sdk auth --help`)

```powershell
# Add a credential (prefer OAuth; basic also supported):
now-sdk auth --add <instance> --type oauth --alias <alias>
now-sdk auth --add <instance> --type basic --alias <alias>   # alternative

now-sdk auth --list                 # show saved credentials
now-sdk auth --use <alias>          # set the default credential
now-sdk auth --delete <alias>       # remove one
```

Guardrails:
- **Confirm the target instance before any write** (`install`/`deploy`). Default to **dev/sub‑prod**;
  never write to prod without explicit per‑action approval.
- Credentials are cached locally (under the SDK/`.now` state) — **never commit** them, never echo
  tokens. Prefer OAuth over stored basic passwords.
- Per‑command override: most write commands accept `-a, --auth <alias>`.

### B.2 Scaffold or convert an app (verified: `now-sdk init`, alias `create`)

```powershell
now-sdk init                 # interactive: new scoped app, or apply a template,
                             # or convert a legacy/instance app into Fluent source
```

### B.3 Project shape (what `init` produces / what to expect)

```
now.config.json     # scope, scopeId, name, tsconfigPath
package.json        # scripts + deps + "imports": { "#now:*": "./@types/servicenow/fluent/*/index.js" }
src/
  fluent/
    index.now.ts    # MUST `export * from './<each>.now'` (entrypoint)
    *.now.ts        # Table/Role/Acl/ScriptInclude/Workspace/ServicePortal/Flow/… definitions
  server/           # plain-JS bodies for Script Includes, BR, SP widget assets, tsconfig.json
@types/             # GENERATED glide type defs (now-sdk dependencies) — IntelliSense
dist/               # GENERATED by now-sdk build (do not edit)
.now/               # GENERATED SDK state / auth cache (NEVER commit)
keys.ts             # stable Key/SysId map (Now.ID) — keep in source for deterministic identity
```

Never commit / regenerated locally: `node_modules/`, `dist/`, `.now/`, `@types/`.

### B.4 now.config.json

```json
{
  "scope": "<scope>",
  "scopeId": "<scopeId>",
  "name": "<AppName>",
  "tsconfigPath": "./src/server/tsconfig.json"
}
```

Stable scope sys_id (deterministic, so re‑installs match the same app):

```powershell
node -e "console.log(require('crypto').createHash('md5').update('<scope>').digest('hex'))"
```

### B.5 package.json scripts (verified command names)

```json
{
  "scripts": {
    "build": "now-sdk build",
    "deploy": "now-sdk install",
    "transform": "now-sdk transform",
    "types": "now-sdk dependencies",
    "pack:sdk": "now-sdk pack",
    "test": "node tests/validate-source.js"
  },
  "devDependencies": {
    "@servicenow/sdk": "4.7.2",
    "@servicenow/glide": "27.0.5",
    "typescript": "5.5.4"
  }
}
```

Get full IntelliSense for server‑side glide APIs:

```powershell
now-sdk dependencies                 # download glide.*.d.ts type defs + fluent dependency types
now-sdk dependencies --type-defs-only
now-sdk dependencies --add sys_security_acl --scope global   # add a dependency to now.config.json
```

---

## PART C — The `explain` documentation system (MANDATORY before writing Fluent)

The SDK ships its own docs; **the API is the source of truth, not memory.** Always consult `explain`
before writing/altering any Fluent metadata (the `fluent:now-sdk-explain` skill automates this).

```powershell
now-sdk explain --list --format=raw                 # list every topic + search tags
now-sdk explain <topic> --list --peek --format=raw  # search topics, show descriptions
now-sdk explain <topic> --peek --format=raw         # PREVIEW first (saves context, avoids wrong topic)
now-sdk explain <topic> --format=raw                # read full topic
```

Rules:
- **Never** open a full topic without `--peek` first.
- First time per session: skim `explain fluent-overview`, `developing-apps-guide`, and the relevant
  `*-guide` topics for the surfaces you're about to touch.
- `No documentation found` = wrong name → use `--list`. `No match` = try a different search term.

---

## PART D — Verified Fluent capability matrix (full potential)

All of the following are **real `explain` topics in 4.7.2** — declarable in Fluent and installable to a
connected instance. (Grouped; consult each `*-api` / `*-guide` topic for exact syntax.)

**Data model**
- `Table()` (`sys_db_object`/`sys_dictionary`), table augments, dictionary `OverrideColumn`.
- Columns: `StringColumn`, `MultiLineTextColumn`, `HtmlColumn`, `ChoiceColumn`, `BooleanColumn`,
  `IntegerColumn`, `DecimalColumn`/`FloatColumn`, `DateColumn`/`DateTimeColumn`/`TimeColumn`/duration,
  `ReferenceColumn` (cascade), `ListColumn`/`SlushBucketColumn`, `JsonColumn`, `UrlColumn`,
  `EmailColumn`, `Password2Column`, `GuidColumn`, `DocumentIdColumn`, `ConditionsColumn`,
  `ScriptColumn`, translated/i18n columns, and more.

**Security**
- `Role()`, `Acl()` (`type:'record'` and `type:'ux_route'`), `DataPolicy()`, `CrossScopePrivilege()`.

**Server logic & automation**
- `ScriptInclude()`, `BusinessRule()`, `ScheduledScript()`, `ScriptAction()` (events),
  `EmailNotification()`, `InboundEmailAction()`, `Property()` (system properties), `RestApi()`
  (scripted REST), `ImportSet()` (transform maps), `Sla()`.

**Flow / Workflow Automation (wfa)**
- `Flow()`, `Subflow()`, custom `Action()`, `FlowStage`, triggers, flow logic.

**Classic UI**
- `ClientScript()`, `UiAction()`, `UiPolicy()`, `Form()`, `List()`, `UiPage()` (Jelly/HTML **or**
  React/Polaris — supports SPA‑style custom pages), platform views.

**Now Experience / Workspace**
- `Workspace()` (`sys_ux_page_registry`), `UxListMenuConfig()`, `Applicability()`, `Dashboard()`.

**Service Portal** (fully supported — see Part E)
- `ServicePortal()`, `SPPage()`, `SPWidget()`, `SPTheme()`, `SPMenu()`, `SPAngularProvider()`,
  `SPHeaderFooter()`, `SPWidgetDependency()`, `SPPageRouteMap()`.

**Service Catalog**
- `CatalogItem()`, `CatalogItemRecordProducer()`, `VariableSet()`, `CatalogUiPolicy()`,
  `CatalogClientScript()`, and the full set of catalog `*Variable` types.

**Testing & quality**
- `Test()` (ATF, `sys_atf_test`) — definitions can also **run** on the connected instance.
- Instance Scan checks: `ColumnTypeCheck`, `LinterCheck`, `TableCheck`, `ScriptOnlyCheck`.

**AI / Now Assist**
- `NowAssistSkillConfig()`, `AiAgent()`, `AiAgenticWorkflow()`.

**Escape hatches**
- `Record()` (generic, any table), `$override` (unknown/custom `x_`/`u_` properties),
  `UserPreference()`.

### ⚠ NOT authorable in Fluent (verified — no such `explain` API)
Custom **UX Builder pages / components / macroponents** (`sys_ux_page`, `sys_ux_component`,
`sys_ux_macroponent`). Fluent only models the **workspace registry / list‑menu / applicability /
dashboard** UX surfaces above — there is **no `UxPage()`/component/macroponent API**, with or without
auth. Two valid paths:
1. **Build a custom single‑page UI as Service Portal (`SPPage`+`SPWidget`) or a React `UiPage`** —
   both are 100 % source‑authorable.
2. **Author the UX Builder page on the instance UI, then `now-sdk transform`/`download` it into Fluent
   source** — this is the round‑trip auth unlocks (see D.3). Net‑new authoring from scratch in Fluent
   is not supported.

### Cross‑cutting helpers
- `Now.ID['stable_key']` (+ `keys.ts`) → stable `$id`/sys_id so re‑installs are idempotent.
- `Now.include('relative/path.ext')` → inline an external file (server JS, client JS, HTML, CSS) into a
  record field.
- `Now.ref(...)` → reference another defined record (coalesce/foreign key).
- `Now.attach(...)` → attach an image/file (logo, icon).
- `index.now.ts` must `export * from './<each>.now'` so every metadata file is built.

---

## PART D.3 — Build / install / round‑trip lifecycle (verified command surface)

The full `now-sdk` command set (`now-sdk --help`): `auth`, `init` (alias `create`),
`download <directory>`, `build [source]`, `install` (**alias `deploy`**), `dependencies [sysIds..]`,
`transform`, `clean [source]`, `pack [source]`, `explain [topic]`.

> There is **no separate promotion command**: `deploy` is an *alias of `install`*. You promote by
> installing against a different instance/credential, or by exporting an update set for change control.

### Build
```powershell
now-sdk build                 # compile Fluent → dist/ + installable package
now-sdk build --frozenKeys    # CI: fail if keys/sysIds drift (deterministic identity)
now-sdk build --errorOnConflict   # treat Fluent↔XML sys_id conflicts as errors
now-sdk build --skipClean
```

### Install (push to the connected instance) — verified flags
```powershell
now-sdk install                         # install/update app on the default-auth instance
now-sdk install -a <alias>              # target a specific credential/instance
now-sdk install --demoData              # (default true) include demo data
now-sdk install --skip-flow-activation  # don't auto-publish flows
now-sdk install -r                      # reinstall: uninstall+reinstall (⚠ on-instance-only metadata is LOST)
now-sdk install -b                      # open the sys_app page in browser after install
now-sdk install -i                      # info on the most recent install of this app
```
`install` is idempotent by sys_id (update in place). Re‑run after every change; no manual XML import,
no Retrieved‑Update‑Sets step.

### Round‑trip (what auth uniquely unlocks)
```powershell
now-sdk download <directory>            # download app metadata from the instance
now-sdk transform -a <alias>            # download instance XML records → Fluent source
now-sdk transform --from <path>         # convert local XML file(s)/dir → Fluent source
now-sdk transform --table <t1,t2>       # transform by table hierarchy
```
Use `transform`/`download` to pull instance‑side work (incl. UX Builder pages built in the UI) back
into source, or to convert a legacy app. Review the generated diff before committing.

### Package / clean
```powershell
now-sdk pack                            # zip a source/installable artifact (NOT an update set)
now-sdk clean                           # clean output dirs
```

### CI / headless (verified via `explain ci-integration`)
Authenticate without interactive prompts using env vars:
```
SN_SDK_NODE_ENV, SN_SDK_AUTH_TYPE (basic|oauth|client-credentials),
SN_SDK_INSTANCE_URL, SN_SDK_USER, SN_SDK_USER_PWD,
SN_SDK_OAUTH_CLIENT_ID, SN_SDK_OAUTH_CLIENT_SECRET
```
Pair with `now-sdk build --frozenKeys` so a CI build fails if `keys.ts`/sys_ids drift.

---

## PART E — Service Portal specifics (hard‑won; read before building SP)

(Apply whenever you use Service Portal — auth does not relax these.)

1. **`SPPage` takes no top‑level `$id`** (keyed by `pageId`); nested containers/rows/columns/instances
   **do** use `$id: Now.ID[...]`. A top‑level `$id` on SPPage fails the build.
2. **Widget client script must be a bare named function**: `function controller() { var c = this; … }`.
   `api.controller = function(){…}` is **rejected**. Dependencies inject as parameters
   (`function controller(spModal) { … }`).
3. **Browser globals are blocked** in client scripts: no `window`, `document`, `alert`, `confirm`.
   Use injected AngularJS services (`spModal.confirm()`, `spModal.alert()`).
4. **`customCss` is served AS‑IS, NOT Sass‑compiled.** Plain CSS only — no nesting, `$variables`,
   `@each`, `#{}`. Any Sass syntax makes the browser discard the **entire** stylesheet (symptom:
   widget renders as unstyled text). Theme via **CSS custom properties** (`--var`).
5. **Bootstrap 3** classes are available globally in Service Portal.
6. Widget server script runs server‑side in the app scope: same‑scope Script Includes are callable
   directly (no GlideAjax). `data.*` → client; `c.data.*` + `c.server.update()` round‑trips; posted
   `data` arrives as `input`.
7. SP widget fields in build output: `template`, `client_script`, `script`, `css`, `option_schema`,
   `demo_data`, etc.

Consult `explain service-portal-guide` and `service-portal-reference-guide` for the full set.

---

## PART F — Platform patterns & security discipline

- **Scoped GlideRecord enforces ACLs by default** — widgets/automation respect your Fluent ACLs.
- **Logical deletion**: every business table carries `active` (Boolean, default true); "delete" sets
  `active=false`; consumers filter `active=true`.
- **Auto‑number**: a `number` String column + platform Number Maintenance (`sys_number`) per table.
- **Retention / data‑minimization**: a `Date` column (e.g. `retention_date`), a configurable retention
  property, a scheduled job listing expiring records, UI Actions to extend/delete.
- **Scope protection**: restricted runtime access; grant cross‑scope only to named consumer scopes via
  `CrossScopePrivilege()` / `ScriptInclude` `accessibleFrom`.
- **ACL pattern that builds cleanly**: `type:'record'` ACLs per table per operation
  (`read`/`create`/`write`/`delete`), `read` to reader+editor, write‑ops to editor,
  `adminOverrides:true`; plus a `type:'ux_route'` ACL `name:'<workspace-path>.*'`, `operation:'read'`
  for workspace route access.

---

## PART G — Verification & the AI operating contract (fresh session)

1. **Configure the session** (Part A): `now-sdk --version` works; `claude plugin list` shows
   `servicenow-sdk` and `superpowers` enabled. If not, install per A.2/A.3.
2. **Authenticate deliberately** (Part B): confirm the **target instance** before any write; default
   to dev/sub‑prod; keep credentials/tokens secret and uncommitted.
3. **Before changing any Fluent metadata, consult `explain`** (`--peek` then full) via the
   `fluent:now-sdk-explain` skill — the API evolves; never code from memory.
4. **Follow the Superpowers discipline**: brainstorm → write a plan → TDD/validators → implement →
   verify. Add `validate-source.js` assertions before risky changes; keep them green.
5. **Build → install → verify on the live instance with fresh output** before claiming success:
   load the app, the workspace/portal route, exercise ACLs as a non‑admin, **run ATF** on the
   instance. Evidence, not assertion.
6. Keep secrets and real personal data out of artifacts; demo/seed data must be non‑personal.
7. **Deliver**: the app installed/verified on the named instance, the source repo (Fluent + `keys.ts`),
   a short note of what to verify and which roles to assign; optional `now-sdk pack` ZIP; for
   air‑gapped/change‑controlled prod, an exported update set instead of a direct `deploy`.

**"Ready to develop" only after confirming:** Node/npm present; `now-sdk` ≥ 4.6; both skills enabled;
`now-sdk auth --list` shows the intended instance as default; `now-sdk dependencies` run (types
present); `npm test` passes; `now-sdk build` passes; `now-sdk install` succeeded and the app is
verified in the instance UI; ATF run if in scope; no secrets committed.

---

## Appendix — Quick command reference (all verified, SDK 4.7.2)

```
# session setup
npm install -g @servicenow/sdk
claude plugin marketplace add anthropics/claude-plugins-official
claude plugin marketplace add obra/superpowers-marketplace
claude plugin install servicenow-sdk@claude-plugins-official
claude plugin install superpowers@superpowers-marketplace
claude plugin list

# auth
now-sdk auth --add <instance> --type oauth --alias <alias>
now-sdk auth --list ; now-sdk auth --use <alias>

# project
now-sdk init                       # scaffold / convert
now-sdk dependencies               # glide type defs + fluent deps
now-sdk explain --list --format=raw
now-sdk explain <topic> --peek --format=raw

# dev loop
now-sdk build                      # → dist/
now-sdk install [-a <alias>] [-b]  # push to instance (alias: now-sdk deploy)
now-sdk transform -a <alias>       # pull instance records → Fluent source
now-sdk pack                       # source ZIP (NOT an update set)
now-sdk clean
```
