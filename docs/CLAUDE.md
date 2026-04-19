# Top-Down AI 설계 워크플로우

이 저장소는 **HLD → LLD → ADR → Validation**으로 이어지는 top-down 설계 과정을 AI와 함께 수행하기 위한 문서 저장소다. 각 단계의 생성물은 파일로 남고, Human-in-the-loop 체크포인트에서 사람의 판단을 받아 상위 단계로 돌아가 반복(iterate)한다.

핵심 원칙:
1. **단계를 건너뛰지 않는다.** skeleton 없이 final LLD를 쓰지 않고, ADR 없이 결정을 LLD에 박지 않는다.
2. **모르는 것은 답하지 않는다.** 답 대신 Open Question으로 남긴다.
3. **결정과 검증을 분리한다.** 결정은 ADR, 검증은 Validation 문서. 섞지 않는다.
4. **사람의 판단은 Validation 리뷰 한 곳에 모은다.** 중간 단계에서 사람을 기다리지 않는다 (워크플로우 전체를 예측 가능하게 유지).
5. **LLD는 언어/프레임워크 독립적이다.** LLD만 보고 Go · Spring · 그 외 프레임워크 어느 쪽으로도 구현이 가능해야 한다. 스택-특수 용어(예: `@RestController`, `Mono<T>`, `goroutine`, `@Async`)는 LLD에 쓰지 않는다. 스택 지식은 `docs/reference/`에만 둔다.
   - 단, **동시성 모델**(event-loop vs 멀티스레딩, blocking vs non-blocking) 같은 **행동에 영향을 주는 선택**은 ADR로 남긴다. 이 ADR은 프레임워크를 지정하지 않고 모델 수준에서 기술한다 ("event-loop 기반 단일 스레드 처리" ○ / "WebFlux 채택" ×).

---

## 디렉터리 구조

```
docs/
├── CLAUDE.md                  # 본 문서
├── context/                   # 비즈니스/문제 컨텍스트 (스택 의존 X)
│   └── <topic>.md             # 배경, 사용자/스테이크홀더, 제약, 목표 KPI
├── reference/                 # 기술 지식 베이스 (외부 의존성 + 스택 정보)
│   ├── kafka.md
│   ├── db.md
│   ├── external-api-<name>.md
│   └── stack-<name>.md        # 예: stack-spring.md (JDK25, WebFlux/MVC 등)
├── hld/
│   └── <feature>.md           # High-Level Design (논리적 구조, 스택 의존 X)
├── lld/
│   ├── skeleton/
│   │   └── <topic>.md         # LLD 골격: 섹션 + Open Questions만
│   └── final/
│       └── <topic>.md         # 완성된 LLD (언어/프레임워크 독립)
├── adr/
│   └── NNNN-<slug>.md         # Architecture Decision Record
└── validation/
    └── <topic>.md             # LLD 기반 검증 항목
```

파일명 규칙:
- `<feature>` / `<topic>`: kebab-case (예: `task-scheduler.md`)
- ADR 번호(`NNNN`): 4자리 zero-padded, 저장소 전역 순번 (예: `0007-retry-policy.md`)

---

## 입력 자료: Context / Reference

LLD skeleton 단계 이전에 이미 존재해야 하는 참조 문서들이다. **사람이 미리 채워둔다** (또는 별도 단계로 AI가 작성할 수 있다 — 어느 쪽이든 workflow 시작 시점엔 존재해야 한다).

### Context (`docs/context/`)
- 비즈니스 배경, 사용자/스테이크홀더, 제약(규제·타임라인·예산), 목표 KPI.
- "왜 이 시스템이 필요한가"에 대한 답. **기술 선택은 들어가지 않는다.**

### Reference (`docs/reference/`)
- **외부 시스템**: Kafka, DB(Postgres/MySQL/...), External API 명세 요약 (토픽, 테이블, 엔드포인트, SLA).
- **스택 정보**: 이미 결정된 언어/프레임워크/런타임 세부. 예:
  - `stack-spring.md` — JDK25, Spring Boot X.Y, **WebFlux vs MVC 중 어느 쪽을 쓰는지**, 팀이 보유한 패턴.
  - `stack-go.md` — Go 버전, 사용 라이브러리, 팀 관행.
- Reference는 "**이미 결정된** 환경/지식"을 기술한다. **결정이 진행 중이라면 ADR**이다. 둘을 섞지 않는다.

> **경계선**: LLD와 HLD는 reference의 스택-특수 용어를 **소비하지 않는다**. Reference는 구현 단계(LLD → 코드)에서 읽힌다. LLD 안에 `Mono<T>`, `@Controller`, `goroutine`이 등장하면 그건 버그다.

