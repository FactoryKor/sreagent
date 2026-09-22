[한국어](README.md) | **English**

# Azure SRE Agent — Custom Agent (Subagent) Instruction Pack

Instruction text for the custom agents ("subagents") that make the **diag-tools MCP server**
usable from Azure SRE Agent.

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
Agents for those tools are still provided (see `08-optional-agents.en.md`) so the same MCP
connector can be reused when you extend the lab.

---

## 2. MCP tools registered on the server

Verified against `FactoryKor/mcp` → `mcp_server.py` (the image running in Azure Container Apps).
Use these **exact** names when you select Custom Tools for each agent.

| Tool | Required args | Optional args |
|---|---|---|
| `diagnose_postgres` | `host` | `user`, `dbname`, `resource_id`, `hours` |
| `diagnose_mssql` | `host` | `user`, `database`, `auth_mode`, `resource_id`, `region`, `hours`, `password_env` |
| `diagnose_mysql` | `host` | `user`, `database`, `auth_mode`, `resource_id`, `region`, `hours`, `password_env` |
| `diagnose_windows` | `computer` | `workspace_id`, `resource_id`, `hours` |
| `diagnose_linux` | `computer` | `workspace_id`, `resource_id`, `hours` |
| `diagnose_service_map` | one of `appinsights_id` / `workspace_id` | `workload`, `window_minutes` |
| `diagnose_aks` | — | `namespace`, `context`, `all_namespaces`, `prometheus_url`, `appinsights_id` |
| `diagnose_adx` | `cluster` | `database`, `resource_id`, `region`, `hours` |
| `diagnose_eventhub` | `resource_id` | `event_hub`, `region`, `window_minutes`, `checkpoint_store` |
| `diagnose_appgateway` | `resource_id` | `region`, `window_minutes`, `backend_health` |
| `diagnose_webapp` | `resource_id` | `region`, `window_minutes` |
| `diagnose_hana` | `host`+`user` or `userkey` | `port`, `deployment_type`, `hours`, `password_env` |

> `diagnose_postgres` no longer requires `user`. When it is omitted the server falls back to the
> container's own managed identity name (`DIAG_DB_USER` → `SRE_OPS_PRINCIPAL_NAME`). An
> authentication failure such as an OID mismatch is almost always a wrong principal name, not a
> missing permission.

### v1.3+ privileged and deep-analysis tools (`privileged_ops_expert` only)

The server registers 28 tools in total. Beyond the 12 `diagnose_*` tools above, these come from the
`sre-ops` layer. Tools marked **approval** are refused with `approval_required` until an
administrator approves the matching request out-of-band (`sre-ops approve`).

| Tool | Purpose | Approval |
|---|---|---|
| `describe_diagnostic_identity` | Which managed identity needs which permission | — |
| `detect_memory_leak` | Time-series regression to name the leaking process | — |
| `list_os_dumps` | Existing dumps and dump configuration state | — |
| `preflight_process_dump` | Estimate dump size, pause time, free disk | — |
| `list_staged_dumps` / `analyze_dump` | Staged dump inventory / structural analysis | — |
| `request_privileged_action` | Create an approval request (changes nothing) | — |
| `list_privileged_requests` | Pending approval queue | — |
| `ensure_diagnostic_access` | Obtain a read-only role when missing — grants immediately in `auto` mode, otherwise creates an approval request | mode-dependent |
| `grant_temporary_access` | Time-boxed grant (TTL default 60 min, capped by `maxTemporaryAccessMinutes`) | **approval** |
| `capture_process_dump` / `stage_os_dump` | Capture / export a memory dump | **approval** |
| `revoke_temporary_access` | Revoke immediately — never needs approval | — |
| `list_temporary_access` | Active leases; auto-revokes expired ones on every call | — |
| `get_audit_log` | Append-only audit ledger | — |

JIT providers: `azure_rbac` (also used for VM Run Command, so **no local admin account is ever
created on a VM**), `postgresql_entra_admin`, `mysql_entra_admin`, `mssql_entra_admin`. The last
two replace the single Entra-admin slot and restore the previous admin on revoke.

---

## 3. Files in this pack

