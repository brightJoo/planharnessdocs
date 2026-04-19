---
Topic: caching
Type: portfolio-reference
LastUpdated: 2026-04-19
---

# Reference — Caching

> **시스템 전체가 공유하는 캐싱 원칙.** 트래픽 레코딩/리플레이 플랫폼은 **수천 대 규모(최대 ~5,500) Pod** 환경에서 동작하므로, 모든 설계가 *"중앙 DB를 매 요청마다 때리지 않는다"* 는 제약을 전제로 한다. 이 문서는 그 제약을 어떤 층위로 분해하고 어떤 공통 원칙으로 관통하는지 정의한다.

---

## 1. 왜 다층 캐시인가

### 1.1 제약
- Pod 규모: **수천 대 (최대 ~5,500)**. 라이브러리가 삽입된 각 Pod는 **매 요청마다** 수집 on/off 상태를 참조한다.
- 전용 인프라 신규 할당은 이번 사이클에 불가 → **공유 Redis**와 **중앙 RDB(PostgreSQL)** 에 얹혀야 한다.
- 중앙 API 한 대가 5,500 Pod를 직접 상대하면 병목: 30초 주기 polling만으로도 평균 **~183 RPS**, 피크는 같은 초에 집중 가능.

### 1.2 한 층만 쓰면 무너지는 이유

| 시나리오 | 문제 |
|---|---|
| Pod가 매 요청마다 PostgreSQL 직접 조회 | 커넥션 풀 폭발, 트래픽마다 ms 단위 지연. 현실적으로 불가. |
| Pod가 Shared Redis 직접 조회 | 요청마다 network hop. 공유 자원이라 hot path RPS를 감당할 수 없음. |
| Pod → 중앙 API (캐시 없이) | 중앙 API가 곧 PostgreSQL의 RPS 증폭기가 됨. |

### 1.3 해결: 3층

1. **Pod-local cache** — 매 요청에서 참조되는 **hot path** 담당.
2. **Central-local cache** — PostgreSQL을 직접 때리지 않도록 중앙 API 프로세스 내 방어막.
3. **Shared Redis** — **여러 Pod가 공유해야만 하는** 집계·큐잉 상태 전용.

각 층은 **서로 다른 책임**을 지며, 트래픽 경로가 위에서 아래로 갈수록 **RPS가 급격히 줄어든다.**

---

## 2. 세 층의 역할 분리

```
  매 요청 →  Pod-local Cache       (각 Pod in-memory)        [수천 × 요청 RPS]
              │
              │ 30초 jittered pull
              ↓
  초~분 →   Central-local Cache    (중앙 API in-process)     [≈ 183 RPS]
              │
              │ 수 초 TTL · refresh-ahead
              ↓
  분 →      PostgreSQL             (source of truth)         [수 RPS, refresh만]


  cross-Pod  Shared Redis           (집계·큐 전용, hot path 금지)
```

| 층 | 목적 | 허용 | 금지 |
|---|---|---|---|
| Pod-local | 매 요청 hot path | 단일 Pod 상태 캐싱 | cross-Pod 공유 상태 |
| Central-local | PostgreSQL 보호 | 수 초 TTL 응답 메모이제이션 | 장기 보존 (프로세스 소멸과 함께 소실) |
| Shared Redis | Pod 간 집계·큐 | 매트릭 합산, 작업 큐, Pod 간 합의 상태 | **hot path 조회** (트래픽당 설정 조회) |

---

## 3. 공통 원칙 (모든 층에 적용)

| # | 원칙 | 의미 |
|---|---|---|
| 1 | **Staleness-tolerant by design** | 일정 시간(층마다 다름) 낡은 데이터로 동작하는 것을 **설계상 허용**. 즉시 반영 보장을 하지 않는다. |
| 2 | **Fail-safe defaults** | 캐시 miss / 원본 실패 시, **의심스러우면 안전한 쪽**. 수집 플랫폼에서는 off가 안전값. |
| 3 | **Jittered refresh** | 주기 갱신은 항상 **pod/instance별 랜덤 오프셋**을 가진다. 5,500 Pod가 같은 초에 중앙을 때리지 않도록. |
| 4 | **Observable staleness** | 각 캐시는 *"마지막 성공 갱신 시각"* 을 메트릭으로 노출. 얼마나 낡은 데이터로 동작 중인지 항상 관측 가능해야 한다. |
| 5 | **Refresh-ahead, not on-demand** | TTL 만료 순간 요청 경로가 블록되지 않는다. 만료 **전에 백그라운드**가 갱신한다. |
| 6 | **Pull, not push** | 중앙이 각 Pod를 개별 조작하지 않는다. 각 층이 자기 주기로 상위 층을 **가져간다**. (5,500 Pod에 push 장기 유지하는 설계는 이 프로젝트 범위 밖) |

