# Plan Harness

AI 에이전트와 함께 소프트웨어 설계를 진행할 때, **질문, 결정, 상세 설계, 검증 기준을 서로 다른 문서로 남기기 위한 문서 하네스**다.

핵심 흐름은 다음과 같다.

> **Context -> HLD -> LLD skeleton -> ADR -> final LLD -> Validation -> Human Review**

Plan Harness는 AI가 곧바로 구현 세부사항을 추정하지 않도록 입력과 의사결정의 경계를 만든다. 모르는 내용은 LLD skeleton의 Open Question으로 드러내고, 선택은 ADR에 근거와 함께 기록하며, 구현 전에 Validation으로 실패 가능성과 완료 조건을 검토한다.

## 30초 요약

| 항목 | 내용 |
| --- | --- |
| 목적 | AI 보조 설계를 재현 가능하고 리뷰 가능한 문서 흐름으로 관리한다. |
| 핵심 규칙 | 질문은 LLD skeleton, 결정은 ADR, 상세 설계는 final LLD, 완료 조건과 허용한 한계는 Validation에 둔다. |
| 사람의 판단 | 입력의 누락과 모순을 해소하고, Validation의 `MUST_PASS` / `KNOWN_ISSUE` / `DEFERRED` 분류를 최종 판단한다. |
| 현재 저장소 | 하네스 규칙, 일반화된 Context 예시, 기술 Reference, 워크플로우 설명 예시를 담고 있다. |
| 아직 없는 것 | 생성된 HLD, LLD skeleton/final, ADR, Validation 산출물은 현재 저장소에 없다. |

## 왜 필요한가

- **질문과 답을 분리한다.** LLD skeleton은 답을 꾸며내지 않고 설계가 답해야 할 질문만 보존한다.
- **결정의 이유를 보존한다.** ADR은 대안, 선택, 트레이드오프를 한 결정씩 기록한다.
- **구현 전에 완료 조건을 만든다.** Validation은 정상 동작뿐 아니라 보안, 동시성, 성능, 장애 상황을 검토한다.
- **사람의 책임을 명시한다.** AI는 초안을 만들 수 있지만, 허용할 위험과 이번 범위의 경계는 사람이 판단한다.

생산성 배수와 같은 효과를 전제하지 않는다. 이 저장소의 관심사는 속도 주장이 아니라, 설계 근거와 미해결 위험을 추적 가능한 형태로 남기는 것이다.

## 워크플로우

`Reference`는 이미 확정된 외부 시스템과 기술 환경의 사실을 담으며, Context와 HLD를 읽을 때 함께 로드한다.

| 단계 | 입력과 작업 | 결과 |
| --- | --- | --- |
| 1. Context | 문제, 사용자, 제약, 성공 기준을 기술한다. 기술 선택은 넣지 않는다. | `docs/context/<topic>.md` |
| 2. HLD | Context를 바탕으로 시스템 경계와 논리적 구조를 정의한다. | `docs/hld/<feature>.md` |
| 3. LLD skeleton | HLD를 구체화할 주제로 나누고, Scope, Interfaces, Open Questions만 작성한다. | `docs/lld/skeleton/<topic>.md` |
| 4. ADR | Open Question별 대안을 비교하고 결정과 결과를 기록한다. | `docs/adr/NNNN-<slug>.md` |
| 5. final LLD | skeleton의 질문을 ADR 결론으로 치환하고, 스택 독립적인 행동 계약을 완성한다. | `docs/lld/final/<topic>.md` |
| 6. Validation | LLD와 ADR의 누락을 대조하고 Correctness, Security, Concurrency, Performance, Resilience 관점에서 실패 시나리오를 분류한다. | `docs/validation/<topic>.md` |
| 7. Human Review | 사람이 Validation 카테고리와 근거를 검토한다. 승격된 항목이 있으면 skeleton과 ADR로 돌아간다. | 수락 또는 다음 iteration |

단계를 묶어서 건너뛰지 않는다. 입력 자료에 누락이나 모순이 있으면 하류 문서를 생성하지 않고 먼저 사람에게 보고한다.

## 사람의 판단이 들어가는 지점

1. **입력 준비와 승인**: Context, HLD, Reference의 사실과 제약은 사람이 작성하거나 확인한다.
2. **입력 충돌 해소**: Context, HLD, Reference가 충돌하면 에이전트는 진행을 멈추고 사람이 기준을 정한다.
3. **Validation 판정**: AI는 시나리오와 초기 분류를 제안할 수 있지만, 카테고리의 승격과 강등 권한은 사람에게만 있다.
4. **Iteration 결정**: `KNOWN_ISSUE`나 `DEFERRED`를 이번 범위의 `MUST_PASS`로 올리면 LLD skeleton과 ADR을 다시 작성한다.

final LLD 이전의 Open Question 대조는 누락을 찾는 품질 점검이다. 필수 Human Review 중단 지점은 Validation 이후이며, 상세 규칙은 [`docs/CLAUDE.md`](./CLAUDE.md)를 기준으로 한다.

## 현재 저장소에 있는 것

- [`docs/CLAUDE.md`](./CLAUDE.md): 단계, 파일 규칙, 권한 경계를 정의하는 authoritative workflow.
- [`docs/example.md`](./example.md): 하나의 iteration이 어떻게 연결되는지 보여주는 설명용 예시.
- [`docs/context/traffic-recording-replay.md`](./context/traffic-recording-replay.md): 공개용으로 일반화된 Context 자료.
- [`docs/reference/`](./reference/): 캐시, Kafka producer/consumer, object storage, replay mock user에 관한 참고 자료.

**현재 저장소는 하네스와 참고 자료를 문서화한다.** `docs/example.md`의 LLD/ADR/Validation 내용은 흐름을 설명하기 위한 서술형 예시이며, 실제로 생성된 산출물 파일이 아니다. 현재 트리에는 `docs/hld/`, `docs/lld/`, `docs/adr/`, `docs/validation/`이 없다.

## 예시를 확인하는 순서

1. [`docs/CLAUDE.md`](./CLAUDE.md)에서 단계 0~8과 반복 종료 조건을 읽는다.
2. [`docs/example.md`](./example.md)에서 LLD skeleton의 질문이 ADR과 final LLD, Validation으로 이어지는 한 번의 iteration을 확인한다.
3. [`Context 예시`](./context/traffic-recording-replay.md)에서 설계 입력이 어떤 수준의 문제와 제약을 담는지 본다.
4. [`Reference 디렉터리`](./reference/)에서 이미 결정된 기술 사실을 기능 설계와 분리하는 방식을 확인한다.

이 저장소를 실제 프로젝트에 적용할 때는 문서 세트를 프로젝트의 `docs/` 아래에 두고 Context, Reference, HLD를 프로젝트 상황에 맞게 준비한 뒤 `docs/CLAUDE.md`를 에이전트의 작업 규칙으로 사용한다.
