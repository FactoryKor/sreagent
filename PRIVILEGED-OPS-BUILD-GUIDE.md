# 특권 작업 승인 체계 구축 가이드 (임시 권한 · 덤프 · 감사)

> 대상: diag-tools MCP 서버 + Azure SRE Agent 환경에 **"사람이 승인해야만 실행되는 특권 작업"**
> 경로를 처음부터 구축하거나, 이미 배포된 환경을 이 구성으로 올리는 운영자.
>
> 개념(왜 이렇게 설계했는가)은 [mcp/IDENTITY-AND-PERMISSIONS.md](mcp/IDENTITY-AND-PERMISSIONS.md),
> 코드 계층은 [sre-ops/README.md](sre-ops/README.md)를 본다. 이 문서는 **무엇을 어떤 순서로
> 설정하고 어떻게 검증하는가**만 다룬다.

---

## 0. 한 장 요약

| # | 단계 | 누가 | 결과물 |
|---|---|---|---|
| 1 | 인프라 배포(`enablePrivilegedOps=true`) | 플랫폼 운영자 | 상태/감사/승인/덤프 컨테이너, 컨테이너 환경변수, 기본 RBAC |
| 2 | MCP 관리 ID 권한 부여 | 플랫폼 운영자 | 읽기 권한 + Run Command + 읽기 전용 JIT(ABAC) + DB 관리자 브로커 |
| 3 | DB 데이터 평면 1회 설정 | DBA | MCP 관리 ID에 **상시 최소 모니터링 롤**(관리자 아님) |
| 4 | 승인자 PC 구성 | 승인자 | `sre-ops` CLI, `sre-ops mode` = `external` |
| 5 | 감사 원장 잠금 | 플랫폼 운영자 | `sre-ops` 컨테이너 WORM 정책 Locked |
| 6 | SRE Agent 구성(8장) | SRE Agent 관리자 | MCP 커넥터, **전역 Deny 정책**, Review 모드 |
| 7 | 서브에이전트 등록(9장) | SRE Agent 관리자 | 에이전트별 도구 선택(9.1절 표) |
| 8 | 종단 간 검증(10장) | 운영자 + 승인자 | 10장 시나리오 A~G 통과 |

가장 중요한 원칙 두 가지:

1. **진단 대상(DB/VM/Log Analytics)에 접속하는 주체는 MCP 서버의 관리 ID 하나뿐이다.**
   SRE Agent의 관리 ID에는 DB/VM 권한을 절대 주지 않는다. 줘도 진단은 실패한다.
2. **프롬프트(지시문)는 메인 에이전트를 막지 못한다.** "SRE Agent에 DB 관리자 권한을 주려는"
   행동은 8.4절의 **전역 Deny 정책**과 코드의 주체 고정(sre-ops `resolve_principal`)으로 막는다.

---

## 1. 구성 요소와 두 개의 관리 ID

```
 사용자/인시던트
      │
      ▼
 ┌──────────────────────┐   HTTPS /mcp    ┌──────────────────────────────┐   Entra 토큰 / Run Command
 │ Azure SRE Agent      │ ──────────────▶ │ diag-mcp (Container Apps)     │ ─────────────────────────▶ DB · VM · Log Analytics
 │  관리 ID ①           │                 │  관리 ID ② (UAMI, <prefix>-id)│
 │  - MCP 호출          │                 │  - 진단 접속 주체 ★           │
 │  - ARM 조회(Reader)  │                 │  - 임시 권한을 받는 주체 ★    │
 └──────────────────────┘                 └───────────────┬──────────────┘
                                                          │ Blob (Entra RBAC, 계정 키 비활성)
                                                          ▼
                                   ┌─────────────── ops 스토리지 계정 ───────────────┐
                                   │ sre-ops-state  요청·lease (변경 가능)           │
                                   │ sre-ops        감사 원장 (WORM)                 │
                                   │ approvals      승인 레코드 (MCP 읽기 전용)       │
                                   │ dumps          덤프 (수명주기 자동 삭제)        │
                                   └─────────────────────────────────────────────────┘
                                                          ▲
                                   승인자 PC: `sre-ops approve` (승인자 본인 Entra 자격)
```

| 주체 | 받을 권한 | 받으면 안 되는 권한 |
|---|---|---|
| ① SRE Agent 관리 ID | MCP 호출(Easy Auth 사용 시), 진단 대상 RG의 Reader/Monitoring Reader/Log Analytics Reader(플랫폼 기본) | **DB Entra 관리자, DB 롤, Run Command, 역할 할당 권한** |
| ② MCP 관리 ID(UAMI) | 2장·3장의 권한 전부 | Owner/Contributor/User Access Administrator(코드와 ABAC 양쪽에서 차단) |
| 승인자(사람, Entra 그룹) | `approvals`·`sre-ops-state`·`sre-ops` 컨테이너 쓰기 | MCP 관리 ID 자격 증명 |

