# 설계 워크플로우 — 배경과 예시

> 이 문서는 [`CLAUDE.md`](./CLAUDE.md) 워크플로우의 **배경(rationale)**과 **한 번의 iteration 예시**를 담는다.
> 규칙·파일 경로·네이밍은 CLAUDE.md가 authoritative. 이 문서와 충돌 시 CLAUDE.md 우선.

---

## 하네스 엔지니어링

이 저장소는 **하네스 엔지니어링**으로 개발한다. 잘 만든 설계 문서(LLD + ADR + Validation)가 AI 에이전트를 제어하고, 코드의 품질을 결정한다.

- LLD를 쓰려면 **결정**이 필요하고, 결정을 내리려면 **큰 그림**이 필요하다.
- LLD skeleton이 **질문을 발견**하고, ADR이 **질문에 답**하고, LLD final이 **조립**한다.
- **LLD에 없으면 구현하지 않는다.** 불명확하면 질문한다.

---

## 워크플로우 한눈에

```
analyze (HLD + context + reference 로드)
    ↓
LLD skeleton (섹션 헤더 + Open Questions만)
    ↓
ADR 작성 (질문별 옵션 비교 → 결정 + 근거)
    ↓
리뷰 (skeleton의 Open Questions vs ADR 1:1 대조 + 놓친 질문 확인)
    ↓
LLD final (ADR 결론 조립, 언어·프레임워크 독립)
    ↓
Validation (5관점 시나리오 + 판정)
    ↓
🛑 Human Review (카테고리 승격 권한은 사람에게만)
    ↓
(승격 있으면 skeleton·ADR 단계로 돌아가 루프)
    ↓
구현 (LLD 기준, validation의 MUST_PASS로 완료 판단)
```

단계별 상세는 [`CLAUDE.md`](./CLAUDE.md) 참조.

---

## LLD skeleton과 final — 둘 다 보존하는 이유

skeleton은 **"원래 뭘 몰랐는지"** 를 기록한다. final만 있으면 이미 결정된 것처럼 보여서 놓친 질문을 발견하기 어렵다.

에이전트의 점검은 기계적 대조에 그치지 않고 **skeleton 작성 시점에 발견하지 못한 질문**까지 찾는다. final LLD 이전의 이 점검은 별도 Human Review가 아니다. 사람의 필수 검토는 Validation 뒤에 모으며, 입력 모순이 있으면 그보다 먼저 보고하고 멈춘다.

```
skeleton: Open Questions 4개 나열
ADR:      4개 작성
    ↓
리뷰 4-1 (AI): skeleton의 Open Questions vs ADR 1:1 대조 → 누락 없음
    ↓
리뷰 4-2 (AI): "더 필요한 질문은 없나?"
  ├── skeleton에 적지 못한 질문이 있는가?
  ├── ADR 결론을 보니 새로운 질문이 드러나는가?
  └── 추가 질문이 있다면 → ADR 단계로 돌아감
    ↓
LLD final: ADR 결론 조립
```

## LLD와 ADR의 관계

**1 LLD : N ADR.** ADR은 LLD skeleton 작성 중에 발생한다. skeleton을 쓰다가 "이거 어떻게 하지?"가 나오면 그것이 ADR 주제가 된다. ADR에서 옵션을 비교해 결정하면, 그 결론이 LLD final에 반영된다.

구현할 때는 **LLD final만 보면 된다.** ADR은 "왜 이렇게 했지?"가 궁금할 때 참고한다. 이것이 ADR과 LLD를 분리하는 이유다.

---

## Validation이 가장 중요하다

`MUST_PASS`는 당연한 것. **진짜 가치는 `KNOWN_ISSUE`에서 나온다.**

```
6개월 후: "이거 왜 안 막았어요?"
    → validation.md의 KNOWN_ISSUE 항목:
      "인지하고 있음. 근거: ~~~ 이유로 감수. 해결 시점: Phase 2"
```

