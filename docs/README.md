# Plan Harness 적용 가이드

Plan Harness는 AI와 함께 설계할 때 질문, 결정, 상세 설계, 검증 기준을 나누어 기록하는 문서 세트다. 이 가이드는 서비스 저장소에 문서를 가져오고 에이전트를 연결하는 방법을 설명한다. 설계 절차와 파일 형식의 원본은 [CLAUDE.md](./CLAUDE.md)다.

## 설계 전에 준비할 입력

| 입력 | 담을 내용 | 경로 |
| --- | --- | --- |
| Context | 문제, 사용자, 범위, 제약, 성공 기준 | `docs/context/<topic>.md` |
| Reference | 확정된 외부 시스템과 기술 환경의 사실 | `docs/reference/`의 관련 문서 |
| HLD | 시스템 경계, 주요 컴포넌트와 데이터 흐름 | `docs/hld/<feature>.md` |

사람이 입력을 작성하거나 AI가 만든 초안의 사실과 제약을 확인한다. HLD를 포함한 입력이 준비되어야 LLD 설계를 시작할 수 있다. 확정하지 않은 기술 선택을 Reference의 사실로 기록하지 않고 ADR 주제로 남긴다.

현재 포함된 [Context](./context/traffic-recording-replay.md)와 [Reference](./reference/)는 운영 트래픽 레코딩·리플레이 사례다. 다른 서비스의 사실로 그대로 적용하지 않는다. HLD·LLD·ADR·Validation 파일은 아직 없으며 작업하면서 생성한다.

## 서비스에 문서 가져오기

아래 예시는 Windows PowerShell에서 **서비스 저장소 루트**를 현재 폴더로 두고 실행한다. 상위 폴더의 `planharnessdocs-source`가 없는 경로여야 한다.

```powershell
if (Test-Path -LiteralPath './docs') {
    throw '기존 docs가 있습니다. 아래 병합 안내에 따라 필요한 파일만 가져오세요.'
}

git clone https://github.com/brightJoo/planharnessdocs.git ../planharnessdocs-source
if ($LASTEXITCODE -ne 0) {
    throw '저장소를 가져오지 못했습니다.'
}

Copy-Item -LiteralPath '../planharnessdocs-source/docs' -Destination './docs' -Recurse
```

결과는 다음과 같다. `hld/` 등 산출물 디렉터리는 아직 생성되지 않는다.

```text
service/
├── src/
├── AGENTS.md               # 기존 파일에 연결 지시 추가
└── docs/
    ├── CLAUDE.md           # 공통 설계 절차
    ├── README.md           # 이 가이드
    ├── example.md
    ├── context/
    └── reference/
```

기존 `docs/`가 있으면 임시 위치에 원본을 받은 뒤 경로별 diff를 보고 필요한 파일만 병합한다. 같은 이름의 `CLAUDE.md`, `README.md`, Context와 Reference를 덮어쓰지 않는다. 충돌하는 규칙을 먼저 조정하고, 예시 Context와 Reference는 해당 서비스 자료로 수정하거나 필요한 부분만 가져온다.

원본 저장소 자체가 `docs/`를 포함하므로 전체 저장소를 서비스의 `docs/`에 submodule 또는 subtree로 추가하면 `docs/docs/`가 된다. 위 방법은 원본의 `docs/`만 가져와 규칙의 `docs/...` 경로를 유지한다. 이후 원본 업데이트도 diff를 검토해서 병합한다.

## 에이전트 진입점 연결

### Codex와 Hermes

서비스 루트의 `AGENTS.md`에 다음 절을 추가한다. 파일이 없으면 새로 만들고, 이미 있으면 기존 코딩·검증 규칙을 보존한다.

```markdown
## 설계 문서 작업

설계 작업 전에 docs/CLAUDE.md를 읽고 공통 절차를 따른다.
관련 HLD, Context, Reference를 읽고 입력의 누락·모순을 먼저 보고한다.
LLD skeleton → ADR → final LLD → Validation을 순서대로 작성한다.
Validation 뒤 Human Review에서 멈추며 기존 항목의 카테고리를 임의로 바꾸지 않는다.
구현 작업은 이 저장소의 코딩·빌드·테스트 지침을 함께 따른다.
```

Codex는 서비스 루트를 프로젝트로 열어 새 세션을 시작한다. Hermes는 모델과 파일 읽기 도구가 설정된 환경에서 서비스 루트를 지정한다.

```powershell
hermes --in 'C:\work\my-service'
```

`C:\work\my-service`는 실제 서비스 경로로 바꾼다. Hermes에 `.hermes.md` 같은 별도 프로젝트 진입점이 있으면 그 파일에도 공통 절차를 읽도록 연결한다. 전역 설정이나 모델을 바꿀 필요는 없다.

### Claude Code

서비스 루트의 `CLAUDE.md`에 다음 지시를 추가한다. 기존 구현 규칙은 보존한다.