---

## Workflow 단계

### 0. Context / Reference 로드 (필수 선행)
Skeleton 작업을 시작하기 전, AI는 다음을 **전부 읽는다**:
1. 관련 `docs/hld/<feature>.md`
2. `docs/context/` 중 이 feature와 연관된 모든 문서
3. `docs/reference/` 중 관련 외부 의존성 + 스택 문서

세 부류 모두에서 누락·모순을 먼저 식별한다. 모순이 있으면 작업을 진행하지 말고 사람에게 보고한다 (Context와 Reference가 맞지 않으면 하류 모든 산출물이 틀어진다).

### 1. HLD → LLD 주제 추출
- **입력**: `docs/hld/<feature>.md` + Context + Reference
- **출력**: LLD 주제 목록 (각 주제가 하나의 `lld/skeleton/*.md` 파일이 된다)
- **AI의 역할**: HLD를 읽고 **구체화가 필요한 모듈/컴포넌트/데이터 경로 단위**로 LLD 주제를 쪼갠다. 한 주제가 너무 크면 더 쪼갠다.
- **주의**: 이 단계에서는 LLD 파일을 아직 만들지 않는다. 목록만 뽑는다.

### 2. LLD Skeleton + ADR 주제 추출
- **입력**: 단계 1의 LLD 주제 목록 + HLD + Context + Reference
- **출력**:
  - `docs/lld/skeleton/<topic>.md` — 섹션 헤더와 **Open Questions**만 포함
  - ADR로 다룰 **결정 사항 목록** (아직 ADR 파일 생성 X)
- **핵심**: skeleton은 **답을 쓰지 않는다. 질문만 쓴다.** "이 모듈은 어떻게 동작할 것인가"가 아니라 "이 모듈에서 답해야 할 질문은 무엇인가"를 쓴다.
- **Skeleton 최소 섹션**:
  - `## Scope` — 이 LLD가 다루는 범위와 다루지 않는 범위
  - `## Interfaces` — 이 컴포넌트가 노출하는/의존하는 인터페이스 (질문 형태 가능)
  - `## Open Questions` — 결정되어야 할 사항 목록. 각 항목에 `→ ADR-XXXX (proposed)` 태그를 붙여 ADR과 연결

### 3. ADR 작성
- **입력**: 단계 2의 ADR 주제 목록
- **출력**: `docs/adr/NNNN-<slug>.md` — 주제당 1개
- **형식**:
  ```markdown
  # NNNN. <결정 제목>
  Status: Proposed | Accepted | Superseded by NNNN
  Date: YYYY-MM-DD

  ## Context
  왜 이 결정이 필요한가. 제약은 무엇인가.

  ## Options
  1. Option A — 장단점
  2. Option B — 장단점
  3. (최소 2개 이상)

  ## Decision
  채택된 옵션과 그 이유.

  ## Consequences
  이 결정의 긍정적/부정적 파급 효과, trade-off.
  ```
- **원칙**: 하나의 ADR은 하나의 결정만. 여러 결정이 엮이면 쪼갠다.

### 4. LLD 완성
- **입력**: `docs/lld/skeleton/<topic>.md` + 관련 ADR 전부 (+ Context · Reference 재확인용)
- **출력**: `docs/lld/final/<topic>.md`
- **작업**: skeleton의 Open Question을 해당 ADR의 Decision으로 치환하고, 구체 설계를 채운다.
- **링크**: 각 결정 지점에 `(see ADR-NNNN)` 참조를 남긴다.
- **허용되는 표현** (언어/프레임워크 독립):
  - 데이터 스키마: JSON Schema / 유사 pseudo-IDL / ER 다이어그램
  - 인터페이스: "함수 A는 입력 X를 받아 출력 Y를 반환한다. 실패 시 에러 타입 E를 반환한다" 수준의 **행동 계약** (method signature가 아님)
  - 상태 전이: 상태 머신 / ASCII 다이어그램 / Mermaid
  - 동시성: "이 연산은 non-blocking이어야 한다", "이 두 연산은 strict ordering이 필요하다" 수준의 **제약**
  - 시퀀스/플로우: Mermaid sequence diagram, pseudo-code
- **금지되는 표현**: 특정 언어의 문법·어노테이션·타입, 특정 프레임워크의 클래스명, Kafka client 라이브러리의 구체 API. 이런 것은 구현 단계에서 reference를 보고 mapping 한다.
- **Self-check**: LLD를 Go 팀과 Spring 팀에게 각각 건넸을 때, 각 팀이 서로 다른 기술로 구현해도 동일한 **외부 관찰 가능 동작**(API 계약 + 이벤트/상태 변화)에 도달할 수 있는가? 아니면 스택-특수 가정을 어딘가에 흘린 것이다.