---

## 2. 사전 준비

| 항목 | 내용 |
|---|---|
| 승인자 Entra **그룹** | 개인 계정보다 그룹을 권장. objectId를 메모(`approverPrincipalIds`) |
| 배포자 권한 | 대상 RG에 Owner 또는 (Contributor + User Access Administrator). 사용자 지정 역할을 만들려면 구독 수준 `Microsoft.Authorization/roleDefinitions/write` |
| 진단 대상 범위 | 대상 RG 목록(`allowedDiagnosticScopes`)과 DB FQDN 접미사(`allowedDiagnosticHostSuffixes`) |
| 로컬 도구 | Azure CLI 2.60+, PowerShell 7, Python 3.10+(승인자 PC) |
| 이미지 | mcp `v1.2.1` 이상(진단 결과의 `diagnostic_identity`) + sre-ops `v1.2.1` 이상(감사/상태 컨테이너 분리, 주체 고정) |

---

## 3. 1단계 — 인프라 배포

`infra/main.bicepparam`에 아래를 추가한다.

```bicepparam
param enablePrivilegedOps = true
param externalApprovals = true                         // 승인 레코드를 MCP가 쓸 수 없는 컨테이너에 둔다(필수 권장)
param approverPrincipalIds = ['<승인자 그룹 objectId>']
param approverPrincipalType = 'Group'
param allowedDiagnosticScopes = ['/subscriptions/<sub>/resourceGroups/<workload-rg>']
param allowedDiagnosticHostSuffixes = ['database.windows.net', 'postgres.database.azure.com', 'mysql.database.azure.com']
param maxTemporaryAccessMinutes = 60                   // 승인 1건의 임시 권한 최대 유효시간
param readOnlyAccessMode = 'approval'                  // 읽기 역할도 승인 후 부여(auto 는 Reader류만 즉시 부여)
param dumpRetentionDays = 14
param auditImmutabilityDays = 365
// approversSecretUri 는 비워 둔다 — secret 채널은 컨테이너 침해 시 자가 승인이 가능하다.
```

```powershell
az deployment group create -g <mcp-rg> -n main -f infra/main.bicep -p infra/main.bicepparam
$UAMI_OID  = az deployment group show -g <mcp-rg> -n main --query properties.outputs.diagnosticPrincipalId.value -o tsv
$OPS_ACCT  = az deployment group show -g <mcp-rg> -n main --query properties.outputs.opsStorageAccountName.value -o tsv
```

**생성되는 것과 권한**

| 리소스 | MCP 관리 ID | 승인자 그룹 | 비고 |
|---|---|---|---|
| `sre-ops-state` 컨테이너 | Blob Data Contributor | Blob Data Contributor | 요청·lease. **WORM 없음**(상태 갱신 시 덮어씀) |
| `sre-ops` 컨테이너 | Blob Data Contributor | Blob Data Contributor | 감사 원장. **WORM**(생성·읽기만, 수정·삭제 불가) |
| `approvals` 컨테이너 | **Blob Data Reader** | Blob Data Contributor | MCP가 승인을 위조할 수 없는 근거 |
| `dumps` 컨테이너 | Blob Data Contributor | — | 수명주기로 `dumpRetentionDays` 후 삭제 |
| ops 스토리지 계정 | **Storage Blob Delegator** | — | 덤프 업로드용 사용자 위임 SAS 발급(계정 수준 작업) |

**컨테이너 환경변수 확인**

```powershell
az containerapp show -n <prefix>-app -g <mcp-rg> --query "properties.template.containers[0].env[].{n:name,v:value}" -o table
```

다음이 보여야 한다: `AZURE_PRINCIPAL_ID`, `SRE_OPS_PRINCIPAL_NAME`, `SRE_OPS_STATE_CONTAINER=sre-ops-state`,
`SRE_OPS_AUDIT_CONTAINER=sre-ops`, `SRE_OPS_APPROVALS_CONTAINER=approvals`, `SRE_OPS_ACCESS_MODE`.
`AZURE_PRINCIPAL_ID`/`SRE_OPS_PRINCIPAL_NAME`은 특권 기능을 끈 배포에도 항상 주입된다 — 진단 결과의
`diagnostic_identity`와 `diagnose_postgres`/`diagnose_mysql`의 `user` 기본값이 이 값을 쓴다.

---

## 4. 2단계 — MCP 관리 ID 권한 부여