```markdown
설계 작업 전에 docs/CLAUDE.md를 읽고, 관련 HLD·Context·Reference를 확인한다.
공통 설계 절차에 따라 진행하고 Validation 뒤 Human Review에서 멈춘다.
```

이 원본 저장소의 루트 `CLAUDE.md`는 `@AGENTS.md`로 공통 진입 안내를 가져온다. 서비스에 이식할 때는 그 서비스의 기존 진입점에 필요한 지시를 병합한다.

자동 파일 발견과 링크 대상 읽기는 별개다. 세션을 새로 시작한 뒤 실제 읽은 파일 경로를 확인한다. 동작 기준은 [Codex 공식 문서](https://developers.openai.com/codex/guides/agents-md), [Hermes 공식 문서](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files), [Claude Code 공식 문서](https://code.claude.com/docs/en/memory)를 참고한다.

## 에이전트에 맡기는 요청 예시

### 입력 점검

```text
 docs/CLAUDE.md를 읽고 traffic-recording-replay 설계의 입력을 점검해줘.
 관련 Context·Reference·HLD의 실제 경로와 읽은 내용을 먼저 확인해줘.
 누락된 문서, 모순, 확정되지 않은 사실과 필요한 질문을 정리해줘.
 지금은 점검만 하고 설계 산출물이나 코드를 만들지 마.
```

처음에는 이 요청으로 연결과 입력 상태를 확인한다. 현재 원본 저장소에는 HLD가 없으므로 이를 누락으로 보고하는 것이 맞다. HLD 초안 작성이 필요하면 확인된 Context를 바탕으로 별도 요청하고 사실을 검토한다.

### 입력 준비 후 설계

다음 요청은 해당 서비스에서 관련 HLD·Context·Reference를 준비한 뒤 사용한다. feature 이름은 실제 입력 문서에 맞춘다.

```text
 docs/CLAUDE.md에 따라 traffic-recording-replay의 상세 설계를 진행해줘.
 관련 HLD·Context·Reference를 먼저 읽고 입력의 누락·모순이 있으면 보고하고 멈춰.
 입력이 충분하면 아래 단계를 각각의 산출물로 남겨줘.
 1. HLD에서 LLD 주제 목록 추출
 2. 질문 중심의 LLD skeleton과 ADR 주제 목록 작성
 3. ADR별 대안, 결정, 근거와 결과 작성
 4. ADR을 연결한 final LLD 작성
 5. 다섯 관점의 Validation 초안 작성
 입력 경로와 관련 ADR을 산출물 상단에 남기고 Human Review에서 멈춰.
 이번 요청에서는 구현 코드를 작성하지 마.
```

## 산출물과 검토 기준

| 산출물 | 남기는 것 | 경로 |
| --- | --- | --- |
| LLD skeleton | Scope, Interfaces, 답해야 할 Open Questions | `docs/lld/skeleton/<topic>.md` |
| ADR | 결정 하나의 대안, 선택, 근거와 결과 | `docs/adr/NNNN-<slug>.md` |
| final LLD | ADR을 반영한 행동 계약, 상태·데이터·처리 흐름 | `docs/lld/final/<topic>.md` |
| Validation | 검증 항목, 실패 시나리오, 카테고리와 근거 | `docs/validation/<topic>.md` |

HLD와 LLD에는 특정 프레임워크의 어노테이션·클래스명 대신 외부에서 관찰 가능한 동작을 쓴다. 동시성·순서 보장처럼 행동을 바꾸는 선택은 모델 수준의 ADR로 기록하고 구체 구현은 Reference와 서비스 코딩 규칙에 연결한다.

Validation은 **Correctness, Security, Concurrency, Performance, Resilience**의 다섯 관점으로 검토한다.

| 카테고리 | 의미와 필요한 근거 |
| --- | --- |
| `MUST_PASS` | 릴리즈 전 확인할 요구사항. 테스트·메트릭·수동 확인 등 통과 방법을 남긴다. |
| `KNOWN_ISSUE` | 인지하고 감수할 한계. 수락 이유와 해결 시점·조건을 적고 사람이 수락한다. |
| `DEFERRED` | 이번 범위 밖의 항목. 제외 이유를 남기고 사람이 범위를 확인한다. |

AI는 초기 분류까지 제안하며, 기존 카테고리의 변경은 사람이 판단한다. 설계를 바꾸는 요구사항이 승격되면 skeleton을 다시 열고 필요한 ADR을 추가·대체한 뒤 final LLD와 Validation을 갱신한다. 변경 이력과 전역 4자리 ADR 번호를 보존한다.

설계 리뷰 뒤의 구현·테스트는 서비스 저장소의 별도 작업이다. 반복 종료에는 `MUST_PASS`의 실제 통과 근거, `KNOWN_ISSUE`의 명시적 수락, `DEFERRED`의 범위 확인이 필요하다. 상세 형식과 단계 0~8은 [공통 설계 규칙](./CLAUDE.md), 흐름의 배경과 한 번의 iteration은 [설명 예시](./example.md)를 따른다.
