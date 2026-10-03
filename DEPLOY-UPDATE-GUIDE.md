# diag-tools MCP · SRE Agent — 업데이트/배포 운영 가이드

> 대상: **diag-mcp**(Azure Container Apps 상주) + **Azure SRE Agent** 연동 환경을
> 신규 구축하거나, 새 기능/버그픽스를 반영하기 위한 실무 런북.
> 진단기 6종(`pg/adx/eh/agw/svcmap/aks`)은 모두 **읽기 전용**이며, 이 가이드의 어떤 단계도 진단 대상 리소스를 변경하지 않습니다.

---

## 0. 먼저 이해할 것 — 리포 구조와 "소스가 있는 곳 ≠ 실행되는 곳"

### 0-1. 리포 구조 (모노레포 아님 — 컴포넌트별 독립 리포)

`FactoryKor` 조직에 **9개 리포가 각각 분리**되어 있다. 예전의 "Install File 하나에 모든 폴더"(모노레포) 구조가 **아니다**.

| 리포 | 내용 | pip 패키지명 / 콘솔 명령 |
|---|---|---|
| `FactoryKor/pg` | PostgreSQL 진단 | `pg-diagnose` |
| `FactoryKor/adx` | Azure Data Explorer 진단 | `adx-diagnose` |
| `FactoryKor/eh` | Event Hub 진단 | `eh-diagnose` |
| `FactoryKor/agw` | Application Gateway 진단 | `agw-diagnose` |
| `FactoryKor/svcmap` | 서비스 맵 진단 | `svcmap-diagnose` |
| `FactoryKor/aks` | AKS 진단 | `aks-diagnose` |
| `FactoryKor/avd` | Azure Virtual Desktop 진단(레벨 400) | `avd-diagnose` |
| `FactoryKor/citrix` | Citrix on Azure VDI 진단(레벨 400) | `citrix-diagnose` |
| `FactoryKor/report-publish` | HTML 리포트 Blob 게시(공용 라이브러리) | `report-publish` |
| `FactoryKor/mcp` | MCP 서버 + Dockerfile + 배포 워크플로 | (이미지 `diag-mcp`) |
| `FactoryKor/infra` | Bicep IaC + 역할 부여 스크립트 | — |

**결합 방식**: `mcp` 리포의 `requirements.txt`가 나머지 6개 리포를 **버전 태그로 고정해 pip 설치**한다.

```text
pg-diagnose     @ git+https://github.com/FactoryKor/pg.git@v1.0.0
adx-diagnose    @ git+https://github.com/FactoryKor/adx.git@v1.0.0
eh-diagnose     @ git+https://github.com/FactoryKor/eh.git@v1.0.0
agw-diagnose    @ git+https://github.com/FactoryKor/agw.git@v1.0.0
svcmap-diagnose @ git+https://github.com/FactoryKor/svcmap.git@v1.0.0
aks-diagnose    @ git+https://github.com/FactoryKor/aks.git@v1.0.0
```

각 컴포넌트 리포의 `pyproject.toml`이 `[project.scripts]`로 **콘솔 명령**을 등록하고(`pg-diagnose = "pg_diagnose:main"` 등), `dynamic = ["dependencies"]`로 각자의 `requirements.txt`를 그대로 재사용한다. `report-publish`는 6개 도구의 의존성으로 **자동 전이 설치**된다.

> 그래서 `mcp_server.py`는 파일 경로가 아니라 **PATH의 콘솔 명령**을 호출한다: `PG_TOOL = "pg-diagnose"`.

### 0-2. 배포/런타임 흐름

```text
[소스]  컴포넌트 리포(pg/adx/eh/agw/svcmap/aks)
           │  태그 v* push → notify-mcp.yml → repository_dispatch
           ▼
        mcp 리포 (.github/workflows/deploy-mcp.yml)
           │  requirements.txt 버전 핀 자동 갱신 → az acr build
           ▼
[배포]  ACR: diag-mcp:<version> (+ :latest)
           │  az containerapp update → 새 리비전 → 건강 검증 → 실패 시 자동 롤백
           ▼
        Azure Container Apps 에 diag-mcp 상주
           ▲   (User-Assigned Managed Identity 로 진단 대상 읽기 전용 접근)
           │  MCP 호출(/mcp, streamable-http) — 매 요청
[런타임] Azure SRE Agent  ←  사용자/인시던트 트리거
```

