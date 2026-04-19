---
Topic: mock-users-and-k6
Type: portfolio-reference
LastUpdated: 2026-04-19
Source:
  - k6 Scenarios: https://grafana.com/docs/k6/latest/using-k6/scenarios/
  - k6 SharedArray: https://grafana.com/docs/k6/latest/javascript-api/k6-data/sharedarray/
  - context §4.3 (인증 토큰 취급)
---

# Reference — Mock Users × k6 (리플레이 인증 주입)

> **문제.** 리플레이 요청이 그대로 나가면 (a) 캡처된 토큰이 만료되어 401로 되돌아오거나, (b) 민감해서 테스트 환경으로 옮길 수 없거나, (c) 마스킹 단계에서 아예 제거되어 비어 있다. 401만 오면 서버는 "죽지 않고" 부하 테스트 목적을 달성하지 못한다 (context §4.3).
>
> **해결.** 재생 시점에 **합성된 mock user 풀**에서 유효 토큰을 뽑아 `Authorization`/`Cookie`를 교체한다. 이 문서는 그 풀을 어떻게 만들고 k6에 어떻게 먹일지의 **공통 원칙**을 정의한다.
>
> **이중 포지셔닝.**
> - *포트폴리오 관점*: 실제 회원 데이터를 쓸 수 없으므로 mock 풀로 전체 흐름을 시연.
> - *제품 관점*: 실 비프로덕션 환경에서도 부하 테스트 시 mock 풀이 기본 경로 (실 토큰 사용은 예외).

---

## 1. 설계 철학

| # | 원칙 | 의미 |
|---|---|---|
| 1 | **No-real-PII by design** | 실제 회원 데이터를 복제·샘플링·변형하지 않는다. 잔여 PII 위험 0. |
| 2 | **Plausible distribution** | tier·segment·디바이스 분포는 **현실 근사**. 동일 비중으로 뽑지 않는다 (부하 편향의 원인). |
| 3 | **Reusable pool, bounded size** | 같은 풀을 여러 시나리오·스테이지가 공유. VU 수 늘어도 풀을 선형 증설하지 않는다. |
| 4 | **Stateless lookup** | VU가 자기 user를 **O(1)** 로 조회. 락·원격 호출 금지. |
| 5 | **Refreshable mid-run** | 장시간 실행 중 토큰 만료 발생 시 **재발급 경로 필수**. 풀 교체도 가능. |
| 6 | **Observable auth path** | 인증 실패율·풀 고갈·refresh 실패를 메트릭으로 노출. 인증 실패가 "로직 실패"로 오인되면 결과 분석이 전부 왜곡된다. |
| 7 | **Environment-isolated** | Mock 토큰이 **production auth 서버**에 도달하는 일이 없게 환경 분리. secret·키는 테스트 환경 전용. |

---

## 2. Mock user 스키마 (예시)

> 실제 필드셋은 대상 서비스의 JWT claim / 세션 모델에 맞춰 정한다 (ADR A-1에서 확정). 아래는 최소 골격.

| 필드 | 타입 | 생성 규칙 |
|---|---|---|
| `user_id` | string | UUIDv4, 풀 내 unique |
| `tier` | enum | `free` / `premium` / `vip` — 분포 예: 70/25/5 (실 분포 근사, 수치는 ADR) |
| `segment` | enum | 디바이스·지역 태그 — 실 분포 근사 |
| `created_at` | ISO8601 | 최근 N년 내 균등 분포 |
| `locale` | string | `ko-KR` / `en-US` 등 — 분포 근사 |

**금지**
- 실명·이메일·전화번호 — 필요하다면 `faker` 등으로 **완전 합성**. 실 데이터 기반 마스킹 금지.
- 실제 회원 ID 공간과 겹치는 ID — prefix(`mock_`)로 분리 권장.

---

## 3. 토큰 발급 모델 (옵션)

| 옵션 | 구조 | 장점 | 단점 |
|---|---|---|---|
| **A. 테스트 환경 HS256 JWT** | claim에 `user_id`/`tier` 등 담고 테스트 전용 secret으로 서명 | 오프라인 발급 가능, 서버 의존성 無 | 서버가 **테스트용 secret을 수용**해야 함 (별도 config) |
| **B. Opaque token + 테스트 introspection** | 불투명 토큰 + 테스트 전용 introspection endpoint | 실 production 흐름에 가까움 | introspection endpoint 운영 필요 |
| **C. 실 인증 서버의 test mode** | 인증 서버가 비프로덕션 키로 mock 토큰 발급 | 가장 현실적 | 인증 서버 협조 필요(context §4.3 경로 (a)와 유사) |

환경 조건(테스트 서버가 testing secret을 수용하는가, introspection endpoint를 띄울 수 있는가 등)에 따라 선택. 포트폴리오 시연 맥락에서는 **A** 가 가장 단순하다.

---

## 4. k6 연동 패턴

### 4.1 Pool 사전 생성 → SharedArray 로드

```
[외부 스크립트]             [k6 스크립트 setup]              [VU]
  generate.ts                 SharedArray('users',             users[i]
  ──────────▶  users.json      () => JSON.parse(open('...')))  O(1) 조회
  N명 합성                       ──────────▶                  ──────────▶
  ID/tier/segment
  토큰 pre-issue (옵션)
```