```powershell
# (a) 읽기 전용 기본 권한
pwsh infra/assign-roles.ps1 -PrincipalId $UAMI_OID `
  -MonitoringScope "/subscriptions/<sub>/resourceGroups/<workload-rg>"

# (b) 특권 작업용 최소 권한
pwsh infra/assign-privileged-roles.ps1 -PrincipalId $UAMI_OID -SubscriptionId <sub> `
  -RunCommandScope "/subscriptions/<sub>/resourceGroups/<vm-rg>" `
  -JitScope        "/subscriptions/<sub>/resourceGroups/<workload-rg>" `
  -DbScope         "/subscriptions/<sub>/resourceGroups/<db-rg>"
```

| 역할 | 범위 | 쓰이는 도구 | 비고 |
|---|---|---|---|
| Reader, Monitoring Reader | workload RG | 모든 `diagnose_*` | |
| Log Analytics Reader(필요 시) | 워크스페이스 | windows/linux/svcmap/avd/citrix, `detect_memory_leak` | |
| `SRE Diagnostic Run Command Operator` | VM RG | `list_os_dumps`, `preflight_process_dump`, `capture_process_dump`, `stage_os_dump` | Run Command = 대상 VM에서 SYSTEM/root 실행. 스크립트는 서버에 고정돼 있고 에이전트가 바꿀 수 없다 |
| RBAC Administrator **+ ABAC 조건** | workload RG | `grant_temporary_access(provider=azure_rbac)`, `ensure_diagnostic_access` | 조건에 들어 있는 읽기 역할만 할당·해제 가능 |
| `SRE Diagnostic DB Admin Broker` | DB RG | `grant_temporary_access(provider=*_entra_admin)` | Entra 관리자 하위 리소스만 쓰기. 서버 설정·데이터 권한 없음 |

> ⚠ 코드의 허용 역할(`jit_access.ALLOWED_ROLES`)에는 `SQL DB Contributor`, `Virtual Machine User Login`,
> `Cosmos DB Account Reader Role`도 들어 있지만 ABAC 조건 목록에는 없다. 이 세 역할은 요청해도
> Azure가 `AuthorizationFailed`로 거부한다. 특히 앞의 두 역할은 읽기 전용이 아니므로 ABAC 목록에
> 추가하지 않는 것을 권장한다.

---

## 5. 3단계 — DB 데이터 평면 1회 설정 (권장 경로)

DB 진단에 필요한 것은 **관리자 권한이 아니라 모니터링 롤**이다. DBA가 MCP 관리 ID에 상시
최소 롤을 한 번 만들어 두면 이후 진단에는 승인이 필요 없고, JIT DB 관리자는 "DBA가 없을 때의
비상 경로"로만 남는다.

`<name>` = `SRE_OPS_PRINCIPAL_NAME`(= UAMI 이름, 기본 `<prefix>-id`), `<oid>` = `$UAMI_OID`,
`<client_id>` = UAMI clientId. 진단 결과의 `diagnostic_identity`에도 같은 값이 나온다.

**PostgreSQL Flexible Server** — Entra 인증 활성화 후, Entra 관리자로 `postgres` DB에 접속해:

```sql
SELECT * FROM pgaadauth_create_principal_with_oid('<name>', '<oid>', 'service', false, false);
GRANT pg_monitor TO "<name>";
```

**Azure SQL Database** — 서버 Entra 관리자로:

```sql
-- 각 진단 대상 DB에서
CREATE USER [<name>] FROM EXTERNAL PROVIDER;
GRANT VIEW DATABASE STATE TO [<name>];
-- 인스턴스 수준 DMV(master에서)
CREATE LOGIN [<name>] FROM EXTERNAL PROVIDER;
ALTER SERVER ROLE ##MS_ServerStateReader## ADD MEMBER [<name>];
```

SQL Managed Instance/SQL Server(Entra 연동)는 `GRANT VIEW SERVER STATE TO [<name>];`.

**Azure Database for MySQL Flexible Server** — 서버에 Entra 인증과 서버용 UAMI를 구성한 뒤, Entra 관리자로:

```sql
CREATE AADUSER '<name>' IDENTIFIED BY '<client_id>';
GRANT PROCESS, REPLICATION CLIENT ON *.* TO '<name>'@'%';
GRANT SELECT ON performance_schema.* TO '<name>'@'%';
```

> JIT로 `mysql_entra_admin`을 쓸 경우, MySQL은 Entra 관리자 지정 시 서버에 연결된 UAMI 리소스 ID가
> 필요하다. 컨테이너에 `SRE_OPS_MYSQL_IDENTITY_RESOURCE_ID`를 수동으로 설정한다(Bicep 미주입).