> **이 원칙이 API 계약에 반영된다.** 예:
> - 중앙 API 응답에는 **"이 응답이 서빙된 시각"** 이 포함되어야 *observable staleness*가 성립.
> - 응답 포맷은 *refresh-ahead*에 적합해야 한다 (큰 전수 응답은 백그라운드 갱신 비용을 키운다).

---

## 4. 층별 정책

### 4.1 Pod-local cache

| 항목 | 정책 |
|---|---|
| 저장 위치 | 프로세스 in-memory (단일 Pod 내) |
| 주 사용처 | 서비스별 수집 on/off, API 단위 블랙/화이트리스트 |
| 갱신 주기 | **30초 주기 polling** (Pod별 jitter ±수 초) |
| 갱신 방향 | 중앙 API에서 pull |
| 초기 상태 | **off** (fail-safe). 첫 polling 성공 전까지는 수집하지 않는다. |
| Polling 실패 | **이전 값 유지** + exponential backoff 재시도 |
| Staleness 임계 초과 | **off로 fallback.** |
| Pod 재시작 | 메모리 소실 → 다시 fail-safe off부터 시작 |
| 필수 메트릭 | `cache_age_seconds`, `poll_success_ratio`, `poll_rtt_ms` |

**라이브러리 ↔ 소비자 계약.** 라이브러리 사용자(대상 서비스)는 *"이 상태는 최대 ~30초 낡을 수 있다"* 는 가정 하에 이 층을 쓴다. 즉시성이 필요한 제어(긴급 전면 off 등)는 이 층에서 지원하지 않는다 — 필요하다면 별도 경로로 설계한다.

### 4.2 Central-local cache

| 항목 | 정책 |
|---|---|
| 저장 위치 | 중앙 API 프로세스 in-memory |
| 주 사용처 | 5,500 Pod의 polling 응답에 쓰일 최신 상태 |
| TTL | 수 초 단위 (환경 부하 실측 후 튜닝) |
| 갱신 방식 | TTL 만료 직전 백그라운드 refresh (*refresh-ahead*) |
| 원본(PostgreSQL) 접근 | **refresh 시점에만.** Pod polling이 DB에 직접 닿지 않는다. |
| 다중 인스턴스 | Central API가 N개일 때 각 인스턴스 독립 캐시 → **eventual consistency** 허용 |
| 필수 메트릭 | `central_cache_age_seconds`, `db_read_rps` (refresh 트리거 RPS) |

**Pod ↔ 중앙 API 계약.** Pod가 polling 시 받는 응답은 최대 **(central TTL + Pod polling interval)** 만큼 낡을 수 있다. 설정 변경자(플랫폼 운영자)는 이 전파 지연을 인지한 상태로 변경한다.

### 4.3 Shared Redis

| 항목 | 정책 |
|---|---|
| 저장 위치 | 조직 공유 Redis (전용 인스턴스 불가) |
| 용도 (**허용**) | 리플레이 매트릭 합산, 작업 큐, Pod 간 합의가 필요한 집계 상태 |
| 용도 (**금지**) | 수집 on/off 같은 **hot path 조회** — 반드시 Pod-local을 통한다 |
| Key namespace | `<project>:<domain>:<key>` 규약 (충돌 방지) |
| TTL | **명시 필수.** TTL 없는 키 쓰기 금지. |
| 크기 제약 | 공유 자원 — 단건 크기·총 메모리 사용 추정치를 HLD에서 계획에 포함 |
| 실패 시 | **로컬 메모리로 degrade** (context §4.5 "실패 시 로컬 메모리" 원칙) |

**저장소 ↔ 소비자 계약.** Shared Redis를 읽기·쓰기 경로에 쓰는 모든 컴포넌트는 **Redis 없이도 degrade 동작**이 가능해야 한다 (리플레이 매트릭은 일시 정밀도 저하, 큐는 재처리 자연스러운 설계 등으로).