- **SharedArray**: k6에서 큰 데이터셋을 VU 간 **메모리 공유**로 올리는 표준 수단. VU 당 복사본 X.
- **Pool 크기 N**: 시나리오당 동시 VU × 수배가 현실적 관행. 정확한 배수는 부하·refresh 주기에 따라 튜닝.

### 4.2 VU ↔ user 매핑 전략

| 전략 | 식 | 현실감 | 세션/캐시 재사용 |
|---|---|---|---|
| **per-VU** | `users[__VU % N]` | 낮음 (VU가 한 user로만) | ★ 실제 세션에 가깝다 |
| **per-iteration (round-robin)** | `users[__ITER % N]` | 중간 | 낮음 |
| **per-iteration (random)** | `users[randomInt(0, N)]` | 높음 | 없음 |

→ 시나리오 목적에 맞게 선택. 세션/캐시 재사용이 성능 특성을 좌우하면 per-VU, 다양한 사용자 분포를 보려면 per-iteration random.

### 4.3 토큰 주입 (리플레이 엔진 측)

캡처된 요청의 헤더를 재생 직전에 치환한다:

```
captured.Authorization  →  (drop / overwrite with mock[user_id])
captured.Cookie[session] →  (drop / overwrite with mock session)
captured.X-User-Id       →  (overwrite with mock user_id)
```

- 치환 책임은 **리플레이 엔진**에 있다 (k6 스크립트가 아님). 엔진이 request mutation 단계에서 수행.
- k6는 치환된 요청을 **그대로 발사**. 추가 인증 로직 없음.

### 4.4 토큰 refresh

| 조건 | 동작 |
|---|---|
| 정상 발사 | mock pool의 미리 발급된 토큰 사용 |
| 토큰 만료 근접 | 배경으로 refresh (풀에서 교체 또는 재발급 호출) |
| Refresh 실패 | 해당 user를 **차단 목록**에 넣고 다음 요청은 **다른 user로** 진행. 전체 실패 전환은 임계 초과 시에만. |
| Pool 고갈 (차단 누적) | 시나리오 **조기 종료 + 메트릭 알람** (리플레이 결과 신뢰 불가 판정) |

---

## 5. 환경 격리 (필수)

- Mock 토큰을 **production auth 서버가 거부**하도록 환경 분리. 네트워크 / 키 / JWT `iss` claim 중 최소 하나로 구분.
- 테스트 환경의 auth 서버만이 mock secret·test introspection을 안다.
- Replay 엔진·k6 러너는 **production 엔드포인트로 붙지 않는다.** 타깃 URL은 환경 변수로만 주입, 기본값은 테스트 환경.

---

## 6. 관측성 (필수 메트릭)

| 메트릭 | 의미 |
|---|---|
| `mock_user_pool_size` | 현재 활성 풀 크기 |
| `mock_user_pool_exhaustion_total` | 차단/만료로 사용 가능 user가 0에 근접한 횟수 |
| `auth_reject_rate` | 리플레이 응답 중 401/403 비율 — **풀 품질 지표** |
| `token_refresh_total` | refresh 시도 건수 |
| `token_refresh_failure_total` | refresh 실패 건수 |
| `mock_user_assignment_distribution` | user별 할당 횟수 히스토그램 (편향 검증) |

**해석 규칙**
- `auth_reject_rate`가 일정 수준을 넘으면 리플레이 결과의 다른 지표(에러율·지연)를 **신뢰하지 않는다.** 풀이 이미 무효 상태에서 얻은 지표이기 때문. 임계값은 validation 기준과 함께 정의한다.

---

## 7. 금지 사항

- ❌ 실 회원 데이터를 스크럽/마스킹해서 mock 풀에 투입 — 잔여 PII 위험
- ❌ Mock secret을 저장소 코드에 하드코딩 — env/secrets manager 경유
- ❌ Production 인증 서버에 mock 토큰 발사 경로 허용 — 환경 분리 원칙 위반
- ❌ Pool을 VU 당 복제 (SharedArray 미사용) — 메모리 폭발, k6 기본 실수
- ❌ `auth_reject_rate` 무시하고 부하 결과 해석 — 풀이 망가졌는지 모른 채 결론 도출

---

## 8. 원칙 위반 신호 (리뷰 체크리스트)

- [ ] Mock pool 데이터에 실 회원의 흔적(ID 공간·이메일 도메인·가입일 분포 등) 의심
- [ ] `SharedArray` 미사용 → 메모리 낭비·VU 시작 지연
- [ ] Replay 엔진이 치환 안 하고 k6에 생(raw) 캡처 던짐 → 401 폭주
- [ ] `auth_reject_rate` 메트릭 미노출 → 실패를 못 본 상태로 결과 해석
- [ ] Mock secret이 저장소 commit에 존재 → 보안 사고 후보
- [ ] Production 엔드포인트가 기본값 → 환경 격리 원칙 위반

---

## 9. 출처

- k6 Scenarios: https://grafana.com/docs/k6/latest/using-k6/scenarios/
- k6 SharedArray: https://grafana.com/docs/k6/latest/javascript-api/k6-data/sharedarray/
- k6 Execution Context (`__VU`, `__ITER`): https://grafana.com/docs/k6/latest/using-k6/execution-context-variables/
- context §4.3 — 토큰 확보 경로·보안팀 가이드 의존