| Korean | English | Custom agent | Purpose |
|---|---|---|---|
| `01-lab-orchestrator.md` | `01-lab-orchestrator.en.md` | `lab_diagnostics_orchestrator` | Entry point; discovers targets, fans out, aggregates |
| `02-windows-os-expert.md` | `02-windows-os-expert.en.md` | `windows_os_expert` | `diagnose_windows` |
| `03-linux-os-expert.md` | `03-linux-os-expert.en.md` | `linux_os_expert` | `diagnose_linux` |
| `04-sqlserver-expert.md` | `04-sqlserver-expert.en.md` | `sqlserver_expert` | `diagnose_mssql` (IaaS + Azure SQL) |
| `05-mysql-expert.md` | `05-mysql-expert.en.md` | `mysql_expert` | `diagnose_mysql` (IaaS + Flexible Server) |
| `06-postgresql-expert.md` | `06-postgresql-expert.en.md` | `postgresql_expert` | `diagnose_postgres` |
| `07-service-map-expert.md` | `07-service-map-expert.en.md` | `service_map_expert` | `diagnose_service_map` |
| `08-optional-agents.md` | `08-optional-agents.en.md` | 6 agents | AKS / ADX / Event Hubs / App Gateway / Web App / HANA |
| `09-shared-conventions.md` | `09-shared-conventions.en.md` | — | Text blocks reused by every agent |
| `10-privileged-ops-expert.md` | `10-privileged-ops-expert.en.md` | `privileged_ops_expert` | JIT grant/revoke, leak detection, dumps, audit ledger |

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
See `09-shared-conventions.en.md` sections F and G.

**Two rules that exist because they were violated in practice.** Section I forbids an agent from
creating any permission by any route, including `az rest` and `az role assignment create`, even
when an approval prompt appears. Section J forbids rebuilding a failed diagnosis out of `az` CLI
output and presenting it as the diagnosis.

---

## 4. Recommended build order

1. Create the **domain experts** first (02 → 07). They have no handoff dependencies.
2. Create `privileged_ops_expert` (10) if `enablePrivilegedOps=true` was deployed.
3. Create `lab_diagnostics_orchestrator` (01) last and set its **Handoff Agents** to all experts.
4. In each agent, attach only the MCP tool(s) listed in its file — one tool per expert keeps the
   tool-selection decision trivial and avoids cross-domain hallucination.
5. Also attach a read-only **Azure Resource Graph / Azure CLI** built-in tool to every expert.
   The MCP tools require ARM resource IDs and workspace GUIDs that only Resource Graph can supply.
6. Run each agent once in **Builder → Agent Canvas → Test playground** with the sample prompt
   included at the bottom of each file.

---

## 5. Run mode and tool policy

- All 12 `diagnose_*` tools are **strictly read-only**. Set them to **allow** in tool access policies.
- Privileged tools belong to `privileged_ops_expert` only. Do not attach them to domain experts.
- Keep response plans / scheduled tasks in **Review** mode while validating the lab.
- If you attach `azure_cli` to any agent, the instruction blocks forbid write verbs
  (`create`, `delete`, `update`, `set`, `restart`, `scale`, `start`, `stop`) **and** any form of
  permission granting.

---

## 6. Prerequisites the agents cannot fix themselves

| Tool | Prerequisite | Where to fix |
|---|---|---|
| `diagnose_postgres` | MCP managed identity registered as Entra admin in PostgreSQL + `GRANT pg_monitor` | Flexible Server → Authentication |
| `diagnose_mssql` (Azure SQL) | MCP managed identity created as a contained DB user with VIEW DATABASE STATE / VIEW SERVER STATE | `CREATE USER [<mi-name>] FROM EXTERNAL PROVIDER` |
| `diagnose_mssql` (all) | **Microsoft ODBC Driver 18 present in the MCP image** — only this tool uses a native driver | rebuild image with `msodbcsql18`; verify with `odbcinst -q -d` |
| `diagnose_mssql` (IaaS, `auth_mode="sql"`) | `MSSQL_DIAGNOSE_PASSWORD` env var injected into the Container App | ACA → Containers → Environment variables / Key Vault ref |
| `diagnose_mysql` (`auth_mode="mysql"`) | `MYSQL_DIAGNOSE_PASSWORD` env var injected into the Container App | same as above |
| `diagnose_windows` / `diagnose_linux` | AMA + DCR association active; allow 10–15 min after deploy for first data | `<prefix>-dcr-windows` / `<prefix>-dcr-linux` |
| `diagnose_service_map` | Dependency Agent installed (`00_deploy.ps1 -EnableDependencyAgent`) and 15–60 min of traffic | redeploy lab with the switch |
| All | MCP managed identity holds `Reader` + `Monitoring Reader` on the target scope | `infra/assign-roles.ps1`, or `ensure_diagnostic_access` at run time |

---

## 7. Optional: YAML authoring

If you use the SRE Agent MCP server extension for VS Code, each agent file also contains a YAML
block matching the documented schema (`name`, `system_prompt`, `handoff_description`, `tools`).
The portal form fields map as: Name → `name`, Instructions → `system_prompt`,
Handoff Description → `handoff_description`, Custom Tools/Built-in Tools → `tools`.