핵심 원칙 4가지:

1. **런타임 루프에 GitHub가 없다.** SRE Agent → `/mcp`(ACA) → 진단 실행 → JSON. 코드를 배포할 때만 GitHub가 개입한다.
2. **엔드포인트(`https://<fqdn>/mcp`)는 코드 배포로 바뀌지 않는다.** 대부분의 업데이트에서 SRE Agent는 손댈 필요가 없다.
3. **진단기 코드는 이미지 빌드 시점에 고정된다.** 도구를 고쳤으면 **반드시 재빌드·재배포**해야 실제 동작이 바뀐다(main에 push만 해서는 반영 안 됨).
4. **모든 버전은 태그로 고정**한다. 컴포넌트는 `@v1.0.0`, 이미지는 `diag-mcp:v1.0.0`/`sha-<short>`. 재현성·롤백을 위해 `latest`에만 의존하지 않는다.

---

## 1. 사전 준비 (최초 1회 확인)

### 1-1. 로컬 도구
실행 머신에 아래가 설치돼 있어야 한다.

| 도구 | 확인 명령 | 용도 |
|---|---|---|
| Git | `git --version` | 소스 푸시 |
| GitHub CLI | `gh --version` | 리포/태그/시크릿 관리 |
| Azure CLI | `az version` | 수동 빌드/배포·검증 |
| (선택) Docker | `docker --version` | 로컬 이미지 테스트 |

> `az acr build`는 **클라우드(ACR Tasks)에서 빌드**하므로 로컬 Docker 없이도 이미지 빌드가 된다.

### 1-2. 배포에 필요한 값 (미리 메모)

| 이름 | 예시/설명 | 확인 방법 |
|---|---|---|
| `ACR_NAME` | 이미지 레지스트리 이름 | `az acr list -o table` |
| `ACA_NAME` | Container App 이름(`diag-mcp`) | `az containerapp list -o table` |
| `ACA_RG` | ACA 리소스 그룹 | 위 명령 출력 |
| `MCP_ENDPOINT` | `https://<fqdn>/mcp` | Bicep 출력 `mcpEndpoint` 또는 아래 명령 |
| `UAMI_PRINCIPAL_ID` | 진단 자격(관리 ID) principalId | Bicep 출력 `identityPrincipalId` |

```powershell
# 엔드포인트(FQDN) 확인
$fqdn = az containerapp show -n <ACA_NAME> -g <ACA_RG> --query properties.configuration.ingress.fqdn -o tsv
"MCP endpoint: https://$fqdn/mcp"
```

### 1-3. GitHub 시크릿/변수 (CI/CD용, 최초 1회)

**`mcp` 리포** → `Settings → Secrets and variables → Actions`:

| 시크릿 | 용도 |
|---|---|
| `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_SUBSCRIPTION_ID` | OIDC 로그인용 Entra 앱(연합 자격) |
| `ACR_NAME` | 이미지 빌드/보관 ACR |
| `ACA_NAME` / `ACA_RESOURCE_GROUP` | 배포 대상 Container App |
| `COMPONENT_REPOS_PAT` | **컴포넌트 리포가 private일 때만** 필요(read-only fine-grained PAT). 현재 9개 리포는 모두 **Public**이므로 비워두어도 된다 |

**각 컴포넌트 리포**(pg/adx/eh/agw/svcmap/aks) → `notify-mcp.yml`이 쓰는 값:

| 이름 | 종류 | 값 |
|---|---|---|
| `MCP_DISPATCH_PAT` | Secret | `mcp` 리포에 `repository_dispatch` 권한이 있는 PAT |
| `MCP_REPO` | Variable | `FactoryKor/mcp` |

> 이 둘을 설정하지 않으면 컴포넌트 태그 push 시 자동 재배포만 안 될 뿐, 수동 배포(2-3)로는 문제없이 동작한다.

---

## 1-A. 신규 구축 (전부 지우고 처음부터 만드는 경우)

기존 환경 업데이트가 아니라 **처음 만드는** 경우는 이 순서를 따른다.

