---
Topic: kafka-consumer
Type: portfolio-reference
LastUpdated: 2026-04-19
KafkaVersion: "4.2"
Source: https://kafka.apache.org/42/configuration/consumer-configs/
---

# Reference — Kafka Consumer

> **수집된 트래픽 레코드를 Object Storage로 배치 적재하는 Consumer의 설정 원칙.** Kafka **4.2** 기준. Producer reference(`produce.md`)와 쌍으로 본다.

---

## 1. 이 프로젝트에서 Consumer의 위치

- **어디서 돌아가는가**: 별도 수집 백엔드 서비스 (대상 서비스 Pod와 **분리**). 컨슈머 그룹으로 수평 확장.
- **하는 일**: Kafka 토픽 → 배치 누적 → Object Storage 업로드 → 업로드 성공 후 오프셋 커밋.
- **핵심 제약**
  - Object Storage 업로드가 **배치 단위로 느림** (수백 ms ~ 수 s).
  - 중복 허용(at-least-once). 리플레이는 중복 레코드에 민감하지 않음.
  - Consumer 인스턴스 수는 동적 — 스케일링 중 rebalance 영향 최소화 필요.

## 2. 설계 철학

| # | 원칙 | 의미 |
|---|---|---|
| 1 | **Process-then-commit** | Object Storage 업로드 성공 후에만 오프셋 커밋. auto-commit 금지. |
| 2 | **At-least-once OK** | 중복은 허용. downstream 또는 리플레이가 감당함. |
| 3 | **Batch-oriented fetch** | 실시간 아님. `fetch.min.bytes`·`max.poll.records`를 키움. |
| 4 | **Rebalance 최소화** | Static membership + cooperative sticky. 스케일업/다운·재기동이 전체 중단으로 이어지지 않게. |
| 5 | **Slow processing tolerance** | `max.poll.interval.ms`에 업로드 시간 + 재시도 여유 포함. |

## 3. 파라미터 — 핵심 설정

| Parameter | Kafka 4.2 기본값 | 이 프로젝트 값 | 이유 |
|---|---|---|---|
| `group.id` | `null` (필수) | **`traffic-recorder-<env>`** | 환경별 분리. |
| `auto.offset.reset` | `latest` | **`earliest`** | 첫 배포 시 기존 쌓인 데이터부터 수용. `latest`는 누락 위험. (첫 배포 혼선 방지가 더 중요한 환경이면 `latest`) |
| `enable.auto.commit` | `true` | **`false`** | ★ process-then-commit. 업로드 성공 후 수동 commit. auto면 업로드 실패 시에도 커밋 → 데이터 유실. |
| `max.poll.records` | `500` | **`2000`** | 배치 크기 확보. 레코드 크기 분포 실측 후 조정 가능. |
| `max.poll.interval.ms` | `300000` (5 min) | **`900000`** (15 min) | 업로드 지연 + 재시도 포함해 여유. poll 공백을 넉넉히. |
| `session.timeout.ms` | `45000` (45 s) | 기본값 유지 | Kafka 3.0+에서 현실화된 값. |
| `heartbeat.interval.ms` | `3000` (3 s) | 기본값 유지 | session의 ~1/15, 표준 관행. |
| `fetch.min.bytes` | `1` | **`1048576`** (1 MiB) | 브로커가 최소 1 MiB 모일 때까지 대기 후 응답 — 배치 효율. |
| `fetch.max.wait.ms` | `500` | **`5000`** (5 s) | `fetch.min.bytes`를 기다리는 상한. 5 s는 수집용에 충분. |
| `fetch.max.bytes` | `52428800` (50 MiB) | 기본값 유지 | 충분. |
| `max.partition.fetch.bytes` | `1048576` (1 MiB) | **`5242880`** (5 MiB) | 큰 HTTP 바디가 한 파티션에 몰려도 수용. producer의 `max.request.size=4 MiB`와 짝. |
| `isolation.level` | `read_uncommitted` | 기본값 유지 | Producer가 transaction 미사용 → 사실상 동일. |
| `partition.assignment.strategy` | `[RangeAssignor, CooperativeStickyAssignor]` | **`[CooperativeStickyAssignor]`** | Rebalance 시 incremental — 스케일링 중 전체 중단 방지. |
| `group.instance.id` | `null` | **`<env>-<ordinal>`** (StatefulSet pod ordinal 등) | **Static membership** — 일시 재기동에 rebalance 안 유발. |
| `client.id` | `""` | **`traffic-recorder-<ordinal>`** | 브로커측 추적·쿼터. |

## 4. 부가 파라미터 (참고·변경)

| Parameter | Kafka 4.2 기본값 | 주의 |
|---|---|---|
| `group.protocol` | `classic` | 신 프로토콜 `consumer` (KIP-848, 4.0 GA)는 별도 전환 검토 대상. 현재는 classic 유지. |
| `auto.commit.interval.ms` | `5000` | auto-commit 꺼뒀으므로 의미 없음. |
| `allow.auto.create.topics` | `true` | **`false` 권장** — 운영 토픽은 인프라팀이 관리. |
| `check.crcs` | `true` | 기본값 유지. 작은 CPU 비용으로 corruption 차단. |
| `request.timeout.ms` | `30000` (30 s) | 기본값 유지. |
| `receive.buffer.bytes` | `65536` (64 KiB) | **`1048576`** (1 MiB) 권장 — 고 throughput fetch에 TCP 버퍼 상향. |
| `connections.max.idle.ms` | `540000` (9 min) | 기본값 유지. |

## 5. 원칙 위반 신호 (설계·코드 리뷰 체크리스트)

- [ ] `enable.auto.commit == true` → Process-then-commit 위반
- [ ] `partition.assignment.strategy`가 기본값 그대로 → rebalance 중 전면 중단 위험 (Cooperative Sticky 미채택)
- [ ] `group.instance.id` 미설정 → 일상적 재기동이 rebalance 유발
- [ ] `max.poll.interval.ms == 기본값(5 min)` → 긴 업로드 중 consumer 강퇴 위험
- [ ] `fetch.min.bytes == 1` → 배치 철학과 충돌, 브로커 왕복 과다

## 6. 출처 & 버전 주의

- Apache Kafka 4.2 Consumer Configs: https://kafka.apache.org/42/configuration/consumer-configs/
- **최근 변경된 기본값**:
  - `session.timeout.ms`: `10000` → `45000` (Kafka 3.0, KIP-735)
  - `partition.assignment.strategy`: `CooperativeStickyAssignor` 디폴트 포함 (3.0, KIP-726)
  - `group.protocol=consumer` (KIP-848) GA: Kafka 4.0 (기본값은 `classic`)
