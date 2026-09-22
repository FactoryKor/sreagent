**한국어** | [English](README.en.md)

# Azure SRE Agent — 커스텀 에이전트(서브에이전트) 지시문 팩

Azure SRE Agent에서 **diag-tools MCP 서버**를 실제로 쓸 수 있게 만드는 커스텀 에이전트
("서브에이전트")의 지시문 모음입니다.

모든 파일은 두 언어로 있습니다. `<이름>.md`가 한국어, `<이름>.en.md`가 영어이며,
각 파일 첫 줄에서 서로 오갈 수 있습니다.

> **포털에는 어느 쪽을 붙여넣나요?**
> **영문판**을 붙여넣으십시오. 지시문은 모델이 읽는 텍스트이고, 영어로 두어야 도구 이름과
> severity 값, JSON 키가 흔들리지 않습니다. 한국어판은 읽고 검토하고 유지보수하기 위한
> 번역본입니다. 출력 언어는 별개 문제입니다 — 모든 에이전트는 사용자가 쓴 언어로 답하므로,
> 지시문이 영어여도 한국어로 물으면 한국어로 답합니다.

---

## 1. 랩이 실제로 배포하는 것

출처: `Total-Lab/full-lab/main.bicep` + `modules/*.bicep`. 기본 `namePrefix`는 `dtlab`입니다.

| 랩 리소스 | 리소스 이름 패턴 | Log Analytics `Computer` / DB | 대응 MCP 도구 |
|---|---|---|---|
| Windows Server 2022 + SQL Server 2022 Dev VM | `<prefix>-win-sql` | `win-sql` | `diagnose_windows`, `diagnose_mssql` |
| Ubuntu VM + MySQL Server | `<prefix>-linux-mysql` | `linux-mysql` | `diagnose_linux`, `diagnose_mysql` |
| Azure SQL Database (PaaS) | `<prefix>-sqlsvr-<hash>` / db `diagdb` | — | `diagnose_mssql` |
| Azure DB for MySQL Flexible Server | `<prefix>-mysql-<hash>` / db `diagdb` | — | `diagnose_mysql` |
| Azure DB for PostgreSQL Flexible Server | `<prefix>-pg-<hash>` / db `diagdb` | — | `diagnose_postgres` |
| Log Analytics 작업 영역 | `<prefix>-law` | — | (windows/linux/svcmap 입력) |
| Application Insights | `<prefix>-appi` | — | (svcmap 입력) |
| DCR (Windows Perf/Event, Linux Perf/Syslog, VM Insights) | `<prefix>-dcr-*` | — | — |
| Dependency Agent (**기본 꺼짐**) | — | — | `diagnose_service_map` |

이 랩이 배포하지 않는 것: AKS, Event Hubs, Application Gateway, App Service, ADX, SAP HANA.
해당 도구용 에이전트도 함께 제공하므로(`08-optional-agents.md` 참조), 랩을 확장할 때 같은
MCP 커넥터를 그대로 재사용할 수 있습니다.

---

## 2. 서버에 등록된 MCP 도구

`FactoryKor/mcp`의 `mcp_server.py`(Azure Container Apps에서 도는 이미지) 기준으로 확인했습니다.
각 에이전트의 Custom Tools를 고를 때 **이 이름 그대로** 쓰십시오.

| 도구 | 필수 인자 | 선택 인자 |
|---|---|---|
| `diagnose_postgres` | `host` | `user`, `dbname`, `resource_id`, `hours` |
| `diagnose_mssql` | `host` | `user`, `database`, `auth_mode`, `resource_id`, `region`, `hours`, `password_env` |
| `diagnose_mysql` | `host` | `user`, `database`, `auth_mode`, `resource_id`, `region`, `hours`, `password_env` |
| `diagnose_windows` | `computer` | `workspace_id`, `resource_id`, `hours` |
| `diagnose_linux` | `computer` | `workspace_id`, `resource_id`, `hours` |
| `diagnose_service_map` | `appinsights_id` / `workspace_id` 중 하나 | `workload`, `window_minutes` |
| `diagnose_aks` | — | `namespace`, `context`, `all_namespaces`, `prometheus_url`, `appinsights_id` |
| `diagnose_adx` | `cluster` | `database`, `resource_id`, `region`, `hours` |
| `diagnose_eventhub` | `resource_id` | `event_hub`, `region`, `window_minutes`, `checkpoint_store` |
| `diagnose_appgateway` | `resource_id` | `region`, `window_minutes`, `backend_health` |
| `diagnose_webapp` | `resource_id` | `region`, `window_minutes` |
| `diagnose_hana` | `host`+`user` 또는 `userkey` | `port`, `deployment_type`, `hours`, `password_env` |

> `diagnose_postgres`는 이제 `user`가 필수가 아닙니다. 비우면 서버가 컨테이너 자신의 관리 ID
> 이름(`DIAG_DB_USER` → `SRE_OPS_PRINCIPAL_NAME`)으로 채웁니다. OID 불일치 같은 인증 실패는
> 거의 항상 권한 부족이 아니라 주체 이름이 틀린 것입니다.