```powershell
az login
az account set --subscription <SUB_ID>
$RG = "RG-SRELAB"
az group create -n $RG -l koreacentral

# 1) 인프라 배포 (ACR + UAMI + Container Apps 환경/앱)
#    첫 배포 시점엔 ACR에 이미지가 없으므로 placeholder 이미지로 생성된다.
cd infra
az deployment group create -g $RG -f main.bicep -p main.bicepparam

# 2) 출력값 보관
az deployment group show -g $RG -n main --query properties.outputs -o json
#    mcpEndpoint / identityPrincipalId / identityClientId / reportStorageAccountName

# 3) 진단 대상에 읽기 전용 권한 부여
./assign-roles.ps1 -PrincipalId <identityPrincipalId> `
  -MonitoringScope "/subscriptions/<sub>/resourceGroups/<진단대상RG>" `
  -EventHubNamespaceId "<eh-namespace-resource-id>"
#    ADX/PostgreSQL 데이터 평면 권한은 스크립트 출력 안내대로 각 서비스 내부에서 부여

# 4) 실제 이미지 빌드 + 배포 → 2-3(수동 배포) 참고

# 5) SRE Agent에 MCP 커넥터 등록 → 3-2 참고
```

주요 Bicep 파라미터(`infra/main.bicep`):

| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `acrName` | (필수) | ACR 이름 |
| `imageTag` | `latest` | 배포할 이미지 태그 |
| `externalIngress` | `true` | `false`면 내부 전용(Private Endpoint와 조합) |
| `enableEntraAuth` / `entraAuthClientId` / `entraAuthTenantId` | `false` / `''` / 현재 테넌트 | ACA Easy Auth로 `/mcp` 401 보호 |
| `allowedIpRanges` | `[]` | 값을 넣으면 ingress IP 허용목록 적용 |
| `enableReportPublish` / `reportStorageAccountName` | `false` / `''` | HTML 리포트 Blob 게시용 스토리지 생성 + 환경변수 자동 주입 |
| `minReplicas` / `maxReplicas` | `1` / `2` | 스케일 범위 |

> 기본값은 모두 **기존 동작 유지**(인증 없음·IP 무제한·리포트 게시 꺼짐)다. 운영 전환 시에만 `main.bicepparam`에서 값을 채운다.

---

## 2. 시나리오 A — 배포된 MCP 서버에 새 기능 반영

먼저 **어느 리포를 고쳤는지**에 따라 경로가 갈린다.

| 고친 대상 | 필요한 절차 |
|---|---|
| 진단 도구(pg/adx/eh/agw/svcmap/aks) | 2-1 → 해당 리포에 태그 push → mcp 자동 재빌드 |
| MCP 서버 자체(`mcp_server.py`, Dockerfile) | 2-2 → mcp 리포에 push |
| 인프라(Bicep) | `az deployment group create` 재실행(1-A의 1단계) |

### 2-1. (권장) 진단 도구 수정 — 태그 릴리스 체인

컴포넌트 리포에 `v*` 태그를 push하면 **자동으로 끝까지 흘러간다**.

```powershell
# 예: pg 도구 수정
cd <pg 리포 클론>
git switch main
git pull
git add pg_diagnose.py
git commit -m "fix: bloat 판정 임계값 조정"
git push origin main

git tag -a v1.1.0 -m "v1.1.0 - bloat 임계값 조정"
git push origin v1.1.0
```

태그 push 직후 자동 수행되는 일:

1. `pg` 리포의 **`notify-mcp.yml`** → `mcp` 리포로 `repository_dispatch`(`component-released`, payload = `{component: pg, version: v1.1.0}`).
2. `mcp` 리포의 **`deploy-mcp.yml`**:
   - `requirements.txt`의 `.../pg.git@v1.0.0` → `@v1.1.0` 로 **자동 치환 후 커밋·푸시**
   - `az acr build` (빌드 컨텍스트 = **mcp 리포 루트**, `--file Dockerfile`)
   - `az containerapp update --image ...:<version>` → 새 리비전
   - **리비전 건강 검증**(최대 5분 폴링) → 실패 시 **이전 이미지로 자동 롤백**

