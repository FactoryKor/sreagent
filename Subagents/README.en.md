[한국어](README.md) | **English**

# Azure SRE Agent — Custom Agent (Subagent) Instruction Pack

Instruction text for the custom agents ("subagents") that make the **diag-tools MCP server**
usable against the **Total-Lab / full-lab** environment.

The instruction text in this folder is written in **English** and is meant to be pasted directly into
**Azure portal → your SRE Agent → Builder → Agent Canvas → Create → Custom Agent**.

Every file exists in two languages. `<name>.en.md` is English, `<name>.md` is Korean, and each
file links to its counterpart on the first line.

> **Which one do I paste into the portal?**
> Paste the **English** version. The instructions are what the model reads, and English keeps
> tool names, severity values and JSON keys unambiguous. The Korean files are a faithful
> translation for reading, review and maintenance. Output language is a separate matter — every
> agent answers in the language of the user's message, so a Korean question gets a Korean answer
> even when the instructions are English.

---

## 1. What the lab actually deploys (ground truth)

Source: `Total-Lab/full-lab/main.bicep` + `modules/*.bicep`. Default `namePrefix` is `dtlab`.

| Lab resource | Resource name pattern | Log Analytics `Computer` / DB | Matching MCP tool |
|---|---|---|---|
| Windows Server 2022 + SQL Server 2022 Dev VM | `<prefix>-win-sql` | `win-sql` | `diagnose_windows`, `diagnose_mssql` |
| Ubuntu VM + MySQL Server | `<prefix>-linux-mysql` | `linux-mysql` | `diagnose_linux`, `diagnose_mysql` |
| Azure SQL Database (PaaS) | `<prefix>-sqlsvr-<hash>` / db `diagdb` | — | `diagnose_mssql` |
| Azure DB for MySQL Flexible Server | `<prefix>-mysql-<hash>` / db `diagdb` | — | `diagnose_mysql` |
| Azure DB for PostgreSQL Flexible Server | `<prefix>-pg-<hash>` / db `diagdb` | — | `diagnose_postgres` |
| Log Analytics workspace | `<prefix>-law` | — | (input for windows/linux/svcmap) |
| Application Insights | `<prefix>-appi` | — | (input for svcmap) |
| DCRs (Windows Perf/Event, Linux Perf/Syslog, VM Insights) | `<prefix>-dcr-*` | — | — |
| Dependency Agent (**off by default**) | — | — | `diagnose_service_map` |

Not deployed by this lab: AKS, Event Hubs, Application Gateway, App Service, ADX, SAP HANA.
Agents for those tools are still provided (see `08-optional-agents.md` and `11-appgateway-expert.md`) so the same MCP
connector can be reused when you extend the lab.

---

## 2. MCP tools actually registered on the server

Verified against `FactoryKor/mcp` → `mcp_server.py` (the image running in Azure Container Apps).
Use these **exact** names when you select Custom Tools for each agent.

| Tool | Required args | Optional args |
|---|---|---|
| `diagnose_postgres` | `host` | `user` (leave empty = MCP identity), `dbname`, `resource_id`, `hours` |
| `diagnose_mssql` | `host` | `user`, `database`, `auth_mode`, `resource_id`, `region`, `hours`, `password_env` |
| `diagnose_mysql` | `host` | `user`, `database`, `auth_mode`, `resource_id`, `region`, `hours`, `password_env` |
| `diagnose_mysql_fleet` | `targets` (list of `host`/`resource_id`/`region`/`database`) | `user`, `auth_mode`, `hours`, `deep`, `password_env`, `max_findings_per_server`, `per_target_timeout` |
| `diagnose_windows` | `computer` | `workspace_id`, `resource_id`, `hours` |
| `diagnose_linux` | `computer` | `workspace_id`, `resource_id`, `hours` |
| `diagnose_service_map` | one of `appinsights_id` / `workspace_id` | `workload`, `window_minutes` |
| `diagnose_aks` | — | `namespace`, `context`, `all_namespaces`, `prometheus_url`, `appinsights_id` |
| `diagnose_adx` | `cluster` | `database`, `resource_id`, `region`, `hours` |
| `diagnose_eventhub` | `resource_id` | `event_hub`, `region`, `window_minutes`, `checkpoint_store` |
| `diagnose_appgateway` | `resource_id` | `region`, `window_minutes`, `backend_health`, `workspace_id` |
| `diagnose_webapp` | `resource_id` | `region`, `window_minutes` |
| `diagnose_hana` | `host`+`user` or `userkey` | `port`, `deployment_type`, `hours`, `password_env` |
| `diagnose_avd` | `host_pool_id` | `workspace_id`, `hours`, `deep`, `max_hosts`, `storage_account_ids`, `anf_volume_ids`, `storage` |
| `diagnose_citrix` | — | `subscription`, `resource_group`, `vda_prefix`, `connector_prefix`, `connector_vms`, `workspace_id`, `citrix_*`, `api_host`, `client_secret_env`, `hours`, `deep`, storage args |