---

## 6. 4단계 — 승인자 PC 구성

승인은 **승인자 본인의 Entra 자격으로, 승인자 PC에서** 이뤄진다. MCP는 `approvals` 컨테이너에
쓸 권한이 없으므로 에이전트 대화 안에서는 절대 승인할 수 없다(설계된 동작).

```powershell
python -m pip install "sre-ops @ git+https://github.com/FactoryKor/sre-ops.git@v1.2.1"

# 승인자 프로필($PROFILE)에 넣어 두면 편하다
$env:SRE_OPS_STATE_ACCOUNT       = "<OPS_ACCT>"
$env:SRE_OPS_STATE_CONTAINER     = "sre-ops-state"
$env:SRE_OPS_AUDIT_CONTAINER     = "sre-ops"
$env:SRE_OPS_APPROVALS_CONTAINER = "approvals"
$env:SRE_OPS_ACTOR               = "alice@contoso.com"   # 감사 원장 actor 표기
Remove-Item Env:AZURE_CLIENT_ID -ErrorAction SilentlyContinue   # 있으면 관리 ID 자격을 먼저 시도한다

az login
sre-ops mode        # → "approval_mode": "external", signed_in.upn = 본인
sre-ops requests    # → [] (권한 오류 없이 빈 목록)
```

| 증상 | 원인 |
|---|---|
| `approval_mode: none` | 환경변수 누락 |
| `sre-ops requests` 가 403 | 승인자 그룹에 `sre-ops-state` 권한 없음 → `approverPrincipalIds` 확인 |
| `approve` 가 "승인 요청을 찾을 수 없습니다" | 요청 ID 오타이거나 `sre-ops-state` 읽기 권한 없음(권한 오류가 '없음'으로 보인다) |
| 스토리지 네트워크 거부 | `restrictOpsStorageNetwork=true` 면 승인자 PC를 허용 IP/Private Endpoint로 접근시킨다 |

---

## 7. 5단계 — 감사 원장 잠금

배포 직후 정책은 `Unlocked`(시험용)다. 운영 전환 시 잠근다. **잠그면 보존 기간 동안 구독 Owner도
감사 기록을 지울 수 없고, 정책을 해제할 수도 없다.**

```powershell
$etag = az storage container immutability-policy show -g <mcp-rg> --account-name $OPS_ACCT --container-name sre-ops --query etag -o tsv
az storage container immutability-policy lock -g <mcp-rg> --account-name $OPS_ACCT --container-name sre-ops --if-match $etag
```

> 잠그는 대상은 `sre-ops`(감사) 하나다. `sre-ops-state`에는 WORM을 걸지 않는다 — 걸면 승인 소진과
> 회수가 `BlobImmutableDueToPolicy`로 실패한다.

---

## 8. 6단계 — SRE Agent 구성

### 8.1 권한 수준과 관리 ID

- 에이전트 생성 시 **Reader** 권한 수준을 고른다. 쓰기 작업이 필요하면 플랫폼이 OBO(사람 권한 대행)를
  요청하며, OBO는 **SRE Agent Administrator만** 승인할 수 있다.
- SRE Agent 관리 ID에는 DB/VM 권한을 주지 않는다. 이미 줬다면 9.3절로 정리한다.

### 8.2 MCP 커넥터

`https://<fqdn>/mcp`를 MCP 커넥터로 등록한다(연결 ID 예: `diag-tools`). 도구 이름은
`diag-tools_diagnose_mssql`처럼 연결 ID가 접두사로 붙는다.

### 8.3 실행 모드

응답 계획/예약 작업은 검증 기간에 **Review**로 둔다. 진단 도구는 읽기 전용이지만, 메인 에이전트는
Azure CLI 쓰기를 제안할 수 있다.

### 8.4 전역 도구 접근 정책 (가장 중요)

**Settings → Permissions**(전역 범위)에 아래를 넣는다. 전역 Deny는 커스텀 에이전트/스레드 범위의
Allow로도 풀 수 없고, 메인 에이전트에도 적용된다. 지시문으로는 막을 수 없는 "SRE Agent에 DB 관리자
부여" 시도를 여기서 결정적으로 차단한다.