> 여러 컴포넌트를 동시에 릴리스해도 `concurrency: deploy-mcp` 로 순차 처리된다.

**여러 도구를 한꺼번에 올릴 때**: 각 리포에 태그를 push하면 빌드가 여러 번 돈다. 한 번에 끝내려면 각 리포에 태그만 push해 두고(자동 배포는 실패해도 무방), 마지막에 `mcp` 리포에서 `requirements.txt`의 버전을 한꺼번에 수정해 push한다.

### 2-2. MCP 서버 자체 수정

```powershell
cd <mcp 리포 클론>
git add mcp_server.py Dockerfile requirements.txt
git commit -m "feat: diagnose_cosmos 도구 추가"
git push origin main            # → sha-<short> 태그로 빌드/배포

# 릴리스 버전으로 고정하려면
git tag -a v1.1.0 -m "v1.1.0"
git push origin v1.1.0          # → diag-mcp:v1.1.0 (롤백 지점 확보)
```

### 2-3. 수동 배포 폴백 (Actions 없이 손으로)

CI가 막혔거나 긴급 배포일 때, 실행 머신에서 직접:

```powershell
cd <mcp 리포 클론>              # ← 빌드 컨텍스트는 mcp 리포 루트 하나
az login
az account set --subscription <SUB_ID>

$VER = "v1.1.0"
# 1) 빌드 + 푸시 (버전 + latest)
#    컴포넌트 리포가 Public이면 GH_PAT 불필요.
az acr build --registry <ACR_NAME> `
  --image "diag-mcp:$VER" --image "diag-mcp:latest" `
  --file Dockerfile .

#    (컴포넌트 리포가 private인 경우에만)
#    az acr build ... --secret-build-arg GH_PAT=<token> .
#    ※ 일반 --build-arg 는 이미지 히스토리에 토큰이 남으므로 쓰지 말 것

# 2) ACA 이미지 교체
$acrLogin = az acr show -n <ACR_NAME> --query loginServer -o tsv
az containerapp update -n <ACA_NAME> -g <ACA_RG> `
  --image "$acrLogin/diag-mcp:$VER"
```

> 도구만 고치고 태그를 올리지 않았다면 `requirements.txt`의 버전 핀이 그대로라 **옛 코드가 다시 설치된다**. 수동 배포 전에 `requirements.txt`의 `@vX.Y.Z`가 의도한 태그인지 반드시 확인한다.

### 2-4. 배포 전 로컬 스모크 테스트 (선택, 권장)

```powershell
$env:PYTHONIOENCODING = "utf-8"

# 1) 진단기 단독 (데모 데이터) — 각 도구 리포 클론 루트에서 실행
python pg_diagnose.py     --demo --format json | python -m json.tool
python adx_diagnose.py    --demo --format json | python -m json.tool
python agw_diagnose.py    --demo --format json | python -m json.tool
python svcmap_diagnose.py --demo --format json | python -m json.tool
python aks_diagnose.py    --demo --format json | python -m json.tool
# eh 는 --demo 가 없으므로 실제 네임스페이스로 확인

# 2) 패키지 결합 검증 — mcp 리포에서 (실제 설치 없이 해석만)
pip install --dry-run -r requirements.txt

