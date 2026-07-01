# 02. 결재(Approval) 도메인 설계의도

> 메인 리팩토링 대상인 결재 도메인의 현재 규칙과 목표 설계를 정리한다.  
> 개별 "결정"의 대안·근거·트레이드오프는 `adr/`에 별도 기록한다(이 문서는 큰 그림).  
> 기준일: 2026-07-01. 근거는 코드 기준(`approval/…`), 상태전이는 `ApprovalCommandServiceImpl` 검증.

## 1. 도메인 어휘

| 개념 | 설명 | 코드 |  
|---|---|---|  
| **Approval** | 결재 문서 1건. 제목·본문·상태·차수를 가짐 | `ApprovalEntity` |  
| **ApprovalLine** | 결재선의 한 줄(누가 어떤 역할로 결재하는가) | `ApprovalLineEntity` |  
| **degree(차수)** | 재상신 이력. 반려 후 다시 올리면 `maxDegree+1` | `ApprovalEntity.degree` |  
| **category** | 결재 대상 문서 유형 | `ApprovalCategoryType` = ORDER(1)/RETURN(2)/PURCHASE_ORDER(3) |  
| **문서 링크** | 결재 ↔ 실제 업무문서 연결(유형별 별도 테이블) | `OrderApprovalEntity`/`PurchaseOrderApprovalEntity`/`ReturnApprovalEntity` |  

### 1.1 결재선 종류 (`ApprovalLineType`)

| 타입 | 순서 있음 | 상태 |  
|---|---|---|  
| APPROVER(결재자) | O | WAITING→APPROVED/REJECTED |  
| COOPERATOR(협조자) | O | 동일 |  
| REFERENCE(참조자) | X | 상태 없음(null) |  
| RECIPIENT(수신자) | X | 상태 없음(null) |  

> 이미 도메인 로직의 씨앗이 있다: `ApprovalLineType.isOrdered()`, `getInitialStatus(index)`.  
> (첫 순번 → WAITING, 이후 → EXPECTED, 비순서 타입 → null). 목표 설계는 이 씨앗을 확장하는 방향.

### 1.2 상태 (`ApprovalStatus` / `ApprovalLineStatus`)

- 문서: `DRAFT`(임시) → `IN_PROGRESS`(결재 중) → `APPROVED` / `REJECTED`
- 라인: `EXPECTED`(예정) → `WAITING`(대기=내 차례) → `APPROVED` / `REJECTED`

## 2. 현재 상태전이 규칙 (검증됨 — 보존 대상)

`ApprovalCommandServiceImpl` 기준. **이 규칙들이 리팩토링에서 보존해야 할 "외부 동작"이다.**

- **생성**(`createApproval` 59): 기안 시 문서 `IN_PROGRESS`, 아니면 `DRAFT`. `degree = maxDegree+1`. 순서 라인은 `getInitialStatus`로 초기화, 참조/수신은 상태 null.
- **승인**(`approverApprove` 370):
    1. 현재 라인이 `WAITING`이 아니면 거부(388)
    2. 현재 라인 → `APPROVED`(394)
    3. 다음 `EXPECTED` 라인 → `WAITING`(412)
    4. 남은 `WAITING`/`EXPECTED`가 없으면 문서 → `APPROVED`(425)
- **반려**(`approverReject` 445):
    1. 라인 `WAITING` 확인(464)
    2. 라인 → `REJECTED`(471)
    3. 문서 → `REJECTED`(479)
- **재상신**: 반려된 문서를 다시 올리면 `degree` 증가(`modifyApproval` 근처 616~).

## 3. 진단 — 왜 리팩토링하는가

`docs/01` §3.2에서 식별한 악취를, 결재 도메인 맥락으로.

- **상태전이가 서비스에 흩어져 있음(Anemic Domain)**: 위 §2의 규칙이 전부 867줄 서비스 안에 `setStatus(...)`로 존재. `ApprovalEntity`는 필드+`onCreate`뿐. → 규칙의 응집도 0, 어디서든 상태를 깰 수 있음.
- **분기 중복(switch 4곳)**: `categoryType`(ORDER/RETURN/PURCHASE_ORDER)로 문서 링크·조회를 분기(113, 284, 431, 662).
- **라인 생성 3중 중복**: create / createDrafts / modify가 같은 라인 초기화 로직 반복(174·184 / 339·349 / 719·729).
- **트랜잭션 경계 침범**: 승인/반려 트랜잭션 내부에서 알림 호출(199 요청 / 489 반려).

## 4. 목표 설계 (Rich Domain)

원칙: **§2의 상태전이 규칙을 서비스에서 애그리거트로 이동**한다. 규칙은 한 곳(애그리거트)에서만 바뀐다.

```  
[서비스]  얇은 오케스트레이션(조회·저장·이벤트 발행)  
   │  호출  
   ▼[Approval]         approve(line) / reject(line) / resubmit()  ← 상태전이·불변식 소유  
[ApprovalLine]     approveBy(user) / rejectBy(user) / activateNext()  
[ApprovalLineType] 이미 가진 isOrdered/getInitialStatus 확장  
```  

## 5. 주요 설계 결정 (각 항목 = ADR 후보)

각 결정은 `adr/`에 대안·근거·트레이드오프로 상세화한다. 여기서는 방향만.

| # | 결정 | 근거(악취) | 트레이드오프 |  
|---|---|---|---|  
| D1 | **Rich Domain** — 상태전이를 `Approval`/`ApprovalLine`로 이동 | Anemic, 규칙 산재 | 엔티티에 로직 → JPA 매핑과 도메인 순수성 긴장 |  
| D2 | **Strategy** — 문서 링크(`ApprovalDocumentLinker`)로 category 분기 대체 | switch 4곳 | 타입 추가 시 전략 구현 필요(대신 OCP 확보) |  
| D3 | **Factory** — `ApprovalLineFactory`로 라인 생성 통합 | 라인 생성 3중 중복 | 생성 경로 일원화(간접층 1개 추가) |  
| D4 | **도메인 이벤트 `AFTER_COMMIT`** — 승인 완료 후처리(주문/반품/발주 반영 + 알림) 분리 | 트랜잭션 내 알림 | 최종 일관성(즉시성↓, 결합도↓) |  
| D5 | **낙관적 락 `@Version`** — 동시 승인 이중완료 방지 | `@Version` 0개 | 충돌 시 재시도 필요(상세는 `03-concurrency`) |  

> D4·D5는 각각 `04-resilience`/`03-concurrency`와도 얽힘 — 결재 도메인 관점만 여기에.

## 6. 불변식 (Invariants) — 애그리거트가 지켜야 할 것

- 문서가 `APPROVED`이려면 모든 순서 라인이 `APPROVED`여야 한다.
- 한 시점에 `WAITING`인 순서 라인은 최대 1개(현재 차례).
- 라인 승인/반려는 `WAITING` 상태에서만 가능하다.
- 참조/수신 라인은 상태를 갖지 않는다(null).
- 반려 후 재상신은 `degree`를 증가시킨다.

## 7. 보존 계약 (01 §5와 연결)

리팩토링 전후로 **동일해야 하는 것**:

- 위 §2 상태전이의 **결과**(같은 입력 → 같은 최종 상태·응답).
- 공개 API(`ApprovalCommandController`)의 요청/응답 스키마.
- `approval*` 테이블 스키마.

이를 고정하는 장치가 **W1 특성화 테스트**다. §2를 테스트로 먼저 박은 뒤 §4~5를 적용한다.