### v1.3+ 특권·심층 진단 도구 (`privileged_ops_expert` 전용)

서버에는 총 28개 도구가 등록돼 있습니다. 위 12개 `diagnose_*` 외에는 `sre-ops` 계층에서
옵니다. **승인**으로 표시된 도구는 관리자가 별도 경로(`sre-ops approve`)로 승인하기 전까지
`approval_required`로 거부됩니다.

| 도구 | 하는 일 | 승인 |
|---|---|---|
| `describe_diagnostic_identity` | 어느 관리 ID에 어떤 권한이 필요한지 알려줌 | — |
| `detect_memory_leak` | 시계열 회귀로 누수 프로세스 특정 | — |
| `list_os_dumps` | 기존 덤프와 덤프 설정 상태 조사 | — |
| `preflight_process_dump` | 덤프 크기·정지 시간·디스크 여유 사전 평가 | — |
| `list_staged_dumps` / `analyze_dump` | 반출된 덤프 목록 / 구조적 분석 | — |
| `request_privileged_action` | 승인 요청 생성(변경 없음) | — |
| `list_privileged_requests` | 승인 대기 목록 | — |
| `ensure_diagnostic_access` | 읽기 전용 권한이 없으면 확보 — `auto` 모드면 즉시 부여, 아니면 승인 요청 생성 | 모드에 따름 |
| `grant_temporary_access` | 시간 제한 권한 부여 (TTL 기본 60분, `maxTemporaryAccessMinutes` 상한) | **승인** |
| `capture_process_dump` / `stage_os_dump` | 메모리 덤프 생성 / 반출 | **승인** |
| `revoke_temporary_access` | 즉시 회수 — 승인 불필요 | — |
| `list_temporary_access` | 유효한 lease 조회, 호출할 때마다 만료분 자동 회수 | — |
| `get_audit_log` | append-only 감사 원장 조회 | — |

JIT provider: `azure_rbac`(VM Run Command도 이 경로를 쓰므로 **VM에 로컬 관리자 계정을 절대
만들지 않습니다**), `postgresql_entra_admin`, `mysql_entra_admin`, `mssql_entra_admin`.
뒤 두 개는 Entra 관리자 슬롯이 하나뿐이라 기존 관리자를 대체하고, 회수할 때 복원합니다.

---

## 3. 이 팩의 파일

| 한국어 | 영어 | 커스텀 에이전트 | 용도 |
|---|---|---|---|
| `01-lab-orchestrator.md` | `01-lab-orchestrator.en.md` | `lab_diagnostics_orchestrator` | 진입점. 대상 탐색 → 위임 → 결과 병합 |
| `02-windows-os-expert.md` | `02-windows-os-expert.en.md` | `windows_os_expert` | `diagnose_windows` |
| `03-linux-os-expert.md` | `03-linux-os-expert.en.md` | `linux_os_expert` | `diagnose_linux` |
| `04-sqlserver-expert.md` | `04-sqlserver-expert.en.md` | `sqlserver_expert` | `diagnose_mssql` (IaaS + Azure SQL) |
| `05-mysql-expert.md` | `05-mysql-expert.en.md` | `mysql_expert` | `diagnose_mysql` (IaaS + Flexible Server) |
| `06-postgresql-expert.md` | `06-postgresql-expert.en.md` | `postgresql_expert` | `diagnose_postgres` |
| `07-service-map-expert.md` | `07-service-map-expert.en.md` | `service_map_expert` | `diagnose_service_map` |
| `08-optional-agents.md` | `08-optional-agents.en.md` | 6개 에이전트 | AKS / ADX / Event Hubs / App Gateway / Web App / HANA |
| `09-shared-conventions.md` | `09-shared-conventions.en.md` | — | 모든 에이전트가 공유하는 규약 블록 |
| `10-privileged-ops-expert.md` | `10-privileged-ops-expert.en.md` | `privileged_ops_expert` | 임시 권한 부여·회수, 누수 탐지, 덤프, 감사 원장 |

---

## 3.5 출력 언어와 환경 프로파일

**출력 언어는 이 팩이 고정하지 않습니다.** 모든 에이전트는 사용자의 마지막 메시지 언어를
따라가고, 사용자가 바꿀 때까지 유지합니다. 한국어로 물으면 한국어로, 영어로 물으면 영어로
답합니다. 다만 severity enum, 도구·인자 이름, 메트릭 이름, SQL/KQL, 리소스 ID,
"Not evaluated" 헤딩은 **절대 번역하지 않습니다.** 그래야 오케스트레이터가 여러 전문가의
결과를 병합할 수 있습니다. 고객 제출 문서에는 severity를 인라인으로 번역하는 대신
대조표(critical→위험, warning→주의, info→정보, ok→양호)를 한 번만 내보냅니다.