# 3) MCP 서버 기동 → /mcp, /health 확인
pip install -r requirements.txt
python mcp_server.py            # streamable-http, 포트 8000
```

**JSON 계약 확인**: 모든 도구의 `--format json` 출력에 아래 키가 있어야 SRE Agent가 제대로 해석한다.

`tool` · `version` · `health_score` · `worst_severity` · `severity_counts` · `summary` · `recommended_actions` · `needs_input` · `findings`(또는 `checks`)

---

## 3. 시나리오 B — SRE Agent 업데이트

SRE Agent는 **엔드포인트만 가리키고, 도구 스키마는 MCP `tools/list`로 캐시**한다. 그래서 변경 종류에 따라 필요한 조치가 다르다.

| 변경 내용 | SRE Agent 조치 | 이유 |
|---|---|---|
| 도구 **내부 동작**만 변경 (버그픽스, 한글 라벨, 완결화, health_score 계산 등) | **없음** | 같은 도구를 같은 인자로 호출 → 새 동작이 자동 반영 |
| 도구 **스키마 변경** (파라미터 추가/삭제/이름변경, 새 도구, 설명 변경) | **커넥터 도구 목록 새로고침** (재동기화 또는 제거 후 재등록) | 에이전트가 캐시한 tool manifest를 갱신해야 새 인자를 인식 |
| **엔드포인트 URL 변경** (ACA 재생성, external↔internal 전환 등) | 커넥터의 MCP 엔드포인트 주소 수정 | 가리키는 대상 자체가 바뀜 |
| **새 리소스 유형**을 다루는 도구 추가 | UAMI에 read 역할 부여(`infra/assign-roles.ps1`) | 관리 ID가 새 대상에 접근 권한 필요 |

### B-1. 스키마가 바뀐 경우 — 커넥터 재동기화

MCP 도구의 **파라미터가 추가/변경**되면 SRE Agent가 캐시한 manifest를 갱신해야 새 인자를 인식한다.

1. Azure Portal → **Azure SRE Agent** → 해당 에이전트 → **Tools / MCP 커넥터** 설정.
2. 등록된 diag-mcp 커넥터에서 **Refresh / Re-sync**(또는 커넥터 제거 후 `MCP_ENDPOINT`로 재등록).
3. 도구 목록에 변경된 파라미터가 보이는지 확인.

> 기존 인자만 쓰는 호출은 재동기화 없이도 계속 동작한다. 새 파라미터 활용을 원할 때만 재동기화가 필요.

**참고 — 자율 발견 루프**: 진단 도구는 스스로 알아낼 수 없는 값(예: agw의 `resource_id`, eh의 `checkpoint_store`, svcmap의 `appinsights_id`/`workspace_id`)을 억지로 찾지 않고, JSON의 `needs_input` 배열로 **Resource Graph 쿼리 힌트와 재호출 예시**를 돌려준다. SRE Agent가 그 힌트로 값을 확정한 뒤 같은 도구를 다시 호출하는 구조이므로, 도구에 추가 권한을 주지 않아도 된다.

### B-2. 엔드포인트 등록/변경

```powershell
# 등록에 쓸 엔드포인트 재확인
$fqdn = az containerapp show -n <ACA_NAME> -g <ACA_RG> --query properties.configuration.ingress.fqdn -o tsv
"등록/수정할 값: https://$fqdn/mcp"
```

SRE Agent 커넥터에 위 `https://<fqdn>/mcp` 를 등록/수정한다.

> **보안 주의**: `externalIngress=true`(PoC 기본)는 `/mcp`를 공개한다. 운영에서는 `externalIngress=false`(내부) + Private Endpoint/APIM 인증, 또는 최소한 IP 제한을 건다. 진단 출력은 `_clean`으로 secret/PII/prompt-injection을 필터링하지만, 엔드포인트 접근 제어는 별도로 확보한다.
>
> 위 접근 제어는 `main.bicep`의 `enableEntraAuth`(ACA Easy Auth로 401 차단)·`allowedIpRanges`(IP 허용목록) 파라미터로도 켤 수 있다. 기본값은 둘 다 기존 동작 유지(인증 없음·무제한)이며, 운영 전환 시에만 `main.bicepparam`에서 값을 채운다. 자세한 내용은 `mcp/README.md` 4.1절 참고.

### B-3. 새 도구가 새 리소스 유형을 다룰 때 — 권한 부여

```powershell
./infra/assign-roles.ps1 -PrincipalId <UAMI_PRINCIPAL_ID> `
  -MonitoringScope "/subscriptions/<sub>/resourceGroups/<rg>" `
  -EventHubNamespaceId "<eh-namespace-resource-id>"
# ADX/PG/AKS 데이터 평면 권한은 스크립트 실행 후 출력되는 안내대로 각 서비스 내부에서 부여
```

---

## 4. 시나리오 C — GitHub 경로가 바뀔 때

### C-1. **mcp 리포** 자체가 이동/이름 변경 (`OrgA/repoA` → `OrgB/repoB`)

**런타임은 영향 없음** — SRE Agent→`/mcp`→ACA 경로에 GitHub가 없으므로 진단은 계속 동작한다. **CI/CD만** 고친다.

