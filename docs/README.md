# planharnessdocs

AI와 함께 top-down 설계(HLD → LLD → ADR → Validation)를 수행하기 위한 **문서 템플릿 & 규칙 저장소**.

이 저장소는 **단독으로 쓰는 프로젝트가 아니라**, 실제 서비스 코드가 있는 다른 프로젝트(예: 백엔드 서버 repo)에 `docs/` 폴더로 끼워 넣어 쓰는 **문서 세트**다. 붙여 넣은 뒤 그 프로젝트의 비즈니스 컨텍스트(`context/`)와 스택 정보(`reference/`)를 채워 넣으면, AI 에이전트가 [`CLAUDE.md`](./CLAUDE.md)의 워크플로우에 따라 설계 문서를 생성·갱신한다.

> **README는 사람을 위한 소개문이고, 실제 작업 규칙은 [`CLAUDE.md`](./CLAUDE.md)에 있다.** AI 에이전트에게 코딩을 맡길 때는 CLAUDE.md를 reading list에 반드시 포함시킨다. README는 보조 문서다.

---

## 어떻게 사용하는가

이 저장소를 가져다 쓸 프로젝트(예: `planharness-backend/`)의 **루트에서** 다음 중 하나를 실행하면, 이 repo가 그 프로젝트의 `docs/` 폴더로 들어온다:

```bash
# submodule 방식
git submodule add <repo-url> docs

# 또는 subtree 방식
git subtree add --prefix docs <repo-url> main --squash
```

결과 구조:
```
planharness-backend/       ← 실제 서비스 코드가 있는 프로젝트
├── src/
└── docs/                  ← 여기로 이 저장소가 들어옴
    ├── CLAUDE.md
    ├── context/
    ├── reference/
    └── ...
```

이후:
1. `docs/context/`에 비즈니스 배경·스테이크홀더·제약·KPI를 기록한다.
2. `docs/reference/`에 외부 시스템(Kafka, DB, External API)과 이미 결정된 스택 정보(`stack-spring.md`, `stack-go.md` 등)를 기록한다.
3. `docs/hld/`에 High-Level Design을 작성한다.
4. AI 에이전트가 `docs/CLAUDE.md`의 단계에 따라 LLD skeleton → ADR → LLD final → Validation을 생성한다.
5. Validation 문서에서 사람이 리뷰 후 카테고리를 조정한다 (→ 필요 시 iteration 루프).

---

## 디렉터리 구조

```
docs/
├── CLAUDE.md                 # ★ AI가 따르는 워크플로우 규칙 (authoritative)
├── README.md                 # 본 문서 (소개·보조)
├── context/                  # 비즈니스/문제 컨텍스트 — 사람이 미리 채움
│   └── <topic>.md
├── reference/                # 외부 의존성 + 스택 정보 — 사람이 미리 채움
│   ├── kafka.md
│   ├── db.md
│   ├── external-api-<name>.md
│   └── stack-<name>.md
├── hld/
│   └── <feature>.md          # High-Level Design (스택 의존 X)
├── lld/
│   ├── skeleton/             # Open Questions 중심의 LLD 초안
│   │   └── <topic>.md
│   └── final/                # 결정 반영된 완성 LLD (언어·프레임워크 독립)
│       └── <topic>.md
├── adr/
│   └── NNNN-<slug>.md        # Architecture Decision Record (저장소 전역 4자리)
└── validation/
    └── <topic>.md            # LLD 검증 항목 + Human Review 체크포인트
```

파일명 컨벤션과 각 문서의 상세 포맷은 [`CLAUDE.md`](./CLAUDE.md) 참조.

---

## 철학 — 한눈에

1. **단계를 건너뛰지 않는다.** skeleton 없이 final LLD를 쓰지 않고, ADR 없이 결정을 LLD에 박지 않는다.
2. **모르는 것은 답하지 않는다.** Open Question으로 남기고, 어느 ADR이 답할지 표기한다.
3. **결정과 검증을 분리한다.** 결정은 ADR, 검증은 Validation. 섞지 않는다.
4. **사람의 판단은 Validation 리뷰 한 곳에 모은다.** AI는 중간 단계에서 사람을 기다리지 않는다 — 워크플로우 전체를 예측 가능하게 유지하기 위함이다.
5. **LLD는 언어·프레임워크 독립적이다.** LLD만 보고 Go · Spring · 그 외 어느 스택으로도 구현 가능해야 한다. 스택-특수 용어는 `reference/`에만 둔다. 단, **동시성 모델처럼 행동에 영향을 주는 선택**은 ADR로 남긴다(모델 수준: "event-loop 단일 스레드" ○ / "WebFlux 채택" ×).

---

## Validation 카테고리 (한눈에)

| 카테고리 | 의미 |
|---|---|
| `MUST_PASS` | 릴리즈 전 반드시 통과해야 하는 요구사항 |
| `KNOWN_ISSUE` | 알려진 한계 — 현재는 허용 가능하다고 판단 |
| `DEFERRED` | 이번 iteration 범위 밖으로 밀어둔 항목 |

카테고리 **승격 권한은 사람에게만** 있다. 사람이 `KNOWN_ISSUE → MUST_PASS` 같이 승격하면 LLD skeleton / ADR 단계로 돌아가 iteration 루프가 돈다. 모든 항목이 `MUST_PASS 통과` / `KNOWN_ISSUE 수락` / `DEFERRED 범위 밖 확인` 중 하나로 정리되면 iteration 종료.

상세는 [`CLAUDE.md`](./CLAUDE.md)의 단계 5~8.

---

## 이 저장소에 포함되지 **않는** 것

- 실제 프로젝트의 비즈니스 컨텍스트, 코드, 스택 선택 — 이 저장소를 가져다 쓰는 프로젝트가 `context/`, `reference/`에 채운다.
- 구현 단계의 코딩 규칙 — 이 저장소를 가져다 쓰는 프로젝트의 **루트** `CLAUDE.md`의 관심사다 (`docs/CLAUDE.md`가 아니다).
- 패키지 구조·레이어 규칙·architecture test — LLD의 관심사가 아니다.
