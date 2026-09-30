# Azure 진단 도구 제품군 — 단독 실행(Standalone) 가이드

이 리포(Install File) 아래의 진단 도구(`pg` / `aks` / `adx` / `eh` / `agw` / `svcmap` / `windows` / `linux` / `mssql` / `mysql` / `avd` / `citrix` 등)는
**모두 독립 실행 가능한 CLI**입니다. MCP 서버(`mcp/mcp_server.py`)나 Azure SRE Agent는 이 CLI들을 감싸서
호출하는 실행 편의 계층일 뿐이며, 진단 로직·리포트 생성 자체는 각 스크립트 안에서 완결됩니다.

즉 **MCP/SRE Agent가 전혀 없어도** 터미널에서 직접 실행해 table/JSON/HTML 리포트를 생성할 수 있습니다.

---

## 왜 단독 실행이 되는가

| 요소 | 내용 |
|---|---|
| 인증 | 각 도구가 자체적으로 `azure.identity.DefaultAzureCredential()`을 사용 — `az login`(또는 관리 ID)만 되어 있으면 동작 |
| 데이터 소스 | Azure Monitor 메트릭 / Log Analytics KQL / ARM 조회를 스크립트가 직접 호출(중간에 MCP 필요 없음) |
| 출력 | `--format table`(터미널) / `json`(SRE Agent용 스키마지만 사람이 봐도 무방) / `html -o file.html`(브라우저로 열기) |
| 데모 모드 | `--demo`로 Azure 자격 증명·네트워크 연결 없이도 리포트 미리보기 가능 |
| 리포트 게시(옵션) | `REPORT_STORAGE_ACCOUNT` 환경변수를 설정하면 로컬 실행에서도 Blob 정적 웹사이트에 자동 게시됨 |

MCP는 이 CLI를 `subprocess.run([..., "--format", "json"])`으로 호출해 SRE Agent가 소비하기 좋은 형태로
감싸주는 역할만 합니다. `--workspace-id`/`--resource-id`처럼 도구가 스스로 알아낼 수 없는 값은 사람이
직접 넘겨야 하는데, MCP 경유 시에는 Agent가 `needs_input` 힌트를 보고 자동으로 재호출해주는 차이만 있습니다.

---

## 설치 (도구별 공통)

각 도구 폴더에서:

```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Windows 콘솔에서 한글이 깨지면 실행 전 `$env:PYTHONIOENCODING="utf-8"` 을 설정하세요.

---

## 도구별 단독 실행 예시

| 도구 | 대상 | 예시 명령 |
|---|---|---|
| `pg_diagnose` | PostgreSQL Flexible Server | `python pg_diagnose.py --host <fqdn> --user <role> --aad --format table` |
| `aks_diagnose` | AKS 클러스터 | `python aks_diagnose.py --namespace default --format html -o aks.html` |
| `adx_diagnose` | Azure Data Explorer(Kusto) | `python adx_diagnose.py --cluster https://<cluster>.kusto.windows.net --auth default --format table` |
| `eh_diagnose` | Event Hubs | `python eh_diagnose.py --resource-id <namespace RID> --azure-auth --eh-auth entra --format table` |
| `agw_diagnose` | Application Gateway | `python agw_diagnose.py --resource-id <agw RID> --azure-auth --format table` |
| `svcmap_diagnose` | 워크로드 서비스 맵 | `python svcmap_diagnose.py --appinsights-id <RID> --format html -o svcmap.html` |
| `windows_diagnose` | Windows 서버(VM/Arc) | `python windows_diagnose.py --computer WIN-APP01 --workspace-id <guid> --format table` |
| `linux_diagnose` | Linux 서버(VM/Arc) | `python linux_diagnose.py --computer linux-app01 --workspace-id <guid> --format table` |
| `mssql_diagnose` | SQL Server(온프레미스/IaaS/Azure SQL DB/MI) | `python mssql_diagnose.py --host sqlsrv01.corp.local --auth-mode sql --user diag_reader --format table` |
--auth-mode mysql --user diag_reader --format table` |
| `avd_diagnose` | Azure Virtual Desktop 호스트 풀(레벨 400) | `python avd_diagnose.py --host-pool-id <hostPool RID> --workspace-id <guid> --format table` |
| `citrix_diagnose` | Citrix DaaS/CVAD on Azure VDI(레벨 400) | `python citrix_diagnose.py --subscription <sub> --resource-group <rg> --vda-prefix <prefix> --connector-prefix <prefix> --workspace-id <guid> --citrix-customer-id <id> --citrix-client-id <guid> --format table` (비밀은 `CITRIX_CLIENT_SECRET` 환경변수) |

`mssql-diagnose`, `mysql-diagnose`, `avd-diagnose`, `citrix-diagnose` …).

> [!TIP]
> `windows_diagnose`/`linux_diagnose`는 **기본적으로 Azure Arc 연결을 전제**로 설계되었습니다 —
> 온프레미스/AWS/GCP 서버도 [Azure Arc](https://learn.microsoft.com/azure/azure-arc/servers/overview)로
> 온보딩하면 Azure VM과 동일하게 Log Analytics 원격측정을 쓸 수 있습니다(`--resource-id`에
> `Microsoft.HybridCompute/machines` 리소스 ID 지정 가능). Arc를 연결할 수 없는 예외적인 경우를 위해
> **숨은 옵션**(`--source direct --host <ip> --winrm-user <user>`/`--ssh-user <user>`)도 내부적으로 지원합니다 —
> `--help`에는 나오지 않지만 명시하면 그대로 동작합니다. 비밀번호는 CLI 인자로 받지 않고 환경변수
> (`WINDOWS_DIAGNOSE_WINRM_PASSWORD`/`LINUX_DIAGNOSE_SSH_PASSWORD`) 또는 SSH 키로 전달합니다.
> 자세한 내용은 각 도구의 README.md를 참고하세요.

### 자격 증명 없이 미리보기 (`--demo`)

```bash
python windows_diagnose.py --demo --format html -o demo.html
python linux_diagnose.py --demo --format table
python svcmap_diagnose.py --demo --format json
```

모든 도구가 `--demo` 옵션으로 합성 데이터를 사용해 Azure 호출 없이 리포트 형태를 확인할 수 있습니다.

---

## 출력 형식

- `--format table` (기본값 도구도 있음) — 사람이 읽기 쉬운 텍스트 리포트, 터미널에 바로 출력
- `--format json` — `health_score` / `severity_counts` / `summary` / `recommended_actions` / `needs_input` 포함,
  SRE Agent·자동화 파이프라인 연동용이지만 그대로 읽어도 무방
- `--format html -o <path>` — 요약 카드 + 진단 표(일부 도구는 그래프 포함) HTML 파일 생성, 브라우저로 열람
- `--exit-code` — CI/스케줄 작업용 종료 코드(critical=2, warning=1, 그 외=0)

---

## 자율 발견 → 재호출 (`needs_input`)

`--workspace-id`, `--resource-id`, `--appinsights-id` 등 도구가 스스로 유도할 수 없는 값을 생략하면,
JSON 출력의 최상위 `needs_input`에 어떤 값이 왜 필요한지와 Resource Graph로 찾는 방법(`discovery_hint`),
재호출 명령 예시(`reinvoke_example`)가 채워집니다. 단독 실행 시에도 이 힌트를 참고해 값을 채운 뒤
다시 실행하면 됩니다.

---

## 조회 기간 선택 — "1일치", "1주일치" 등 원하는 기간 지정

모든 도구가 상대 기간 파라미터를 지원합니다(도구마다 이름은 조금씩 다름: `--hours`, `--window-minutes` 등).
예를 들어 다음처럼 바로 "1일치"/"1주일치"를 지정할 수 있습니다.

```bash
pg-diagnose --host <fqdn> --user <role> --aad --hours 24            # 1일치
windows-diagnose --computer WIN-APP01 --workspace-id <guid> --hours 168   # 1주일치
eh-diagnose --resource-id <RID> --azure-auth --window-minutes 60    # 최근 1시간
```

`windows_diagnose`/`linux_diagnose`는 여기에 더해 **절대 날짜범위**도 지원합니다(azure-monitor 모드 전용 —
"지금부터 N시간 전"이 아니라 "특정 날짜 구간"을 직접 지정):

```bash
# 지난주 특정 3일 구간만 조회 (2026-07-28 00:00 ~ 2026-07-31 00:00 UTC)
windows-diagnose --computer WIN-APP01 --workspace-id <guid> \
  --start-time 2026-07-28T00:00:00Z --end-time 2026-07-31T00:00:00Z --format html -o last_week.html