1. **OIDC 연합 자격 subject 갱신 (가장 중요)**
   Entra 앱의 federated credential subject를 새 리포로 바꾼다. 안 고치면 Actions의 `azure/login`이 실패한다.
   - main 브랜치: `repo:OrgB/repoB:ref:refs/heads/main`
   - 태그 배포도 쓰면: `repo:OrgB/repoB:ref:refs/tags/*` (또는 태그별 subject)
   - environment 보호를 쓰면: `repo:OrgB/repoB:environment:production`
   ```powershell
   # 기존 자격 확인
   az ad app federated-credential list --id <AZURE_CLIENT_ID/appId> -o table
   # 새 subject 추가 (예: main)
   az ad app federated-credential create --id <appId> --parameters '{
     \"name\": \"gh-OrgB-repoB-main\",
     \"issuer\": \"https://token.actions.githubusercontent.com\",
     \"subject\": \"repo:OrgB/repoB:ref:refs/heads/main\",
     \"audiences\": [\"api://AzureADTokenExchange\"]
   }'
   ```
2. **새 리포에 시크릿 재설정** — 1-3의 `mcp` 리포 시크릿을 새 리포 Settings에 다시 등록.
3. **컴포넌트 리포의 `MCP_REPO` 변수 갱신** — 6개 리포 모두 새 `OrgB/repoB` 로 바꿔야 dispatch가 도달한다.
4. **워크플로 파일은 리포와 함께 이동**하므로 1~3만 하면 그대로 동작.
5. **실행 머신 remote 갱신**
   ```powershell
   git remote set-url origin https://github.com/OrgB/repoB.git
   git remote -v
   ```

### C-2. **컴포넌트 리포**가 이동/이름 변경될 때

컴포넌트는 `mcp/requirements.txt`에 URL로 박혀 있으므로 **거기만 고치면 된다**.

```text
# 변경 전
pg-diagnose @ git+https://github.com/FactoryKor/pg.git@v1.0.0
# 변경 후 (조직/리포명 변경)
pg-diagnose @ git+https://github.com/OrgB/postgres-diag.git@v1.0.0
```

추가로 확인할 곳:

1. **`deploy-mcp.yml`의 자동 치환 정규식** — 조직명은 `[^/]+`로 받으므로 조직 변경은 그대로 동작한다. 단 **리포명이 바뀌면** `notify-mcp.yml`이 보내는 `component` 값(= 리포명)과 `requirements.txt`의 리포명이 일치해야 자동 버전 핀 갱신이 먹는다.
2. **새 리포에 `notify-mcp.yml`과 `MCP_DISPATCH_PAT`/`MCP_REPO`** 재설정.
3. **패키지명(`pg-diagnose`)과 콘솔 명령은 리포명과 무관**하다. `pyproject.toml`의 `[project.name]`/`[project.scripts]`가 결정하므로, 이걸 바꾸지 않는 한 `mcp_server.py`는 손댈 필요가 없다.

### C-3. **콘솔 명령 이름**이 바뀔 때 (드묾)

`pyproject.toml`의 `[project.scripts]` 키를 바꿨다면 `mcp/mcp_server.py`의 상수도 함께 바꾼다.

```python
PG_TOOL     = "pg-diagnose"        # ← [project.scripts] 키와 정확히 일치해야 함
AKS_TOOL    = "aks-diagnose"
ADX_TOOL    = "adx-diagnose"
EH_TOOL     = "eh-diagnose"
SVCMAP_TOOL = "svcmap-diagnose"
AGW_TOOL    = "agw-diagnose"
```

> **불변식**: `mcp_server.py`는 파일 경로를 전혀 모른다. PATH에 콘솔 명령이 설치돼 있기만 하면 동작하므로, 폴더 구조/트리 위치는 아무 영향이 없다.

### C-4. 새 진단 도구를 추가할 때

