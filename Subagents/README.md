# Azure SRE Agent — Custom Agent (Subagent) Instruction Pack

Instruction text for the custom agents ("subagents") that make the **diag-tools MCP server**
usable against the **Total-Lab / full-lab** environment.

Everything in this folder is written in **English** and is meant to be pasted directly into
**Azure portal → your SRE Agent → Builder → Agent Canvas → Create → Custom Agent**.

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
Agents for those tools are still provided (see `08-optional-agents.md`) so the same MCP
connector can be reused when you extend the lab.

---

## 2. MCP tools actually registered on the server

Verified against `FactoryKor/mcp` → `mcp_server.py` (the image running in Azure Container Apps).
Use these **exact** names when you select Custom Tools for each agent.

| Tool | Required args | Optional args |
|---|---|---|
| `diagnose_postgres` | `host`, `user` | `dbname`, `resource_id`, `hours` |
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

> The older `README.md` in `Install File/mcp/` documents `diagnose_windows_os` / `diagnose_linux_os`
> with a `source` argument. **That is stale.** The deployed server registers `diagnose_windows` /
> `diagnose_linux` with no `source` argument — Azure Monitor mode only.

---

## 3. Files in this pack

| File | Custom agent | Purpose |
|---|---|---|
| `01-lab-orchestrator.md` | `lab_diagnostics_orchestrator` | Entry point; discovers targets, fans out, aggregates |
| `02-windows-os-expert.md` | `windows_os_expert` | `diagnose_windows` |
| `03-linux-os-expert.md` | `linux_os_expert` | `diagnose_linux` |
| `04-sqlserver-expert.md` | `sqlserver_expert` | `diagnose_mssql` (IaaS + Azure SQL) |
| `05-mysql-expert.md` | `mysql_expert` | `diagnose_mysql` (IaaS + Flexible Server) |
| `06-postgresql-expert.md` | `postgresql_expert` | `diagnose_postgres` |
| `07-service-map-expert.md` | `service_map_expert` | `diagnose_service_map` |
| `08-optional-agents.md` | 6 agents | AKS / ADX / Event Hubs / App Gateway / Web App / HANA |
| `09-shared-conventions.md` | — | Text blocks reused by every agent (output contract, guardrails) |

---

## 4. Recommended build order

1. Create the **domain experts** first (02 → 07). They have no handoff dependencies.
2. Create `lab_diagnostics_orchestrator` (01) last and set its **Handoff Agents** to all experts.
3. In each agent, attach only the MCP tool(s) listed in its file — one tool per expert keeps the
   tool-selection decision trivial and avoids cross-domain hallucination.
4. Also attach a read-only **Azure Resource Graph / Azure CLI** built-in tool to every expert.
   The MCP tools require ARM resource IDs and workspace GUIDs that only Resource Graph can supply.
5. Run each agent once in **Builder → Agent Canvas → Test playground** with the sample prompt
   included at the bottom of each file.

---

## 5. Run mode and tool policy

- All 12 MCP tools are **strictly read-only**. Set them to **allow** in tool access policies.
- Keep response plans / scheduled tasks in **Review** mode while validating the lab. The experts
  never mutate anything, but the orchestrator may propose Azure CLI mitigations.
- If you attach `azure_cli` to any agent, the instruction blocks already forbid write verbs
  (`create`, `delete`, `update`, `set`, `restart`, `scale`, `start`, `stop`).

---

## 6. Prerequisites the agents cannot fix themselves

The instruction blocks tell each agent to report these as blockers instead of looping:

| Tool | Prerequisite | Where to fix |
|---|---|---|
| `diagnose_postgres` | MCP managed identity registered as Entra role in PostgreSQL + `GRANT pg_monitor` | Flexible Server → Authentication |
| `diagnose_mssql` (Azure SQL) | MCP managed identity created as a contained DB user with VIEW DATABASE STATE / VIEW SERVER STATE | `CREATE USER [<mi-name>] FROM EXTERNAL PROVIDER` |
| `diagnose_mssql` (IaaS, `auth_mode="sql"`) | `MSSQL_DIAGNOSE_PASSWORD` env var injected into the Container App | ACA → Containers → Environment variables / Key Vault ref |
| `diagnose_mysql` (`auth_mode="mysql"`) | `MYSQL_DIAGNOSE_PASSWORD` env var injected into the Container App | same as above |
| `diagnose_windows` / `diagnose_linux` | AMA + DCR association active; allow 10–15 min after deploy for first data | `<prefix>-dcr-windows` / `<prefix>-dcr-linux` |
| `diagnose_service_map` | Dependency Agent installed (`00_deploy.ps1 -EnableDependencyAgent`) and 15–60 min of traffic | redeploy lab with the switch |
| All | MCP managed identity holds `Reader` + `Monitoring Reader` on the lab resource group | `infra/assign-roles.ps1` |

---

## 7. Optional: YAML authoring

If you use the SRE Agent MCP server extension for VS Code, each agent file also contains a YAML
block matching the documented schema (`name`, `system_prompt`, `handoff_description`, `tools`).
The portal form fields map as: Name → `name`, Instructions → `system_prompt`,
Handoff Description → `handoff_description`, Custom Tools/Built-in Tools → `tools`.