```

**주의**: 이 절대 날짜범위는 Log Analytics에 이미 수집된 원격측정(azure-monitor 모드)을 조회할 때만
적용됩니다. `--source direct`(숨은 옵션, SSH/WinRM)는 대상 서버의 **현재 시점** 상태만 조회할 수 있으므로
과거 날짜를 가져올 수 없습니다(온도계처럼 "지금 측정"만 가능한 것과 같은 이치) — `--start-time`/`--end-time`을
direct 모드와 함께 쓰면 명확한 오류를 반환합니다.

### direct 모드에서 과거 데이터가 필요하다면 — 직접 이력 축적(`--save-snapshot`)

"미리 특정 경로에 성능데이터를 계속 저장하게 하면 가능하지 않을까?" — 맞습니다. `windows_diagnose`/
`linux_diagnose`에는 **`--save-snapshot <경로>`**가 있어, 실행할 때마다 결과 JSON을 그 경로에
JSON Lines로 이어붙입니다. 이를 cron/작업 스케줄러로 주기 실행(예: 매시간)하면, direct 모드처럼
원래 이력을 저장하지 않는 수집 방식도 **직접 시계열 이력을 만들** 수 있습니다.

```bash
# 매시간 실행되도록 등록 → 스스로 이력을 쌓음
windows-diagnose --source direct --host WIN-ONPREM01 --winrm-user diag_reader \
  --format json --save-snapshot C:\diag-history\win-onprem01.jsonl
```

---

## 리포트를 원하는 언어로 볼 수 있나?

**MCP/Azure SRE Agent를 통해 사용하는 경우(권장 경로)** — 별도 설정 없이 바로 가능합니다. 도구가
반환하는 결과는 구조화된 JSON이고, 이를 사용자에게 보여주는 것은 SRE Agent(LLM)입니다 — "영어로 보여줘",
"일본어로 요약해줘"처럼 요청하면 **코드 변경 없이 그 자리에서 번역된 결과**를 보여줍니다. 어떤 언어든
제약 없이 지원되는 가장 빠르고 유연한 방법입니다.

**CLI로 단독 실행해 생성한 리포트 파일 자체**(HTML/table)를 다른 언어로 보고 싶다면:

- 현재 버전은 진단 항목의 제목/상세/권장조치 텍스트가 한국어로 고정되어 있습니다(`category`/`severity` 같은
  JSON 키/값은 언어와 무관하게 항상 영문 코드로 고정되어 있어 MCP 연동에는 영향이 없습니다).
- 전체를 다국어화(`--lang ko|en|...`)하려면 10개 도구 각각의 수십 개 진단 메시지를 이중화(또는 메시지
  카탈로그로 분리)하는 작업이 필요해 상당한 작업량이 듭니다. 원하시면 어떤 도구부터, 어떤 언어로
  우선 작업할지 정해서 별도로 진행할 수 있습니다.

---

## 정기 실행(스케줄) — MCP/Agent 없이도 가능

`cron`/작업 스케줄러/ACA Job 등으로 다음처럼 주기 실행해 히스토리를 쌓을 수 있습니다.

```bash
windows-diagnose --computer WIN-APP01 --workspace-id <guid> --format html \
  -o "reports/windows-$(date +%Y%m%d-%H%M).html"
```

`REPORT_STORAGE_ACCOUNT` 환경변수를 설정해두면 실행할 때마다 Blob 정적 웹사이트(`index.html`)에
자동으로 게시되어 과거 리포트 목록을 웹에서 확인할 수 있습니다(`report-publish` 모듈, opt-in).
`windows_diagnose`/`linux_diagnose`의 `--save-snapshot`(위 절 참고)과 함께 쓰면, 원격 웹 게시(요약)와
로컬 JSON Lines 이력(상세 원본) 두 가지를 동시에 확보할 수 있습니다.

---

## 요약

이 10개 도구는 **각자 완전한 CLI 프로그램**입니다. MCP 서버·SRE Agent는 선택 사항이며,
Azure 로그인만 되어 있으면(또는 `--demo`로 로그인 없이도) 터미널에서 바로 실행해
table/JSON/HTML 리포트를 생성할 수 있습니다.