### 5. Validation 문서 작성

**이 워크플로우에서 가장 중요한 산출물이다.** LLD·ADR이 "무엇을 만들 것인가"를 정의한다면, validation은 **"무엇이 완성인가"** 를 정의한다. 그리고 ADR이 "왜 이렇게 했는지"를 답한다면, validation은 **"왜 이건 안 했는지"** 를 답한다. 6개월 뒤 "이거 왜 안 막았어요?" 질문은 `KNOWN_ISSUE` 항목이 답한다. MUST_PASS는 당연한 것이고, **실제 가치는 KNOWN_ISSUE의 근거에서 나온다.**

**이 단계는 워크플로우 전체에서 가장 '깊게 생각해야' 하는 단계다.** LLD·ADR은 "해결책"을 쓰지만, validation은 **"해결책이 어디서 깨지는가"** 를 쓴다 — 서로 다른 사고 방식이며, 후자가 훨씬 어렵다. 따라서 **사고 예산을 이 단계에 가장 많이 할당한다.**
- AI 에이전트는 이 단계에서 extended-thinking / deep-reasoning 모드를 활성화한다 (Claude Code라면 `ultrathink`).
- 사람은 다른 단계보다 **의도적으로 더 긴 시간**을 쓴다.
- 빠르게 적당히 넘기면 놓친 시나리오가 그대로 장애가 된다. 각 관점을 **독립된 공격자의 시선**으로 읽고, "통과함"을 확인하려 하지 말고 **"깨뜨릴 방법"** 을 찾는다.

- **입력**: 완성된 LLD + 해당 ADR
- **출력**: `docs/validation/<topic>.md`
- **작성 시점**: **LLD final 직후, 구현 시작 전.** 구현 후에 쓰면 "이미 만든 것"만 체크하게 되어, 의도적으로 감수한 구멍이 보이지 않는다.

**문서 구조 — 3부**

1. **LLD 준수 체크 (기계적 대조)**
   - LLD의 모든 API / ERD 엔티티 / DDL 항목 / ADR 결론이 validation 항목과 1:1로 매핑되는가?
   - 누락이 하나라도 있으면 LLD 단계로 돌아간다.

2. **다관점 브레인스토밍** — LLD를 **다섯 관점에서 공격**한다. 각 관점마다 독립적으로 "이걸 어떻게 깨뜨릴까?"를 묻고, 깨질 수 있는 시나리오를 최대한 많이 나열한다. 이 단계에서는 **너무 많이 뽑는 게 너무 적게 뽑는 것보다 낫다** — 판정은 3부에서 한다.
   | 관점 | 무엇을 묻는가 |
   |---|---|
   | **Correctness** | 정상 흐름, 엣지 케이스, 상태 전이, 빈 값·경계값 *(가정: 인프라는 정상 동작)* |
   | **Security** | 인증 우회, 타인 데이터 접근, 대량 요청, 입력 인젝션 |
   | **Concurrency** | 동시 요청, 다른 작업과 경합, 락·UNIQUE 의존 |
   | **Performance** | 쿼리 복잡도, 데이터 증가 시 성능, 외부 호출 지연 |
   | **Resilience** | 의존성 장애 시 동작 (DB·외부 API·큐), retry/CB/fallback 정책, **관측성** (log·metric·trace로 원인 판별 가능한가), 배포·롤백 안전성 (DDL 호환, in-flight 요청) |

   **관점 경계.** Correctness는 "인프라가 정상일 때 로직이 맞는가", Resilience는 "인프라가 비정상일 때 시스템이 어떻게 행동하는가". 두 축을 분리해야 "해피 패스 전용 설계"를 피할 수 있다. Security(의도적 악의)·Concurrency(타이밍 간섭)·Performance(부하·비용)는 각각 독립 축이다.

3. **판정** — 나열된 시나리오를 다음 세 카테고리 중 하나로 분류. **모두 통과할 필요 없다.**
   | 카테고리 | 의미 |
   |---|---|
   | `MUST_PASS` | 릴리즈 전 반드시 통과 |
   | `KNOWN_ISSUE` | 알려진 한계 — 명시적으로 감수 |
   | `DEFERRED` | 이번 iteration 범위 밖 |
   - 경계를 흐리지 않는다. 애매하면 `MUST_PASS` 쪽으로.
   - `KNOWN_ISSUE`에는 **근거를 반드시** 쓴다 — "왜 감수하는가", "언제 해결할 예정인가"까지. **"나중에 하겠다"만 적으면 안 된다.**
   - **근거 품질 체크:** `KNOWN_ISSUE` 근거는 *"1년 뒤 post-mortem에서 이 근거를 그대로 소리내어 읽었을 때 납득이 되는가"* 로 검증한다. 납득되지 않으면 `MUST_PASS` 승격 후보다.
   - `DEFERRED`는 LLD의 "Scope 제외" 범위와 일치해야 한다.