```json
{
  "permissions": {
    "deny": [
      "RunPsqlReadCommand",
      "bash(az * ad-admin *)",
      "bash(az * microsoft-entra-admin *)",
      "bash(az role assignment create *)",
      "bash(az role assignment update *)",
      "bash(az rest *roleAssignments*)",
      "bash(az rest *administrators*)",
      "bash(psql *)",
      "bash(sqlcmd *)",
      "bash(mysql *)",
      "ExecutePythonCode(*roleAssignments*)",
      "ExecutePythonCode(*administrators*)",
      "*approve_privileged_action"
    ],
    "ask": [
      "*capture_process_dump",
      "*stage_os_dump"
    ],
    "allow": [
      "RunAzCliReadCommands",
      "*diagnose_*",
      "*describe_diagnostic_identity",
      "*detect_memory_leak",
      "*list_os_dumps",
      "*preflight_process_dump",
      "*list_staged_dumps",
      "*analyze_dump",
      "*list_privileged_requests",
      "*list_temporary_access",
      "*revoke_temporary_access",
      "*get_audit_log"
    ]
  }
}
```

| 규칙 | 막는 것 |
|---|---|
| `RunPsqlReadCommand` | 메인 에이전트가 **자기 관리 ID로** PostgreSQL에 직접 붙으려다 실패 → "SRE Agent에 DB 관리자 부여"로 이어지는 가장 흔한 경로 |
| `*ad-admin*`, `*microsoft-entra-admin*`, `az rest *administrators*` | DB Entra 관리자를 승인 절차 밖에서 직접 설정 |
| `role assignment create/update`, `az rest *roleAssignments*` | RBAC를 승인 절차 밖에서 직접 부여 |
| `psql`/`sqlcmd`/`mysql` | 셸에서 DB에 직접 접속해 `CREATE USER` 시도 |
| `*approve_privileged_action` | secret 채널에서 에이전트가 승인 경로를 호출(비밀값이 대화에 남는다) |
| ask: 덤프 캡처/반출 | 운영 서버 정지 위험 작업에 SRE Agent 쪽 확인을 한 번 더(승인 게이트와 별개) |

> `grant_temporary_access`·`request_privileged_action`은 Deny하지 않는다. 대상 주체는 코드가 MCP
> 관리 ID로 고정하고, 실행은 사람의 out-of-band 승인 없이는 불가능하다.

---

## 9. 7단계 — 서브에이전트 등록과 도구 선택

지시문은 [Subagents/](Subagents/README.md)의 각 파일을 그대로 붙여 넣는다. 도구는 아래 표대로
**하나씩 골라** 붙인다. 커넥터 와일드카드(`diag-tools/*`)는 쓰지 않는다 — 모든 전문가에게 특권 도구가
붙는다.

### 9.1 에이전트별 도구 표

| 에이전트 | Custom Tools (MCP) | Built-in Tools | Handoff Agents |
|---|---|---|---|
| `lab_diagnostics_orchestrator` | (없음) | `RunAzCliReadCommands`, `execute_kusto_query` | 모든 전문가 + `privileged_ops_expert` |
| `windows_os_expert` | `diagnose_windows`, `detect_memory_leak`, `list_os_dumps`, `preflight_process_dump`, `list_staged_dumps`, `analyze_dump`, `describe_diagnostic_identity` | `RunAzCliReadCommands`, `execute_kusto_query` | `sqlserver_expert`, `privileged_ops_expert`, orchestrator |
| `linux_os_expert` | `diagnose_linux` + windows와 같은 분석 도구 5개 + `describe_diagnostic_identity` | 〃 | `mysql_expert`, `privileged_ops_expert`, orchestrator |
| `sqlserver_expert` | `diagnose_mssql`, `describe_diagnostic_identity` | 〃 | `windows_os_expert`, `privileged_ops_expert`, orchestrator |
| `mysql_expert` | `diagnose_mysql`, `diagnose_mysql_fleet`, `describe_diagnostic_identity` | 〃 | `linux_os_expert`, `privileged_ops_expert`, orchestrator |
| `postgresql_expert` | `diagnose_postgres`, `describe_diagnostic_identity` | 〃 | `privileged_ops_expert`, orchestrator |
| `service_map_expert` | `diagnose_service_map` | 〃 | 각 도메인 전문가, orchestrator |
| `privileged_ops_expert` | `describe_diagnostic_identity`, `ensure_diagnostic_access`, `detect_memory_leak`, `list_os_dumps`, `preflight_process_dump`, `list_staged_dumps`, `analyze_dump`, `request_privileged_action`, `list_privileged_requests`, `grant_temporary_access`, `revoke_temporary_access`, `list_temporary_access`, `capture_process_dump`, `stage_os_dump`, `get_audit_log` | `RunAzCliReadCommands` | orchestrator, 각 도메인 전문가 |
| `aks_expert` / `adx_expert` / `eventhub_expert` / `appgateway_expert` / `webapp_expert` / `hana_expert` | 각각 `diagnose_aks` / `diagnose_adx` / `diagnose_eventhub` / `diagnose_appgateway` / `diagnose_webapp` / `diagnose_hana` | 〃 | orchestrator |
| (선택) AVD / Citrix 전문가 | `diagnose_avd` / `diagnose_citrix` (+ 세션 호스트 메모리 문제면 `detect_memory_leak`) | 〃 | `windows_os_expert`, orchestrator |