1. 새 리포 생성 → 진단기 `.py` + `requirements.txt` + `pyproject.toml`(`[project.scripts]`에 콘솔 명령 등록) + `.github/workflows/notify-mcp.yml` 복사.
2. `v1.0.0` 태그 push.
3. `mcp/requirements.txt`에 한 줄 추가 → `mcp/mcp_server.py`에 `@_tool()` 함수 추가(`_run()` 래퍼 사용 — `@mcp.tool()`을 쓰면 진단이 끝날 때까지 다른 요청이 모두 멈춘다).
4. 필요하면 `infra/assign-roles.ps1`로 UAMI에 새 리소스 읽기 권한 부여.
5. SRE Agent 커넥터 재동기화(B-1).

> HTML 리포트 게시(`report-publish`)는 `kind` 문자열만 새로 주면 되고, 인덱스 페이지 코드는 수정할 필요가 없다.

---

## 5. 검증 & 롤백

### 5-1. 배포 검증

```powershell
# 새 리비전이 Active/Running 인지
az containerapp revision list -n <ACA_NAME> -g <ACA_RG> -o table

# 현재 서비스 중인 이미지 태그 확인 (버전 고정 확인)
az containerapp show -n <ACA_NAME> -g <ACA_RG> `
  --query "properties.template.containers[0].image" -o tsv

# 컨테이너 로그 (기동/에러)
az containerapp logs show -n <ACA_NAME> -g <ACA_RG> --tail 50

# 헬스 엔드포인트(인증 불필요)
curl "https://$fqdn/health"        # {"status":"ok"}
```

**설치된 도구 버전 확인**(이미지 안에 의도한 태그가 들어갔는지):

```powershell
az containerapp exec -n <ACA_NAME> -g <ACA_RG> --command "pip list"
# pg-diagnose / adx-diagnose / eh-diagnose / agw-diagnose / svcmap-diagnose / aks-diagnose / report-publish
```

**기능 검증(종단)**: SRE Agent 채팅에서
> "Diagnose Event Hub `<namespace resource id>` in region `koreacentral`."

응답 JSON에 `version`, `summary`, `health_score`, `severity_counts`, `recommended_actions`가 들어 있는지 확인. 필수 값이 빠졌을 때 `needs_input`에 발견 힌트가 돌아오는지도 함께 본다.

### 5-2. 롤백 (이전 버전으로 즉시 복귀)

```powershell
# 이전에 배포했던 버전 태그로 다시 update
$acrLogin = az acr show -n <ACR_NAME> --query loginServer -o tsv
az containerapp update -n <ACA_NAME> -g <ACA_RG> `
  --image "$acrLogin/diag-mcp:v1.1.0"     # ← 직전 안정 버전

# 또는 리비전 기반 롤백 (해당 리비전으로 트래픽 100%)
az containerapp revision list -n <ACA_NAME> -g <ACA_RG> -o table
az containerapp ingress traffic set -n <ACA_NAME> -g <ACA_RG> `
  --revision-weight <이전리비전이름>=100
