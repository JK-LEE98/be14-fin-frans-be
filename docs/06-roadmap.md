# 06. 로드맵 (W0 ~ W8)

> 약 2개월(2026-06 ~ 2026-08)의 실행 계획. 각 주는 **목표 → 산출물 → 검증** 형식.
> "검증"은 `00-overview` §4 성공 기준과 연결된다. 완료 판정은 검증 통과로 한다.
> 기준일: 2026-07-21 (W0 진행 중).

## 원칙

- 코드를 바꾸기 전에 **특성화 테스트로 현재 동작을 고정**한다(W1이 W2 앞에 오는 이유).
- 각 주제(동시성/회복탄력성/AI)는 "필요한 지점"에만 관통 적용한다(`00` §3 범위).
- 설계 결정은 그 주에 **ADR로 기록**한다(구현 시점에 씀).

---

## W0 — 토대 정립 (문서 · 설계의도 · AI 관점) 〔진행 중〕

코드/CI를 건드리기 전에 "무엇을·왜·어디로"를 확정한다.

- 산출물:
  - [x] `docs/00-overview` — 목적·범위·성공기준
  - [x] `docs/01-architecture-as-is` — 현재 구조 진단(ground truth)
  - [x] `docs/02-domain-approval` — 결재 설계의도
  - [x] `docs/adr/0001` (ADR 관례), `0002` (Rich Domain)
  - [x] `docs/06-roadmap` (이 문서)
  - [ ] `docs/03-concurrency`, `04-resilience`, `05-ai-anomaly-detection`
  - [ ] `CLAUDE.md` 최종 확정(이미 존재 → 보완)
  - [ ] `00` §4 성공 기준 수치 확정
- 검증: docs/가 리팩토링의 ground truth로 성립(스스로 읽어 큰 그림이 잡힘).

## W1 — 안전망 (특성화 테스트 + CI 그린)

리팩토링이 동작을 깨뜨리는지 자동 검증되는 상태를 만든다.

- 목표: 결재 핵심 플로우를 테스트로 고정하고, 실DB 통합 테스트가 CI에서 돈다.
- 산출물:
  - Testcontainers(MariaDB) 기반 통합 테스트 토대(`project_cicd_setup` 참조).
  - 결재 특성화 테스트: 생성/승인/반려/재상신(=`02` §2 규칙)의 입력→최종상태·응답 고정.
  - `.github/workflows/ci.yml`(PR + develop push, 전체 build) 그린. `cr.yml`은 `workflow_dispatch`로 봉인.
- 검증: CI 그린 + 특성화 테스트가 `02` §2 규칙을 커버.

## W2–W3 — 결재 엔진 리팩토링 (메인)

`ApprovalCommandServiceImpl` 867줄을 해체한다. **외부 동작 보존**(W1 테스트가 계속 그린).

- 목표: 상태전이를 애그리거트로, 분기/생성 중복 제거, 트랜잭션 내 알림 분리.
- 산출물 + ADR:
  - Rich Domain(`Approval`/`ApprovalLine`) — ADR 0002 구현.
  - 문서 링크 Strategy(`ApprovalDocumentLinker`) — 신규 ADR(D2).
  - 라인 생성 Factory(`ApprovalLineFactory`) — 신규 ADR(D3).
  - 승인 후처리 도메인 이벤트 `AFTER_COMMIT` — 신규 ADR(D4).
- 검증: 서비스 라인 수 목표 달성(`00` §4), `setStatus` 외부 호출 0, 특성화 테스트 그린.

## W4 — 동시성

경합을 구조적으로 다룬다.

- 목표: 동시 승인 이중완료 방지 + 스케줄러 레이스 제거.
- 산출물 + ADR:
  - 낙관적 락 `@Version`(결재 동시 승인) — 신규 ADR(D5), 상세 `03-concurrency`.
  - 분산 락(Redisson) 필요 지점 식별·적용.
  - `OrderStatusScheduler` `hasKey→set`(01 §4.1) → `setIfAbsent`(원자적).
- 검증: 동시 요청 부하 테스트에서 이중완료 0건.

## W5 — 회복탄력성 (Redis CB + SSE 확장)

외부 의존 장애에 견디고, SSE를 다중 인스턴스로 확장한다.

- 목표: Redis 장애 시 fallback, SSE 인메모리 → 다중 인스턴스 fan-out.
- 산출물: Resilience4j CircuitBreaker + fallback(`04-resilience`), SSE Redis Pub/Sub.
- 검증: Redis 장애 주입 시 전체 실패로 안 번짐, CB 상태 전이 확인.

## W6 — 배치

통계 생성의 성능·멱등성을 확보한다.

- 목표: row-by-row save → bulk upsert + 멱등.
- 산출물: 통계 배치 재작성(`00` §4 관련 지표).
- 검증: 재실행해도 결과 동일(멱등), 처리 시간 개선 측정.

## W7 — AI 이상 주문 감지 〔차별화 핵심〕

3단계 하이브리드. 상세 설계는 `05-ai-anomaly-detection`.

- 목표: 저비용 통계 필터 → 의심건만 LLM 판단 → SSE 실시간 알림.
- 산출물: 1차 z-score 필터, 2차 LLM(structured JSON, temperature 0, 캐싱), 3차 SSE 연동, 평가셋.
- 검증: precision 목표(`00` §4), LLM 호출은 1차 통과분만, LLM 실패 시 z-score degrade.

## W8 — 마무리 (보안 · 관측 · 문서)

- 목표: 보안 강화 + 관측 대시보드 + 포트폴리오 문서 정리.
- 산출물: RTR/블랙리스트/`@PreAuthorize`, Grafana 대시보드, docs/·옵시디언 최종화.
- 검증: 성공 기준(`00` §4) 전 항목 리뷰, 데모 가능 상태.

---

## 트랙 간 의존

```
W0(토대) → W1(안전망) → W2-3(결재) → W4(동시성) → W5(회복탄력성) → W6(배치) → W7(AI) → W8(마무리)
                         └ W1 없이는 W2 이후 리팩토링의 안전 보장 불가 ┘
```

AI(W7)는 통계 테이블·SSE(W5)에 얹히므로 후반. 단 `05` 설계 문서는 W0에서 미리 잡는다(차별화 1순위).
