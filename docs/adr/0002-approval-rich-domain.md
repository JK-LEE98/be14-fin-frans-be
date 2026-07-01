# 0002. 결재 상태전이를 서비스에서 애그리거트로 옮긴다 (Rich Domain)

- 상태(Status): 채택됨(Accepted)
- 날짜: 2026-07-01
- 관련: `docs/02-domain-approval.md` (D1), `docs/01-architecture-as-is.md` §3

## 맥락 (Context)

결재의 상태전이 규칙이 전부 `ApprovalCommandServiceImpl`(867줄) 안에 흩어져 있다.

- 승인(`approverApprove`): 라인 `WAITING` 확인 → `APPROVED` → 다음 라인 `WAITING` → 남은 라인 없으면 문서 `APPROVED`.
- 반려(`approverReject`): 라인 `WAITING` 확인 → 라인 `REJECTED` → 문서 `REJECTED`.
- 생성/수정: 순서 라인은 `getInitialStatus`로 초기화, 참조/수신은 상태 null.

이 규칙들이 서비스에서 `approval.setStatus(...)` / `line.setStatus(...)` 형태로 존재한다.  
결과적으로:

- **불변식을 강제할 곳이 없다.** 누구든 `setStatus`로 규칙을 우회해 잘못된 상태를 만들 수 있다.
- **규칙이 여러 메서드에 중복·산재**한다(라인 초기화 로직만 3곳: create/createDrafts/modify).
- `ApprovalEntity`/`ApprovalLineEntity`는 필드+`onCreate`뿐인 데이터 홀더(Anemic Domain).

한편 `ApprovalLineType`은 이미 `isOrdered()` / `getInitialStatus(index)`라는 도메인 로직을  
갖고 있다 — 즉 이 도메인은 리치하게 갈 **씨앗이 이미 있다.**

## 결정 (Decision)

상태전이와 불변식을 **애그리거트(`Approval`, `ApprovalLine`)로 이동**한다.

- `ApprovalLine`: `approveBy(user)`, `rejectBy(user, opinion)`, `activate()`(EXPECTED→WAITING) 등  
  라인 단위 전이를 소유하고, "WAITING에서만 승인/반려 가능" 같은 라인 불변식을 스스로 강제한다.
- `Approval`: `approve(line)`, `reject(line)`, `resubmit()`을 소유하고,  
  "모든 순서 라인이 APPROVED여야 문서 APPROVED", "WAITING 라인은 최대 1개" 같은  
  집합 불변식을 강제한다. 다음 라인 활성화도 여기서 조율한다.
- `setStatus` 같은 무분별한 setter는 제거하고, 상태 변경은 위 의미 있는 메서드로만 한다.
- 서비스는 얇아진다: 조회 → 애그리거트 메서드 호출 → 저장 → (후처리는 이벤트, 별도 ADR).

**보존 계약**: 위 규칙의 *결과*(같은 입력 → 같은 최종 상태·응답)는 바뀌지 않는다.  
이를 W1 특성화 테스트로 먼저 고정한 뒤 이동한다.

## 대안 (Alternatives)

- **(a) 서비스만 잘게 쪼갠다**: God Class는 줄지만 규칙이 여전히 서비스에 있어 산재·우회 문제는 그대로. → 기각.
- **(b) 상태 패턴(State Pattern) 전면 도입**: 상태별 클래스로 전이를 캡슐화.  
  문서 상태 4개(DRAFT/IN_PROGRESS/APPROVED/REJECTED)에 비해 클래스 폭증 → 현 규모엔 과함.  
  → 지금은 애그리거트 메서드 + enum 로직으로 충분. 규칙이 더 복잡해지면 재검토(보류).
- **(c) 도메인 서비스로 뺀다**: 애그리거트 밖 별도 도메인 서비스에 규칙 배치.  
  일부 교차 규칙엔 유효하나, 여기 규칙 대부분은 `Approval` 한 애그리거트 내부라 애그리거트가 자연스러운 자리. → 기각.

## 결과 (Consequences)

- 얻음: 상태 규칙이 애그리거트 한 곳에 응집. `setStatus` 우회로 인한 잘못된 상태가 구조적으로 불가능해짐.
- 얻음: 라인 초기화 등 중복이 애그리거트/팩토리(별도 ADR)로 수렴.
- 얻음: 서비스가 얇아져 테스트·가독성 향상.
- 비용: 엔티티가 도메인 로직을 갖게 되어 **JPA 매핑과 도메인 순수성 사이 긴장**이 생긴다  
  (기본 생성자·식별자 관리 등). 이 긴장은 감수하며, 필요 시 별도 ADR로 다룬다.
- 후속: 문서 링크 분기(Strategy), 라인 생성(Factory), 승인 후처리(도메인 이벤트),  
  동시 승인 방지(낙관적 락)는 각각 별도 ADR에서 이 결정 위에 얹는다.
