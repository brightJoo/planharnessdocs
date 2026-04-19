---
Topic: kafka-producer
Type: portfolio-reference
LastUpdated: 2026-04-19
KafkaVersion: "4.2"
Source: https://kafka.apache.org/42/configuration/producer-configs/
---

# Reference — Kafka Producer

> **트래픽 수집 라이브러리가 삽입된 ~5,500 Pod에서 돌아가는 Producer의 설정 원칙.** Kafka **4.2**(현재 stable) 기준. 각 파라미터는 *"Kafka 기본값 → 이 프로젝트 값 → 왜"* 형태로 제시한다.

---

## 1. 이 프로젝트에서 Producer의 위치

- **어디서 돌아가는가**: 대상 서비스(Service A/B/C)의 각 Pod 내부. 트래픽 수집 라이브러리 모듈로 동작.
- **핵심 제약**
  - Host 서비스(대상 서비스)의 **메모리/CPU/지연에 "유의미하지 않은" 영향**만 허용 (context §4.1, K-1).
  - 수집 Off 구간에서는 **네트워크/컴퓨팅 비용 0** (Kafka client가 idle 또는 not-initialized).
  - 일부 레코드 손실은 허용 — 부하 "재현" 용도지 회계용 로그가 아님.

## 2. 설계 철학

| # | 원칙 | 의미 |
|---|---|---|
| 1 | **Host-first, fail-soft** | Kafka 문제가 host 서비스에 전파돼선 안 된다. `send()`는 블록 금지, 버퍼 full이면 drop. |
| 2 | **Loss-tolerant, not loss-free** | 일부 레코드 손실은 허용. at-most-once에 준하는 선택이 가능. |
| 3 | **Throughput over latency** | 10~100 ms 단위 송신 지연은 수용 가능. 대신 배치 효율·압축률 최대화. |
| 4 | **Low host footprint** | 버퍼 메모리·스레드·커넥션 수를 host 예산에 맞춰 작게 잡는다. |
| 5 | **Observable loss** | drop된 레코드 수·buffer exhaustion 횟수를 메트릭으로 노출. 손실이 블랙박스여선 안 된다. |

## 3. 파라미터 — 핵심 설정

| Parameter | Kafka 4.2 기본값 | 이 프로젝트 값 | 이유 |
|---|---|---|---|
| `acks` | `all` | **`1`** | 리더만 ack하면 충분 — loss-tolerant. `all`은 replica 전체 대기로 latency + 브로커 부하↑. (loss 허용폭 더 큰 환경이면 `0`까지 낮춤 가능) |
| `enable.idempotence` | `true` | **`false`** | `acks != all`이면 자동으로 disable. at-most-once 가까운 수집용에 중복 방지 비용 불필요. |
| `compression.type` | `none` | **`zstd`** | HTTP 트래픽 본문은 압축률 매우 높음. 5,500 Pod × RPS 대역폭이 주 병목. host CPU 예산 타이트하면 `lz4`로 대체. |
| `linger.ms` | `5` | **`50`** | 배치 효율↑. 수집 latency는 SLA 아님. |
| `batch.size` | `16384` (16 KiB) | **`65536`** (64 KiB) | 큰 배치 → 압축·네트워크 효율. `linger.ms`와 짝. |
| `buffer.memory` | `33554432` (32 MiB) | **`8388608`** (8 MiB) | Host 서비스 메모리 예산 보호. "유의미하지 않은 풋프린트" 제약. |
| `max.block.ms` | `60000` (1 min) | **`0`** | ★ 핵심. 버퍼 full 시 host를 **절대 블록하지 않는다.** 즉시 예외 → 라이브러리가 catch하고 drop metric 증가시킨 뒤 조용히 반환. |
| `delivery.timeout.ms` | `120000` (2 min) | **`30000`** (30 s) | Fail-fast. 2분 재시도 매달릴 여유 없음. |
| `request.timeout.ms` | `30000` (30 s) | **`10000`** (10 s) | 동일 사유. |
| `max.in.flight.requests.per.connection` | `5` | 기본값 유지 | idempotence 꺼도 5 OK. 파티션 내 순서 보장은 이 use-case에 불필요(수집용 재정렬 무방). |
| `retries` | `2147483647` | 기본값 유지 | `delivery.timeout.ms`로 상한. |
| `max.request.size` | `1048576` (1 MiB) | **`4194304`** (4 MiB) | 큰 HTTP 바디 커버. 대신 라이브러리에서 **레코드 크기 상한 사전 필터**(초과는 drop + metric) 병행. |
| `client.id` | `""` | **`<service>-<pod>`** | 서버측 추적·쿼터 분리. Pod identity 포함. |

## 4. 부가 파라미터 (참고·유지)

| Parameter | Kafka 4.2 기본값 | 주의 |
|---|---|---|
| `metadata.max.age.ms` | `300000` (5 min) | 기본값 유지. |
| `reconnect.backoff.max.ms` | `1000` | 기본값 유지 — 20% jitter 내장. |
| `partitioner.adaptive.partitioning.enable` | `true` | 느린 브로커 자동 회피. 유지. |
| `connections.max.idle.ms` | `540000` (9 min) | 기본값 유지. |
| `transactional.id` | `null` | **설정 금지.** 이 프로젝트는 EOS 불필요, 트랜잭션 비용만 추가. |
| `key.serializer`/`value.serializer` | (required) | 라이브러리에서 명시. 직렬화 포맷은 별도 설계에서. |

## 5. 원칙 위반 신호 (설계·코드 리뷰 체크리스트)

- [ ] `max.block.ms > 0` → Host-first 원칙 위반 가능성 검토 필요
- [ ] `compression.type == none` → 대역폭 원칙 위반 (반드시 이유 기록)
- [ ] `acks == all` → loss-tolerant와 충돌 (왜 필요한지 근거 명시 필수)
- [ ] drop metric이 없음 → Observable loss 원칙 위반
- [ ] `buffer.memory` 기본값 그대로 → host 메모리 예산 검토 안 한 것

## 6. 출처 & 버전 주의

- Apache Kafka 4.2 Producer Configs: https://kafka.apache.org/42/configuration/producer-configs/
- **최근 변경된 기본값**(예전 가이드 참조 시 주의):
  - `acks`: `1` → `all` (Kafka 3.0)
  - `enable.idempotence`: `false` → `true` (Kafka 3.0)
  - `linger.ms`: `0` → `5` (Kafka 4.0)
  - `metadata.recovery.strategy`: `rebootstrap` (4.x)