**항목 포맷**
```markdown
### V-001 <항목 제목>
Category: MUST_PASS | KNOWN_ISSUE | DEFERRED
Source: LLD §2.3 / ADR-0007 / 브레인스토밍-Resilience
Rationale: 왜 이 카테고리인가. KNOWN_ISSUE·DEFERRED는 **근거 필수** —
           왜 감수하는가, 언제 해결할 예정인가까지 기술.
Acceptance: MUST_PASS인 경우 어떻게 통과를 확인하는가 (테스트·메트릭·수동 확인).
```

**AI의 권한 제한 (재확인)**
- AI는 `MUST_PASS ↔ KNOWN_ISSUE ↔ DEFERRED` **카테고리 변경 권한이 없다.** 브레인스토밍과 초안 작성까지만 수행한다.
- 카테고리 승격·강등은 단계 6 Human Review에서 사람이 한다.

### 6. 🛑 Human Review — 체크포인트
사람이 validation 문서를 보고 다음 세 가지만 판단한다:
1. `MUST_PASS` 항목이 정말 must-pass 수준인가? (과잉이거나 부족한가)
2. `KNOWN_ISSUE` 중 실제로는 `MUST_PASS`로 **승격**되어야 할 것이 있는가?
3. `DEFERRED` 중 이번 iteration에서 `MUST_PASS`로 당겨야 할 것이 있는가?

리뷰 결과는 validation 문서에 직접 수정(카테고리 변경 + Rationale 업데이트)으로 반영한다. 이 수정이 다음 iteration의 트리거다.

### 7. LLD Skeleton 재작성
- 승격된 항목이 있으면 `docs/lld/skeleton/<topic>.md`를 다시 연다.
- 승격으로 인해 기존 Decision이 흔들리는지 확인하고, **새로운 Open Question**을 추가한다.
- 기존 skeleton을 덮어쓰지 않고, 변경 이력(`## Changelog`)을 파일 하단에 누적한다.

### 8. ADR 재작성
- 승격으로 기존 결정을 뒤집어야 하는 경우:
  - 기존 ADR을 직접 수정하지 말고, **새 ADR**을 만들어 `Status: Accepted (Supersedes NNNN)`로 표기한다.
  - 기존 ADR은 `Status: Superseded by MMMM`으로 업데이트한다.
- 새 결정이 필요한 경우: 새 ADR 추가.

→ 이후 **단계 4 (LLD 완성)** 로 돌아가 루프를 계속한다.

---

## 반복 종료 조건

모든 validation 항목이 다음 중 하나를 만족하면 iteration 종료:
- `MUST_PASS` → 통과 확인됨
- `KNOWN_ISSUE` → 사람이 명시적으로 수락
- `DEFERRED` → 사람이 명시적으로 이번 범위 밖임을 확인

---

## AI가 각 단계를 수행할 때의 규칙

- **입력 파일을 먼저 읽는다.** 이전 단계 산출물을 읽지 않고 다음 단계를 시작하지 않는다. LLD 작업 시작 전에는 반드시 HLD + Context + Reference를 전부 로드한다 (단계 0).
- **누락된 정보는 추측하지 않는다.** Open Question으로 남기고, 어느 ADR이 이를 답할 것인지 표기한다.
- **한 번에 여러 단계를 묶지 않는다.** 예: "HLD 읽고 바로 final LLD 작성" 금지.
- **Human Review 체크포인트(단계 6)에서 반드시 멈춘다.** AI가 카테고리 변경을 임의로 하지 않는다. 카테고리 변경 권한은 사람에게만 있다.
- **각 산출물 상단에 메타데이터**(생성일, 입력이 된 문서 경로, 관련 ADR 목록)를 남긴다. 재작성 시 Changelog를 누적한다.
- **LLD/HLD에서 스택-특수 용어를 쓰지 않는다.** `@RestController`, `Mono<T>`, `goroutine`, `KafkaTemplate`, `RxJava` 등의 단어가 LLD에 등장하면 그건 버그다. 그런 용어는 `docs/reference/` 안에서만 쓴다. 의미 있는 동시성/ordering 제약은 ADR에서 **모델 수준**으로 기술한다.
