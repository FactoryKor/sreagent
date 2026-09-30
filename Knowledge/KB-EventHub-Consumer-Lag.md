---
title: "Event Hub 소비 지연 (Consumer Lag) 대응"
keywords: [Event Hub, consumer lag, 소비 지연, backlog, slow consumer, partition skew, checkpoint, 처리 지연, EventHubs, 이벤트 밀림]
resource_types: [Microsoft.EventHub/namespaces]
severity_hint: [warning, critical]
mcp_tool: diagnose_eventhub
version: 1.0.0
last_updated: 2026-07-22
---

# Event Hub 소비 지연 (Consumer Lag) 대응

> **한 줄 요약**: 컨슈머 그룹이 유입 속도를 못 따라가 lag(backlog)이 쌓이는 상황. 원인은 대개 느린 컨슈머, 파티션 skew, 컨슈머 수 부족, 다운스트림 병목 중 하나다.

## 1. 증상 (Symptoms)
- 특정 컨슈머 그룹의 **consumer lag / backlog(미처리 이벤트 수)**이 지속 증가.
- 이벤트 처리 지연(end-to-end latency) 상승, 다운스트림(ADX/DB) 데이터 신선도 저하.
- `IncomingMessages`(유입)는 정상인데 `OutgoingMessages`(소비)가 낮음.
- 특정 파티션만 lag이 큼(→ skew 의심).
- 알림 제목 예: "Event Hub backlog high", "Consumer lag increasing".

## 2. 원인 후보 (Probable Causes)
| 원인 | 신호 |
|------|------|
| **느린 컨슈머(slow consumer)** | 전체 파티션 균등하게 lag 증가, 컨슈머 CPU/처리시간 상승 |
| **파티션 skew** | 소수 파티션에만 lag 집중, `partition_skew` finding |
| **컨슈머 수 부족** | 파티션 수 > 활성 컨슈머 인스턴스 수, 처리량 한계 |
| **다운스트림 병목** | 컨슈머가 쓰는 DB/ADX/API 지연 → 컨슈머가 blocking |
| **컨슈머 크래시/재시작 루프** | lag 급증 + 체크포인트 갱신 중단 |
| **처리량 유닛(TU)/PU 부족** | throttling(`ThrottledRequests`) 동반 |

## 3. 진단 (Diagnosis)
**진단도구 실행 (MCP):**
```text
diagnose_eventhub(
  namespace_fqdn = "<ns>.servicebus.windows.net",
  resource_id    = "/subscriptions/.../Microsoft.EventHub/namespaces/<ns>",
  region         = "<region>",
  window_minutes = 60,
  checkpoint_store = "<blob checkpoint URL>"   # 없으면 needs_input로 재요청됨
)
```
- 출력의 `summary`, `recommended_actions`, `health_score`, `severity_counts` 확인.
- **`needs_input`에 `checkpoint_store`가 있으면**: 체크포인트 스토리지를 찾아 `checkpoint_store`로 **재호출**해야 정확한 lag 산출 가능. (체크포인트는 앱 데이터평면 → ARM/ARG로 위치 조회 불가. 앱설정/KeyVault 연결문자열에서 찾을 것.)
- 파티션별 lag 분포로 slow consumer vs skew 구분.

## 4. 판정 기준 (Thresholds)
> 고객별로 조정 가능. Agent가 지식에서 이 값을 읽어 `knowledge_context`로 도구에 주입 가능.

| 항목 | 양호 | 주의(warning) | 위험(critical) |
|------|------|---------------|----------------|
| Consumer lag(backlog) | 안정/감소 | 지속 증가 | 급증 + 계속 상승 |
| 파티션 skew | 균등 | 일부 편중 | 소수 파티션 집중 |
| ThrottledRequests | 0 | 간헐 발생 | 지속 발생 |
| 활성 컨슈머 수 | ≥ 파티션 수 대응 | 부족 | 크게 부족 |

## 5. 조치 (Remediation)
1. **원인이 slow consumer**: 컨슈머 처리 로직 최적화(배치 처리, 비동기 I/O), 컨슈머 인스턴스 **수평 확장**(파티션 수까지).
2. **원인이 partition skew**: 파티션 키 전략 재검토(균등 분산), 필요 시 파티션 수 증설(신규 hub).
3. **원인이 다운스트림 병목**: DB/ADX 용량·인덱스·수집 속도 개선(각 진단도구 `diagnose_postgres`/`diagnose_adx` 연계).
4. **원인이 throttling**: 처리량 유닛(TU)/PU 상향 또는 Auto-Inflate 활성화.
5. **컨슈머 크래시**: 예외/재시작 루프 로그 확인 후 안정화, 체크포인트 갱신 재개 확인.

## 6. 검증 (Verification)
- 조치 후 `diagnose_eventhub` 재실행 → lag이 **감소 추세**로 전환, `health_score` 상승 확인.
- `OutgoingMessages` ≥ `IncomingMessages` 회복, 파티션별 lag 균등화.
- 다운스트림 데이터 신선도 정상화.

## 7. 롤백 / 주의 (Rollback & Caveats)
- 파티션 수 증설은 **되돌릴 수 없음** → 신중히.
- 컨슈머 수평 확장 시 파티션 수를 초과해도 처리량은 늘지 않음(파티션당 1 컨슈머 상한).
- TU/PU 상향은 비용 증가 → 조치 후 유입 안정화되면 원복 검토.

## 8. 참고 (연계)
- 관련 도구: `diagnose_eventhub` (health_score/severity_counts/recommended_actions/needs_input).
- 다운스트림 연계: `diagnose_adx`, `diagnose_postgres`.
- 자율 발견 루프: `needs_input.checkpoint_store` → Agent가 앱설정/KeyVault로 위치 발견 후 재호출.