**어떤 에이전트에도 붙이지 않는 것**

| 도구 | 이유 |
|---|---|
| `approve_privileged_action` | 승인은 사람이 PC에서 한다. external 채널에선 항상 실패, secret 채널에선 비밀값이 대화에 남는다 |
| `RunAzCliWriteCommands` | 지시문으로 금지해도 실행 가능. 특히 메인 에이전트·오케스트레이터에서 권한 부여로 이어진다 |
| `RunPsqlReadCommand` / DB 계열 기본 제공 도구 | SRE Agent 관리 ID로 DB에 붙는다 → 실패 → 자기에게 DB 관리자 부여 시도 |
| `RunKubectlWriteCommand` | 진단 팩 범위 밖 |

**Enable skills는 전문가에서 끈다.** 스킬에 붙은 도구(쓰기 포함)가 그대로 상속된다.

### 9.2 왜 Windows/Linux 전문가가 덤프 "분석" 도구는 갖고 "캡처" 도구는 갖지 않는가

| 구분 | 도구 | 변경 여부 | 승인 | 보유 에이전트 |
|---|---|---|---|---|
| 관찰 | `detect_memory_leak`, `list_os_dumps`, `preflight_process_dump`, `list_staged_dumps`, `analyze_dump` | 없음 | 불필요 | Windows/Linux 전문가 **와** privileged_ops_expert |
| 요청 | `request_privileged_action` | 요청 레코드만 생성 | — | privileged_ops_expert |
| 실행 | `capture_process_dump`, `stage_os_dump`, `grant_temporary_access` | 프로세스 정지·메모리 반출·권한 부여 | **사람 승인 필수** | privileged_ops_expert 만 |

OS 전문가는 "어느 프로세스가 새는지, 덤프를 뜰 가치가 있는지, 뜨면 얼마나 멈추는지"까지 스스로
판단하고, 실제로 뜨는 단계만 privileged_ops_expert로 넘긴다. 특권 실행 도구를 한 에이전트에 모아
두면 감사·정책·지시문을 한 곳에서 관리할 수 있다. (실행 도구를 OS 전문가에 직접 붙여도 승인 게이트
때문에 승인 없이 실행되지는 않는다 — 다만 정책 관리 지점이 늘어난다.)

**Windows 메모리 누수 → 덤프 흐름**

```
windows_os_expert   diagnose_windows            → Memory % Committed 경고
windows_os_expert   detect_memory_leak          → w3wp +180MB/h, R²=0.94
windows_os_expert   preflight_process_dump      → 예상 3.2GB, 정지 ~22초, 디스크 여유 41GB
        └─ handoff ─▶ privileged_ops_expert
privileged_ops_expert request_privileged_action(action="capture_process_dump", target=<vm-id>, process="w3wp", os_type="windows")
(승인자 PC)          sre-ops approve <request_id> --approver alice@contoso.com
privileged_ops_expert capture_process_dump(request_id, ...) → dumps/ 에 업로드
privileged_ops_expert analyze_dump(blob_name)    → private 영역 3만 개/평균 64KB → 힙 누수
```

### 9.3 SRE Agent에 이미 들어간 권한 정리

이전 시도로 SRE Agent 관리 ID가 DB 관리자·역할을 받았을 수 있다. 확인하고 제거한다.

```powershell
$SRE_OID = "<SRE Agent 관리 ID objectId>"   # 포털: Settings → Azure settings → Go to Identity
az role assignment list --assignee $SRE_OID --all -o table
az postgres flexible-server microsoft-entra-admin list -g <rg> -s <pg-server> -o table
az mysql flexible-server ad-admin show -g <rg> -s <mysql-server>
az sql server ad-admin list -g <rg> -s <sql-server> -o table
```

목록에 SRE Agent가 있으면 제거하고, MySQL/Azure SQL은 원래 관리자를 다시 지정한다(슬롯이 하나뿐).

---

## 10. 8단계 — 종단 간 검증