**환경 프로파일이 중요합니다.** 각 전문가는 Total-Lab 전용 사실 블록을 갖고 있습니다.
공개 엔드포인트, Basic·Burstable SKU, HA 없음, 최소 보존, 합성 트래픽 같은 허용 사항은
**ENVIRONMENT = lab일 때만** 적용됩니다. 고객 운영 환경에서는 같은 발견사항이 원래 심각도를
유지합니다. 프로파일이 모호하면 한 번 묻고, 답이 없으면 production으로 가정합니다.
`09-shared-conventions.md`의 F·G 절을 참고하십시오.

**실제로 위반돼서 생긴 규칙 두 가지.** I 절은 에이전트가 어떤 경로로도 권한을 만들지 못하게
합니다. `az rest`, `az role assignment create`를 포함하며, 승인 프롬프트가 떠도 마찬가지입니다.
J 절은 진단 도구가 실패했을 때 `az` CLI 출력으로 진단을 재구성해 결과처럼 제시하는 것을
금지합니다.

---

## 4. 권장 생성 순서

1. **도메인 전문가**를 먼저 만듭니다(02 → 07). 핸드오프 의존성이 없습니다.
2. `enablePrivilegedOps=true`로 배포했다면 `privileged_ops_expert`(10)를 만듭니다.
3. `lab_diagnostics_orchestrator`(01)를 마지막에 만들고 **Handoff Agents**에 전문가 전부를 지정합니다.
4. 각 에이전트에는 그 파일에 적힌 MCP 도구만 붙입니다. 전문가 하나에 도구 하나면 도구 선택이
   자명해지고 도메인을 넘나드는 환각이 줄어듭니다.
5. 모든 전문가에 읽기 전용 **Azure Resource Graph / Azure CLI** 기본 도구도 붙입니다. MCP 도구는
   ARM 리소스 ID와 작업 영역 GUID를 요구하는데, 그 값은 Resource Graph만 줄 수 있습니다.
6. **Builder → Agent Canvas → Test playground**에서 각 파일 하단의 예시 프롬프트로 한 번씩
   실행해 봅니다.

---

## 5. 실행 모드와 도구 정책

- 12개 `diagnose_*` 도구는 **엄격히 읽기 전용**입니다. 도구 접근 정책에서 **allow**로 두십시오.
- 특권 도구는 `privileged_ops_expert`에만 붙입니다. 도메인 전문가에 붙이지 마십시오.
- 랩을 검증하는 동안 응답 계획과 예약 작업은 **Review** 모드로 유지하십시오.
- 어떤 에이전트든 `azure_cli`를 붙인다면, 지시문 블록이 쓰기 동사(`create`, `delete`, `update`,
  `set`, `restart`, `scale`, `start`, `stop`)와 **모든 형태의 권한 부여**를 금지합니다.

---

## 6. 에이전트가 스스로 해결할 수 없는 전제 조건

| 도구 | 전제 조건 | 조치 위치 |
|---|---|---|
| `diagnose_postgres` | MCP 관리 ID가 PostgreSQL의 Entra 관리자로 등록 + `GRANT pg_monitor` | Flexible Server → Authentication |
| `diagnose_mssql` (Azure SQL) | MCP 관리 ID를 포함 DB 사용자로 만들고 VIEW DATABASE STATE / VIEW SERVER STATE 부여 | `CREATE USER [<mi-name>] FROM EXTERNAL PROVIDER` |
| `diagnose_mssql` (공통) | **MCP 이미지에 Microsoft ODBC Driver 18 포함** — 12종 중 이 도구만 네이티브 드라이버를 씀 | `msodbcsql18` 포함해 재빌드, `odbcinst -q -d`로 확인 |
| `diagnose_mssql` (IaaS, `auth_mode="sql"`) | Container App에 `MSSQL_DIAGNOSE_PASSWORD` 환경변수 주입 | ACA → Containers → 환경 변수 / Key Vault 참조 |
| `diagnose_mysql` (`auth_mode="mysql"`) | Container App에 `MYSQL_DIAGNOSE_PASSWORD` 환경변수 주입 | 위와 동일 |
| `diagnose_windows` / `diagnose_linux` | AMA + DCR 연결 활성. 배포 후 첫 데이터까지 10~15분 | `<prefix>-dcr-windows` / `<prefix>-dcr-linux` |
| `diagnose_service_map` | Dependency Agent 설치(`00_deploy.ps1 -EnableDependencyAgent`) 후 15~60분 트래픽 | 스위치를 켜고 랩 재배포 |
| 전체 | MCP 관리 ID가 대상 범위에 `Reader` + `Monitoring Reader` 보유 | `infra/assign-roles.ps1`, 또는 실행 시 `ensure_diagnostic_access` |

---

## 7. 선택 사항 — YAML 작성

VS Code용 SRE Agent MCP 서버 확장을 쓴다면, 각 에이전트 파일에 문서화된 스키마(`name`,
`system_prompt`, `handoff_description`, `tools`)에 맞는 YAML 블록도 들어 있습니다.
포털 입력란은 이렇게 대응합니다 — Name → `name`, Instructions → `system_prompt`,
Handoff Description → `handoff_description`, Custom Tools/Built-in Tools → `tools`.