DB tools (`diagnose_postgres`, `diagnose_mssql`, `diagnose_mysql`, `diagnose_mysql_fleet`) return a
top-level `diagnostic_identity` object: the MCP managed identity that actually connected. That is
the principal a DBA must set up — never the SRE Agent identity.

### Privileged and deep-analysis tools

The server registers 31 tools in total. Beyond the 15 `diagnose_*` tools above, these come from the
`sre-ops` layer. Which agent gets which tool is defined in `../PRIVILEGED-OPS-BUILD-GUIDE.md`
(section 9.1): the read-only analysis tools also go to the Windows/Linux experts, everything that
creates a request or a grant stays with `privileged_ops_expert`. Tools marked **approval** are refused with `approval_required` until an
administrator approves the matching request out-of-band (`sre-ops approve`).

| Tool | Purpose | Approval |
|---|---|---|
| `describe_diagnostic_identity` | Which managed identity needs which permission | — |
| `ensure_diagnostic_access` | Read-only role (Reader / Monitoring Reader / Log Analytics Reader) for a short TTL; immediate in `auto` mode, approval request in `approval` mode | mode-dependent |
| `detect_memory_leak` | Time-series regression to name the leaking process | — |
| `list_os_dumps` | Existing dumps and dump configuration state | — |
| `preflight_process_dump` | Estimate dump size, pause time, free disk | — |
| `list_staged_dumps` / `analyze_dump` | Staged dump inventory / structural analysis | — |
| `request_privileged_action` | Create an approval request (changes nothing) | — |
| `list_privileged_requests` | Pending approval queue | — |
| `grant_temporary_access` | Time-boxed grant (TTL default 60 min, capped by `maxTemporaryAccessMinutes`) | **approval** |
| `capture_process_dump` / `stage_os_dump` | Capture / export a memory dump | **approval** |
| `revoke_temporary_access` | Revoke immediately — never needs approval | — |
| `list_temporary_access` | Active leases; auto-revokes expired ones on every call | — |
| `get_audit_log` | Append-only audit ledger | — |
| `approve_privileged_action` | Human approval path for the `secret` channel only. **Never attach to any agent** | n/a |

Every grant goes to the MCP managed identity. `principal_object_id` must be left empty; the server
rejects any other principal unless the operator listed it in `SRE_OPS_ALLOWED_PRINCIPALS`.

JIT providers: `azure_rbac` (also used for VM Run Command, so **no local admin account is ever
created on a VM**), `postgresql_entra_admin`, `mysql_entra_admin`, `mssql_entra_admin`. The last
two replace the single Entra-admin slot and restore the previous admin on revoke.

> `diagnose_windows` / `diagnose_linux` take no `source` argument — Azure Monitor mode only.
> Direct WinRM/SSH mode exists only in the CLI.

---

## 3. Files in this pack

| File | Custom agent | Purpose |
|---|---|---|
| `01-lab-orchestrator.md` | `lab_diagnostics_orchestrator` | Entry point; discovers targets, fans out, aggregates |
| `02-windows-os-expert.md` | `windows_os_expert` | `diagnose_windows` + leak/dump read-only analysis |
| `03-linux-os-expert.md` | `linux_os_expert` | `diagnose_linux` + leak/dump read-only analysis |
| `04-sqlserver-expert.md` | `sqlserver_expert` | `diagnose_mssql` (IaaS + Azure SQL) |
| `05-mysql-expert.md` | `mysql_expert` | `diagnose_mysql`, `diagnose_mysql_fleet` (IaaS + Flexible Server) |
| `06-postgresql-expert.md` | `postgresql_expert` | `diagnose_postgres` |
| `07-service-map-expert.md` | `service_map_expert` | `diagnose_service_map` |
| `08-optional-agents.md` | 5 agents | AKS / ADX / Event Hubs / Web App / HANA (8.4 App Gateway moved to `11`) |
| `09-shared-conventions.md` | — | Text blocks reused by every agent (output contract, output language, environment profile, guardrails) |
| `10-privileged-ops-expert.md` | `privileged_ops_expert` | JIT grant/revoke, leak detection, dumps, audit ledger |
| `11-appgateway-expert.md` | `appgateway_expert` | `diagnose_appgateway` — backend health, gateway-vs-backend 502/504, TLS/certificates, capacity, access/WAF logs |

---

## 3.5 Output language and environment profile

