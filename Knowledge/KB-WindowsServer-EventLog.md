---
title: "Windows Server 이벤트 로그 기반 장애 진단"
keywords: [Windows Server, 이벤트 로그, Event Log, System log, Application log, 이벤트 ID, Event ID, 부팅 실패, 서비스 중단, 디스크 오류, WHEA, 블루스크린, bugcheck, 재부팅]
resource_types: [Microsoft.Compute/virtualMachines, Microsoft.HybridCompute/machines]
severity_hint: [warning, critical]
data_source: [Windows Event Log, Azure Monitor Agent, Log Analytics (Event table)]
version: 1.0.0
last_updated: 2026-07-22
---

# Windows Server 이벤트 로그 기반 장애 진단

> **한 줄 요약**: Windows Server의 System/Application 로그의 핵심 이벤트 ID를 기준으로 하드웨어·서비스·부팅·디스크 문제를 식별하고 대응한다.

## 1. 증상 (Symptoms)
- 서버 예기치 않은 재부팅/블루스크린(BSOD).
- 특정 서비스(SQL, IIS, 사용자 앱)가 반복 중단·재시작.
- 디스크 지연/오류, 이벤트 로그에 반복 오류(Error/Critical) 폭주.
- 로그인/인증 실패 급증(보안 로그).
- 알림 제목 예: "VM unexpected reboot", "Windows service crash", "Disk errors detected".

## 2. 핵심 이벤트 ID (원인 매핑)
| 로그 | Event ID | 소스 | 의미 / 원인 |
|------|----------|------|-------------|
| System | **41** | Kernel-Power | 정상 종료 없이 재부팅(전원 손실/커널 크래시/하드웨어) |
| System | **6008** | EventLog | 예기치 않은 종료 |
| System | **1001** | BugCheck | 블루스크린(BSOD) — bugcheck 코드 포함 |
| System | **7031 / 7034** | Service Control Manager | 서비스 비정상 종료·크래시 |
| System | **7000 / 7009 / 7011** | SCM | 서비스 시작 실패·타임아웃 |
| System | **51 / 153** | Disk | 디스크 페이징/I/O 오류 |
| System | **7 / 55** | Disk/Ntfs | 불량 블록·파일시스템 손상 |
| System | **18 / 19 / 20** | WHEA-Logger | 하드웨어 오류(정정/치명적) |
| Application | **1000 / 1026** | Application Error / .NET Runtime | 앱 크래시(모듈·예외 포함) |
| Application | **2004** | Perflib / Resource-Exhaustion | 메모리 고갈 진단 |
| Security | **4625** | 감사 실패 | 로그인 실패(다수 시 무차별 대입 의심) |
| System | **1074 / 1076** | User32/BugCheck | 재부팅 사유 기록 |

## 3. 진단 (Diagnosis)
**로컬 PowerShell (수동 확인):**
```powershell
# 최근 24시간 System 로그의 Critical/Error
Get-WinEvent -FilterHashtable @{ LogName='System'; Level=1,2; StartTime=(Get-Date).AddHours(-24) } |
  Group-Object Id | Sort-Object Count -Descending | Select-Object Count, Name -First 20

# 예기치 않은 재부팅 사유
Get-WinEvent -FilterHashtable @{ LogName='System'; Id=41,6008,1001 } -MaxEvents 20 |
  Select-Object TimeCreated, Id, Message
```
**Azure(Log Analytics, AMA 수집 시) KQL:**
```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLevelName in ("Error","Critical")
| summarize Count=count() by EventID, Source, EventLevelName
| sort by Count desc
```

## 4. 판정 기준 (Thresholds)
| 항목 | 양호 | 주의 | 위험 |
|------|------|------|------|
| Kernel-Power 41 / BugCheck 1001 | 없음 | 1회 | 반복 발생 |
| 서비스 크래시(7031/7034) | 없음 | 간헐 | 반복·핵심 서비스 |
| 디스크 오류(51/153/7) | 없음 | 산발 | 지속 발생 |
| WHEA 하드웨어 오류(18/19) | 없음 | 정정오류만 | 치명 오류 |
| 로그인 실패(4625) | 정상 수준 | 소폭 증가 | 급증(공격 의심) |

## 5. 조치 (Remediation)
1. **재부팅/BSOD(41,1001)**: bugcheck 코드로 원인 드라이버 식별, 최신 드라이버/펌웨어 적용, 메모리 덤프 분석. Azure VM이면 호스트 하드웨어 이슈 여부 확인(재배포).
2. **서비스 크래시(7031/7034)**: 해당 서비스 로그·앱 이벤트(1000) 확인, 복구 옵션(자동 재시작) 설정, 의존성 점검.
3. **디스크 오류(51/153/7)**: `chkdsk` 점검, SMART 상태 확인, Azure 관리 디스크면 성능 계층/크기 상향 검토.
4. **WHEA(18/19)**: 하드웨어 결함 → Azure는 재배포/호스트 이전, 온프렘은 부품 점검.
5. **메모리 고갈(2004)**: 원인 프로세스 식별, 메모리 증설 또는 앱 누수 수정.
6. **로그인 실패 급증(4625)**: 원본 IP 차단, 계정 잠금 정책·NSG/방화벽 강화, 노출된 RDP 포트 점검.

## 6. 검증 (Verification)
- 조치 후 동일 이벤트 ID가 **재발하지 않는지** 24~72시간 모니터링.
- 서비스 안정 가동(재시작 카운트 0), 디스크 오류 미발생.
- 로그인 실패 정상 수준 회복.

## 7. 롤백 / 주의 (Rollback & Caveats)
- 드라이버/펌웨어 업데이트는 검증 환경 먼저 → 문제 시 롤백 지점(복원 지점/스냅샷) 확보.
- Azure VM 재배포(redeploy)는 임시 디스크 데이터 소실 → 사전 백업.
- 보안 로그 조치(계정 잠금/IP 차단)는 정상 사용자 영향 주의.

## 8. 참고 (연계)
- 데이터 소스: Azure Monitor Agent(AMA) → Log Analytics `Event` 테이블, 또는 로컬 `Get-WinEvent`.
- 연계: VM 성능 지표(CPU/메모리/디스크) 진단과 교차 분석.
- 반복 하드웨어 오류는 Azure 플랫폼(호스트) 이슈일 수 있으므로 재배포로 격리 확인.