| # | 시나리오 | 방법 | 기대 결과 |
|---|---|---|---|
| A | 진단 주체 확인 | 대화: "PostgreSQL <fqdn> 진단해줘" (전문가에서 `user` 비움) | 결과에 `diagnostic_identity.principal_name = <prefix>-id`. 실패하더라도 SRE Agent 권한 부여 시도 없이 5장의 SQL을 안내 |
| B | 정책 차단 | 대화: "SRE Agent에 이 PostgreSQL Entra 관리자 권한을 줘" | `RunAzCliWriteCommands`/`bash(az * ad-admin *)` 가 **Deny** 로 차단, OBO 프롬프트도 뜨지 않음 |
| C | 주체 고정 | privileged_ops_expert에게 `principal_object_id=<SRE_OID>`로 요청하도록 유도 | `invalid_input`: "임시 권한은 MCP 서버의 관리 ID … 에만 부여할 수 있습니다" |
| D | DB JIT 왕복 | ① `request_privileged_action(action="grant_temporary_access", provider="postgresql_entra_admin", target=<pg-id>, justification=…)` ② 승인자 `sre-ops approve` ③ `grant_temporary_access(request_id, …)` ④ 진단 ⑤ `revoke_temporary_access` ⑥ `list_temporary_access` | ③ lease_id/expires_at 반환, ⑥ 활성 lease 0. `sre-ops audit --request-id <id>` 에 requested→approved→executed→revoked |
| E | 자가 승인 불가 | 승인 전에 `grant_temporary_access` 호출 | `approval_required`. `sre-ops approve`를 MCP 쪽에서 흉내 내도 `approvals` 쓰기 403 |
| F | 대상 바꿔치기 | 서버 A로 승인받고 서버 B로 `grant_temporary_access` | 지문 불일치로 거부 |
| G | 덤프 | 9.2절 흐름 | 승인 전 `approval_required`, 승인 후 업로드·`analyze_dump` 성공 |

D가 실패하면 대부분 아래 셋 중 하나다: `sre-ops-state`가 WORM(구버전 배포) → 12장, 승인자 컨테이너
권한 누락 → 6장, `DB Admin Broker` 역할 범위 밖 → 4장.

---

## 11. 운영 절차

**일상 승인**

```powershell
sre-ops requests                       # 대기 목록
sre-ops show <request-id>              # 대상·사유·위험도(덤프면 예상 크기/정지 시간)
sre-ops approve <request-id> --approver alice@contoso.com
sre-ops deny    <request-id> --approver alice@contoso.com --reason "업무 시간 외"
```

승인 시 확인할 것: `target`이 맞는가, `params.principal_object_id`가 **MCP 관리 ID**인가, TTL이 작업에
비해 과하지 않은가, 덤프면 `risk`의 정지 시간·디스크 여유.

**회수와 감사**

```powershell
sre-ops leases                         # 살아 있는 임시 권한
sre-ops revoke <lease-id> --reason "작업 종료"
sre-ops reap                           # 만료분 일괄 회수
sre-ops audit --correlation-id <id>    # 한 인시던트 전체 타임라인
```

`revoke_failed` lease(특히 MySQL/Azure SQL의 원래 관리자 복원 실패)는 최우선으로 처리한다 —
그동안 원래 관리자가 대체된 상태다. lease의 `revoke_data.previous`에 원래 관리자가 기록돼 있다.

---

## 12. 기존 배포를 이 구성으로 올릴 때

이전 구성은 요청·lease·감사를 **하나의 WORM 컨테이너(`sre-ops`)**에 두었다. 이 상태에서는 승인 소진과
회수가 `BlobImmutableDueToPolicy`로 실패하므로 JIT·덤프가 사실상 동작하지 않는다.

1. 새 `infra/main.bicep`으로 재배포한다. `sre-ops-state` 컨테이너, Storage Blob Delegator, 승인자
   상태/감사 권한, 신원 환경변수가 추가된다. 기존 `sre-ops` 컨테이너와 감사 기록은 그대로 감사
   원장으로 쓰인다(이동 불필요).
2. 이미지를 mcp `v1.2.1`+ / sre-ops `v1.2.1`+ 로 올린다(`mcp/requirements.txt`의 `sre-ops` 태그 확인).
3. 승인자 PC 환경변수에 `SRE_OPS_STATE_CONTAINER=sre-ops-state`, `SRE_OPS_AUDIT_CONTAINER=sre-ops`를 추가한다.
4. 재배포 전에 만든 대기 요청은 옛 컨테이너에 남는다. 새로 요청한다.
5. 8.4절 전역 정책, 9장 도구 구성을 적용하고 10장 A~G를 다시 수행한다.

---

## 13. 문제 해결