```

> **버전 고정 배포**를 지켜야 롤백이 쉽다. `latest`만 쓰면 "이전 것"을 특정하기 어렵다.

---

## 6. 트러블슈팅

| 증상 | 원인 후보 | 해결 |
|---|---|---|
| Actions `azure/login` 실패 | OIDC subject 불일치(리포 이동/브랜치·태그) | C-1의 federated credential subject 갱신 |
| 빌드 중 `Cannot find command 'git'` | 베이스 이미지에 git 없음 | mcp 리포의 `Dockerfile`이 `apt-get install git`을 하는지 확인(설치 후 purge함) |
| 빌드 중 `Repository not found` / 404 | 컴포넌트 리포가 private인데 PAT 없음 | `--secret-build-arg GH_PAT=<token>` 또는 리포를 Public으로 |
| 빌드 중 `Could not find a tag or branch 'vX.Y.Z'` | requirements.txt가 존재하지 않는 태그를 가리킴 | 컴포넌트 리포에 해당 태그를 push했는지 확인(`git ls-remote --tags`) |
| 컨테이너가 `pg-diagnose: not found` | 콘솔 스크립트 미등록 | 해당 리포에 `pyproject.toml`의 `[project.scripts]`가 있는지 확인 |
| 도구를 고쳤는데 동작이 그대로 | 버전 핀이 옆 태그에 고정됨 | 컴포넌트 태그 push → `requirements.txt` 버전 갱신 → 재빌드 |
| `az acr build` 권한 오류 | 로그인 주체에 ACR push 권한 없음 | `AcrPush`/기여자 역할 확인 |
| ACA가 이미지 pull 실패 | UAMI에 `AcrPull` 없음 | `infra/main.bicep`이 부여(재배포) 또는 수동 역할 부여 |
| SRE Agent가 새 파라미터 못 씀 | 도구 manifest 캐시 | B-1 커넥터 재동기화 |
| 커넥터가 `no active connection` | ACA Easy Auth와 커넥터 인증 방식 불일치 | 커넥터를 Entra ID 인증으로 맞추거나, 검증 중엔 `az containerapp auth update --enabled false` 로 임시 해제 |
| 진단 도구가 `error` 반환 | 진단기 실패(권한/네트워크) | 반환 JSON의 `stderr`, ACA 로그 확인. 대상별 읽기 역할 점검 |
| 응답에 `needs_input`만 있음 | 필수 식별자 미전달 | 정상 동작. `discovery_hint`의 Resource Graph 쿼리로 값을 찾아 재호출 |
| consumer lag이 `미평가` | 체크포인트 스토리지 접근 불가/미탐색 | UAMI에 `Storage Blob Data Reader` 부여, 또는 `checkpoint_store` 명시 |
| 리포트가 Blob에 안 올라감 | `REPORT_STORAGE_ACCOUNT` 미설정 | `enableReportPublish=true`로 재배포(환경변수 자동 주입) |
| 한글 깨짐(로컬 테스트) | 콘솔 인코딩 | `$env:PYTHONIOENCODING="utf-8"` (HTML/JSON 산출물은 UTF-8이라 무관) |

---

## 7. 빠른 체크리스트

**신규 구축 (처음부터)**
- [ ] 9개 리포에 `v1.0.0` 태그 존재 (`git ls-remote --tags`)
- [ ] `az deployment group create -f infra/main.bicep` 성공
- [ ] `assign-roles.ps1`로 UAMI 읽기 권한 부여
- [ ] `az acr build --file Dockerfile .` (mcp 리포 루트) 성공
- [ ] `az containerapp update --image ...:<VER>` → 리비전 Running
- [ ] `curl https://<fqdn>/health` → `{"status":"ok"}`
- [ ] SRE Agent 커넥터 등록 + 종단 테스트("Diagnose ...")

**진단 도구 수정 배포 (평상시)**
- [ ] 로컬 스모크(`--demo --format json`) + JSON 계약 키 확인
- [ ] 도구 리포에 `git push origin main` + `git tag vX.Y.Z && git push origin vX.Y.Z`
- [ ] mcp 리포 Actions에서 `requirements.txt` 버전 핀 자동 갱신 커밋 확인
- [ ] 새 리비전 Running (실패 시 자동 롤백됐는지 확인)
- [ ] 스키마 변경 시 → SRE Agent 커넥터 재동기화

**mcp 리포 이동**
- [ ] OIDC federated credential subject 갱신
- [ ] 새 리포에 시크릿 재등록
- [ ] 컴포넌트 6개 리포의 `MCP_REPO` 변수 갱신
- [ ] `git remote set-url origin <새 URL>`
- [ ] 푸시 → Actions 성공 확인 (런타임/SRE Agent는 무영향)

**새 진단 도구 추가**
- [ ] 새 리포에 `pyproject.toml`(`[project.scripts]`) + `notify-mcp.yml` + `v1.0.0` 태그
- [ ] `mcp/requirements.txt` 한 줄 추가
- [ ] `mcp/mcp_server.py`에 `@_tool()` 추가(`_run()` 래퍼 사용)
- [ ] 필요 시 `assign-roles.ps1`로 권한 부여
- [ ] 커넥터 재동기화 후 도구 목록에 노출 확인

---

*최종 업데이트: 2026-07-31 (멀티리포 pip 패키지 구조 기준) · 관련 파일: mcp 리포의 `requirements.txt`·`Dockerfile`·`mcp_server.py`·`.github/workflows/deploy-mcp.yml`, 각 컴포넌트 리포의 `pyproject.toml`·`.github/workflows/notify-mcp.yml`, infra 리포의 `main.bicep`·`assign-roles.ps1`*