**Output language is not fixed by this pack.** Every agent answers in the language of the user's
latest message and keeps it until the user switches. Ask in Korean, get Korean; ask in English,
get English. Severity enum, tool and argument names, metric names, SQL/KQL text, resource ids and
the "Not evaluated" heading are **never** translated, so the orchestrator can still merge expert
results. For customer-facing documents each agent can emit a severity mapping table
(critical→위험, warning→주의, info→정보, ok→양호) once, instead of translating inline.

**Environment profile matters.** Each expert carries a Total-Lab block. Those allowances — public
endpoint, basic/burstable SKU, no HA, minimal retention, synthetic traffic — apply **only when
ENVIRONMENT = lab**. Against a customer production environment the agents keep the original
severity. When the profile is ambiguous the agents ask once and otherwise assume production.
See `09-shared-conventions.md` sections F and G.
## 4. Recommended build order

1. Create the **domain experts** first (02 → 07). They have no handoff dependencies.
2. Create `lab_diagnostics_orchestrator` (01) last and set its **Handoff Agents** to all experts.
3. In each agent, attach only the MCP tools listed in its file. **Do not use the connector
   wildcard (`<connection>/*`)** — it hands `grant_temporary_access`, `capture_process_dump` and the
   other privileged tools to every expert. The per-agent matrix is in
   `../PRIVILEGED-OPS-BUILD-GUIDE.md`, section 9.1.
4. For built-in tools select **`RunAzCliReadCommands` only** (Resource Graph / az read) plus
   `execute_kusto_query`. Never select `RunAzCliWriteCommands` or `RunPsqlReadCommand` for any agent
   in this pack, and keep **Enable skills off** for the experts. The MCP tools require ARM resource
   IDs and workspace GUIDs that only Resource Graph can supply.
5. Run each agent once in **Builder → Agent Canvas → Test playground** with the sample prompt
   included at the bottom of each file.

---

## 5. Run mode and tool policy

- All `diagnose_*` tools and the read-only analysis tools are **strictly read-only**. Allow them in
  tool access policies.
- Keep response plans / scheduled tasks in **Review** mode while validating the lab. The experts
  never mutate anything, but the orchestrator may propose Azure CLI mitigations.
- Instruction text alone does not stop the **main agent** — it is not bound by these prompts and
  holds every built-in tool. Add the **global deny** policies from
  `../PRIVILEGED-OPS-BUILD-GUIDE.md` section 8.4 (`ad-admin`, `role assignment create`,
  `RunPsqlReadCommand`, …). That is what actually stops "grant the SRE Agent a DB admin role".

---

## 6. Prerequisites the agents cannot fix themselves

The instruction blocks tell each agent to report these as blockers instead of looping:

| Tool | Prerequisite | Where to fix |
|---|---|---|
| `diagnose_postgres` | MCP managed identity (`diagnostic_identity.principal_name`) registered as Entra role in PostgreSQL + `GRANT pg_monitor` | Flexible Server → Authentication |
| `diagnose_mssql` (Azure SQL) | MCP managed identity created as a contained DB user with VIEW DATABASE STATE (+ `##MS_ServerStateReader##` in master for instance-wide DMVs) | `CREATE USER [<mi-name>] FROM EXTERNAL PROVIDER` |
| `diagnose_mysql` (`auth_mode="entra"`) | Entra auth enabled + MCP managed identity created as `AADUSER` with PROCESS / REPLICATION CLIENT / SELECT on performance_schema | Flexible Server → Authentication |
| `diagnose_mssql` (IaaS, `auth_mode="sql"`) | `MSSQL_DIAGNOSE_PASSWORD` env var injected into the Container App | ACA → Containers → Environment variables / Key Vault ref |
| `diagnose_mysql` (`auth_mode="mysql"`) | `MYSQL_DIAGNOSE_PASSWORD` env var injected into the Container App | same as above |
| `diagnose_windows` / `diagnose_linux` | AMA + DCR association active; allow 10–15 min after deploy for first data | `<prefix>-dcr-windows` / `<prefix>-dcr-linux` |
| `diagnose_service_map` | Dependency Agent installed (`00_deploy.ps1 -EnableDependencyAgent`) and 15–60 min of traffic | redeploy lab with the switch |
| All | MCP managed identity holds `Reader` + `Monitoring Reader` on the lab resource group | `infra/assign-roles.ps1` |
| `list_os_dumps` / `preflight_process_dump` / `capture_process_dump` / `stage_os_dump` | MCP managed identity holds the `SRE Diagnostic Run Command Operator` role on the VM resource group | `infra/assign-privileged-roles.ps1 -RunCommandScope` |

---

## 7. Optional: YAML authoring

If you use the SRE Agent MCP server extension for VS Code, each agent file also contains a YAML
block matching the documented schema (`name`, `system_prompt`, `handoff_description`, `tools`).
The portal form fields map as: Name → `name`, Instructions → `system_prompt`,
Handoff Description → `handoff_description`, Custom Tools/Built-in Tools → `tools`.
