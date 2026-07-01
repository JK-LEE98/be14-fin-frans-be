# 01. 현재 아키텍처 (As-Is)

> 리팩토링 착수 전, 코드 기준으로 검증한 현재 상태 스냅샷.
> 목적: Claude Code와 리팩토링을 진행할 때 참조하는 **ground truth**.
> 기준일: 2026-07-01 (main/develop 기준, 코드 직접 스캔).

## 1. 시스템 개요

- **도메인**: 프랜차이즈 주문관리 ERP (본사 ↔ 가맹점).
- **스택**: Spring Boot 3.2.5 / Java 17 / MariaDB / Redis.
- **아키텍처 원칙**: CQRS — command는 JPA, query는 MyBatis(`Mapper.xml`).
- **규모**: Java 493 파일, `Mapper.xml` 22개, 도메인 패키지 22개.

## 2. 패키지 / 도메인 맵

패키지 컨벤션: `com.x1.frans.<domain>.<command|query>.…`

| 구분 | 도메인 |
|---|---|
| command + query 둘 다 | approval, order, purchase, purchaseorder, returns, product, franchise, supplier, statistics, user |
| command 만 | auth, notification, outbound, warehouse |
| 기타 구조 | aws, email(service 중심), redis, security, config, exception, delivery(enums) |

**관찰된 불균형**: CQRS가 전 도메인에 균일하게 적용돼 있지 않다. 일부는 command만 존재한다.
→ 설계의도 문서(`02`)에서 "왜 이 도메인은 query가 없는가(=조회 요구가 없나, 아니면 미완성인가)"를 정리할 필요.

## 3. 결재(approval) 도메인 — 메인 리팩토링 대상

### 3.1 애그리거트 구성

- 핵심: `ApprovalEntity`, `ApprovalLineEntity`
- 결재선 템플릿: `ApprovalLineTemplateEntity`, `ApprovalLineTemplateDetailEntity`
- 첨부: `ApprovalFileEntity`
- **문서 링크 3종**: `OrderApprovalEntity`, `PurchaseOrderApprovalEntity`, `ReturnApprovalEntity`
  → 결재 문서가 주문/발주/반품 중 무엇에 연결되는지를 별도 링크 엔티티로 표현.

공통 enum: `ApprovalStatus`, `ApprovalLineStatus`, `ApprovalLineType`, `ApprovalCategoryType`.

query 응답 DTO는 `Detail/content/{PurchaseOrder,OrderReturn}/…`로 깊게 분화 → 조회 응답 복잡도가 높다.

### 3.2 확인된 코드 악취 (라인 근거)

파일: `approval/command/application/service/ApprovalCommandServiceImpl.java` (**867줄**)
(같은 패키지의 다른 서비스는 28·31줄 — God Class 대비가 극명)

| 악취 | 근거 |
|---|---|
| God Class | `ApprovalCommandServiceImpl` 867줄, `@Transactional` 메서드 11개 |
| 분기 중복(switch) | `categoryType`/`category` switch 4곳 (line 113, 284, 431, 662) |
| 메서드 중복 | `createApproval`(59) vs `createApprovalDrafts`(252) — 생성 로직 대부분 중복 |
| Anemic Domain | 상태전이·검증이 서비스에 있음. `ApprovalEntity`는 데이터 홀더에 가까움 |
| 트랜잭션 경계 침범 | `@Transactional` 내부에서 알림 호출 (line 199 요청알림 / 489 반려알림) — 알림 생성이 결재 트랜잭션에 동기 결합 |

> **정확성 노트**: 여기서의 "알림"은 `NotificationService`를 통한 DB 기록(알림 row 생성)이다.
> 순수 외부 I/O(메일/SSE 푸시)인지, DB 쓰기인지는 `notification` 도메인 본문 확인 후 확정한다.
> 어느 쪽이든 "결재 성공 트랜잭션 ↔ 알림"의 동기 결합은 도메인 이벤트 `AFTER_COMMIT` 분리 후보다.

## 4. 기술적 갭 (동시성 / 회복탄력성 / 테스트)

코드 전수 스캔 결과:

| 항목 | 현재 상태 |
|---|---|
| `@Version` (낙관적 락) | **0개** |
| `@Lock` (비관적 락) | **1개** — `outbound/…/OutBoundCommandRepository`(`PESSIMISTIC_WRITE`)가 유일 |
| Redisson (분산 락) | 0개 |
| Resilience4j (CircuitBreaker/Retry) | 0개 |
| `SseEmitter` | 4곳 사용 (인메모리 방식) |
| 테스트 | 2개 (`FransApplicationTests` contextLoads + `JwtTestConfig`) |

### 4.1 확인된 대표 결함 — 스케줄러 레이스

`order/command/application/scheduler/OrderStatusScheduler.java`:

```java
if (Boolean.TRUE.equals(redisTemplate.hasKey(redisKey))) return;   // line 44
// ... 처리 ...
redisTemplate.opsForValue().set(redisKey, "true", Duration.ofHours(24)); // line 55
```

`hasKey`(check) → `set`(act) 사이가 원자적이지 않다(TOCTOU). 다중 인스턴스 환경에서 중복 실행 가능.
→ 동시성 파트에서 `setIfAbsent`(원자적 SET NX) 교체 대상.

## 5. 변경 정책 (무엇을, 어떤 조건에서 바꾸는가)

기본 원칙은 "외부 동작(API 응답 · DB 상태) 보존"이다. 단 이것은 **절대 금지가 아니라 기본값 + 예외 게이트**다.
변경 자체가 죄가 아니라, **근거 없이 · 은밀히 · 편의로** 바꾸는 것이 죄다.

### 5.1 자유롭게 바꾸는 것 — 리팩토링 본체

- 서비스 내부 구조, 도메인 로직 위치(서비스→애그리거트), 분기 처리(switch→다형성/전략), 트랜잭션 후처리(동기→`AFTER_COMMIT` 이벤트).
- 조건: **외부 동작 불변** + 특성화 테스트 그린 유지.

### 5.2 기본은 보존, 근거 있으면 변경 가능 — 외부 표면

대상: 공개 API 응답 스키마, DB 스키마/테이블, 결재 상태 규칙의 의미.

- **기본값**: 보존.
- **변경 허용 사유** (둘 중 하나가 명확할 때만):
  1. **정확성 결함** — 현재 동작이 버그다. 이 경우 "동작 보존"은 곧 "버그 보존"이므로 고치는 게 맞다.
  2. **근거 기반 성능 향상** — before/after 측정치가 있는 개선.
- **변경 금지 사유**: 단순 편의 · 취향 · "이게 더 깔끔해서".

### 5.3 절대 원칙 — 은밀히 바꾸지 않는다 (프로세스)

외부 동작을 바꾸는 순간 그것은 "리팩토링"이 아니라 "기능/버그 수정"이다. 따라서:

- 리팩토링 커밋과 **분리**한다(같은 커밋에 섞지 않는다).
- 사유 · 트레이드오프 · 마이그레이션 영향을 **ADR로 남긴다**.
- 특성화 테스트는 "옛 동작 고정"이 아니라 **"의도된 새 동작"으로 갱신**하고, 왜 갱신했는지 커밋 메시지에 남긴다.

> 안전장치는 **특성화 테스트(W1)**다. 5.1은 테스트로 현재 동작을 고정한 뒤 구조만 바꾼다.
> 5.2는 테스트가 "무엇이 의도적으로 달라졌는지"를 드러내는 장치가 된다.