- **ADR이 "왜 이렇게 했는지"를 답하듯, validation이 "왜 이건 안 했는지"를 답한다.**
- LLD final 직후·구현 시작 전에 작성한다. 구현 후에 쓰면 구현 편향이 생겨 감수한 구멍이 안 보인다.
- **5관점** 브레인스토밍(Correctness / Security / Concurrency / Performance / **Resilience**) + 3 카테고리 판정. 상세 포맷은 [`CLAUDE.md`](./CLAUDE.md) 단계 5.
- **이 단계가 가장 깊게 생각해야 하는 단계다.** AI는 extended-thinking 모드(`ultrathink`), 사람은 의도적으로 더 긴 시간을 쓴다. 공격자 시선으로 **"깨뜨릴 방법"** 을 찾는다.

---

## Iteration 예시: 댓글 신고

실제 워크플로우가 한 번 도는 모습이다. 파일 경로·ADR 번호 규칙은 [`CLAUDE.md`](./CLAUDE.md) 기준(전역 4자리).

```
lld/skeleton/comment-report.md:
  ## Scope
    포함: 댓글 신고 접수 + 어드민 처리
    제외: 콘텐츠 신고 (별도 iteration)
  ## Interfaces
    POST 신고 / GET 신고 목록 / PUT 미노출
  ## Open Questions
    Q1 신고 사유 저장 방식?              → ADR-0001 (proposed)
    Q2 신고된 댓글 즉시 미노출 구현?      → ADR-0002 (proposed)
    Q3 신고 시점 원문 스냅샷 저장?        → ADR-0003 (proposed)
    Q4 어드민 전체 미노출 방식?           → ADR-0004 (proposed)

adr/0001-report-reason-storage.md       → Decision: enum + description 컬럼
adr/0002-hide-filter-strategy.md        → Decision: report 존재 여부 NOT EXISTS 필터
adr/0003-reported-body-snapshot.md      → Decision: 원문을 reported_body 컬럼에 스냅샷
adr/0004-admin-bulk-hide.md             → Decision: review 테이블에 is_hidden 컬럼

리뷰:
  4-1 (AI) : Q1~Q4 ↔ ADR-0001~0004, 누락 없음
  4-2 (AI) : ADR-0003를 보니 "신고자 식별 정보도 함께 보존해야 하나?"가 드러남
             → Open Question Q5 추가, ADR-0005 작성 후 다시 올라옴

lld/final/comment-report.md:
  ERD·DDL·API 스펙·쿼리 전략 확정 (언어/프레임워크 독립 — 행동 계약 수준)

validation/comment-report.md:
  MUST_PASS    : 5건  (인증 필수 / 타인 계정 신고 차단 / 동일 신고 UNIQUE /
                       목록 페이지네이션 정확성 / 미노출 즉시 반영)
  KNOWN_ISSUE  : 2건  (예: 신고자 rate-limit 미구현 — 근거: 초기 트래픽 예상치 낮음,
                       해결 시점: Phase 2, 트리거: 일 1,000건 초과)
  DEFERRED     : 1건  (콘텐츠 신고 — Scope 제외와 일치)
```

---

## reference는 왜 필요한가

LLD를 쓸 때 외부 시스템과 이미 결정된 스택 지식이 필요하다. reference는 **기능과 무관하게 공유되는 팩트**다.

- LLD가 "이 기능은 이렇게 만든다"라면, reference는 **"우리 시스템은 이렇게 되어 있다"** 이다.
- 모든 LLD가 같은 reference를 참고한다.
- **"이미 결정된" 것만** reference에 둔다. 결정이 진행 중이면 ADR이다. 둘을 섞지 않는다.
- 스택-특수 용어(예: 특정 프레임워크 클래스명·어노테이션·라이브러리 API)는 **reference 안에서만** 쓴다. LLD/HLD에 흘러들면 버그다.