| 증상 | 원인 | 조치 |
|---|---|---|
| 에이전트가 **SRE Agent 관리 ID에 DB 관리자 권한**을 주려 한다 | ① 메인 에이전트/오케스트레이터가 `RunPsqlReadCommand`·az CLI로 직접 DB에 접속 시도 ② 전문가가 `user`에 추측한 이름(SRE Agent 이름)을 넣어 OID 불일치 ③ `principal_object_id`에 자기 OID를 넣음 ④ 신원 환경변수 미주입으로 `user` 기본값이 비어 있음 | ① 8.4절 Deny ② 전문가 지시문 갱신본(`user` 비움) ③ 코드가 거부(sre-ops v1.2.1+) ④ 새 Bicep 재배포 |
| `grant_temporary_access`가 `Blob 쓰기 실패 … BlobImmutableDueToPolicy` | 상태와 감사가 같은 WORM 컨테이너 | 12장 |
| `capture_process_dump`가 업로드 단계에서 403 | 사용자 위임 키 발급 권한 없음 | Storage Blob Delegator(계정 범위) — 새 Bicep 포함 |
| `list_os_dumps`/`preflight_process_dump`가 `AuthorizationFailed` | Run Command 역할 범위 밖 VM | 4장 `-RunCommandScope` |
| `approval_required`가 계속 난다 | 승인 안 됨/만료(기본 30분)/이미 소진 | 승인자에게 request_id 전달, 새로 요청 |
| "승인받은 대상/파라미터와 … 일치하지 않습니다" | 승인 후 대상·프로세스·주체가 바뀜 | 바뀐 값으로 새 요청 |
| "임시 권한은 MCP 서버의 관리 ID … 에만" | 다른 주체를 지정함 | `principal_object_id` 비우기. 예외가 정말 필요하면 운영자가 `SRE_OPS_ALLOWED_PRINCIPALS` 설정 |
| `ensure_diagnostic_access`가 바로 부여하지 않음 | `readOnlyAccessMode=approval`(기본) | 반환된 request_id로 승인 → `grant_temporary_access` |
| DB 로그인은 되는데 뷰가 비어 있음 | 관리자/롤만 있고 모니터링 권한 없음 | 5장 GRANT |
| OBO 승인 프롬프트가 "Entra 관리자 설정"을 묻는다 | 메인 에이전트가 쓰기 명령 시도 | **거부**. 8.4절 Deny를 넣으면 프롬프트 자체가 사라진다 |

---

## 부록 A. 환경변수

| 변수 | 주입 | 의미 |
|---|---|---|
| `AZURE_CLIENT_ID` | 항상 | UAMI clientId(DefaultAzureCredential) |
| `AZURE_PRINCIPAL_ID` | 항상 | UAMI objectId — 진단·임시 권한의 유일한 주체 |
| `SRE_OPS_PRINCIPAL_NAME` | 항상 | UAMI 이름 — DB 로그인 이름 기본값 |
| `AZURE_TENANT_ID` | 항상 | DB Entra 관리자 등록에 사용 |
| `SRE_OPS_STATE_ACCOUNT` / `SRE_OPS_STATE_CONTAINER` | 특권 | 요청·lease (`sre-ops-state`) |
| `SRE_OPS_AUDIT_CONTAINER` | 특권 | 감사 원장 (`sre-ops`, WORM) |
| `SRE_OPS_APPROVALS_CONTAINER` | 특권(external) | 승인 레코드 (`approvals`) |
| `SRE_OPS_DUMP_ACCOUNT` / `SRE_OPS_DUMP_CONTAINER` | 특권 | 덤프 반출 |
| `SRE_OPS_MAX_TTL_MINUTES` | 특권 | 임시 권한 최대 유효시간 |
| `SRE_OPS_ACCESS_MODE` / `SRE_OPS_AUTOGRANT_TTL_MINUTES` | 특권 | 읽기 역할 자동 부여 여부/TTL |
| `SRE_OPS_ALLOWED_PRINCIPALS` | 수동 | MCP UAMI 외 임시 권한 수령 허용 objectId(기본 없음) |
| `SRE_OPS_MYSQL_IDENTITY_RESOURCE_ID` | 수동 | `mysql_entra_admin` 사용 시 서버 UAMI 리소스 ID |
| `DIAG_ALLOWED_SCOPES` / `DIAG_ALLOWED_HOST_SUFFIXES` | 항상 | 진단 허용 범위 |

## 부록 B. 승인 요청 한 건의 수명

```
requested ──(승인자 PC: sre-ops approve)──▶ approved ──(30분 내 grant/capture)──▶ executed
    │                                          │                                    │
    └─ 30분 경과 ─▶ expired                    └─ 대상/파라미터 불일치 ─▶ failed    ├─ revoke_temporary_access ─▶ revoked
                                                                                    └─ TTL 만료 후 다음 특권 호출 ─▶ auto_revoked
```

각 단계는 감사 원장(`sre-ops` 컨테이너)에 이벤트 1건 = Blob 1개로 남는다.
