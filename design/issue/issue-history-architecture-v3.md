# Issue History 아키텍처 설계 (v3)

## 0. 한 줄 요약

threshold가 이슈 검증 + State Diff(OpenSearch 기반) + OpenSearch 영속(1-row update)까지 완료한 후 도메인 이벤트 발행(Kafka) → alert이 이벤트 소비하여 알림 발송 → Issue History는 OpenSearch 단순 search로 조회, Summary는 OpenSearch aggregation, 단일 쿼리로 리스트+통계 일관성 확보.

---

## 0.0 OpenSearch 사용 전제

| 항목 | 내용 |
|------|------|
| 서비스 | AWS OpenSearch Service (Managed) |
| 인프라 구축 | 본 설계 scope 밖. 클러스터 생성/설정/운영은 인프라 팀 담당. |
| 본 문서 범위 | 인덱스 설계, 매핑, 쿼리 패턴, ISM 정책 등 **애플리케이션 사용 관점**만 기술 |
| API 호환성 | OpenSearch는 Elasticsearch 7.10 OSS 포크. 본 설계의 쿼리/매핑은 OpenSearch 2.x 호환 기준으로 작성 |
| 클라이언트 | `@opensearch-project/opensearch` (OpenSearch JS Client) 사용 |
| 참고 | ILM → OpenSearch에서는 **ISM (Index State Management)** 용어 사용. 본 문서에서는 ISM으로 통일 |

---

## 0.1 1-Row Update 채택 근거

### 이 시스템의 특수성

- 각 incident는 **Open 1회 → Resolve 1회 → 끝**. Reopen 없음. 이후 상태 변경 없음.
- 하나의 OpenSearch 문서는 생애 동안 **최대 1번만 update**됨.
- threshold가 device_id 파티션 기반 순차 처리 → **동시 쓰기 구조적으로 불가능**.
- OpenSearch update의 전통적 문제점(segment fragmentation, version conflict)이 이 패턴에서는 크리티컬하지 않음.

### 1-Row Update의 이점

1. **조회 단순성**: collapse 불필요. 단순 search로 1 doc = 1 incident.
2. **Status 필터 직접 지원**: `{ "term": { "status": "open" } }` 바로 가능.
3. **리스트+통계 단일 쿼리**: search + aggs가 동일 문서 집합 기준.
4. **저장 효율**: 인덱스 크기 절반 (문서 수 N vs 2N).
5. **프론트엔드 단순**: flat 응답, collapse 파싱 불필요.

상세 1 row update vs 2 row append only 문서: src/design/issue/es-storage-strategy-final-comparison.md 참고.

### OpenSearch 호환성 확인

| 기능 | OpenSearch 2.x 지원 여부 |
|------|--------------------------|
| `_update` API (partial doc) | ✅ 지원 |
| `_routing` | ✅ 지원 |
| `search_after` pagination | ✅ 지원 |
| `terms` / `date_histogram` aggs | ✅ 지원 |
| `prefix` query | ✅ 지원 |
| ISM (Index State Management) | ✅ 지원 |

---

## 1. 규모 산정

| 항목 | 값 |
|------|------|
| 디바이스 수 | ~100,000대 |
| 리포트 주기 | 2분 |
| 리포트당 최대 이슈 수 | ~20개 |
| 초당 리포트 수 | 100,000 / 120초 ≈ 833 msg/s |
| 초당 최대 인시던트 변경 | 833 × 20 = ~16,600 ops/s (최악) |
| 배치 이슈 (SERVER) | 5분 주기, 상대적으로 적음 |
| Open 인시던트 (동시) | 디바이스당 평균 2~3개 → 최대 ~300K rows |
| Resolved 인시던트 (누적) | 월 수천만 건, 연 수억 건 |
| workspace 수 | 수백~수천 개 |
| workspace당 디바이스 | 대부분 수십~수백 대, 최대 수천 대 |
| 최대 조회 기간 | 1년 |
| workspace당 1년 누적 (최대) | ~320만 건 |

---

## 2. 아키텍처 개요

### 전체 구조

```
[msl.device.status]  ──→  threshold (EKS)
[dm.batch.issue.created] ──→  │
[dm.device.group.changed] ──→ │  ← (신규) group 변경 이벤트
                               │
                               │  1. 이슈 검증 + State Diff (OpenSearch status=open 조회)
                               │  2. OpenSearch incidents 영속 (INDEX / UPDATE)
                               │  3. dm.incident.* 이벤트 발행 (write 완료 후)
                               │
                               ├──→ dm.incident.new ──→ alert (알림 발송)
                               └──→ dm.incident.resolved ──→ alert (알림 발송)

                                         ┌──────────┐
                                         │OpenSearch│
                                         │ Incid.  │
                                         │ (1row)  │
                                         └──────────┘
                                              ▲
                                              │
                                         Issue History (단순 search)
                                         + Current Issues (status=open)
                                         + Summary 통계
                                         + State Diff (threshold)
                                         + Device Detail Open Issue

[device-read Pizza] ←── ServiceInvoker ── monitoring (OpenSearch 조회)
```

### 핵심 원칙

1. **OpenSearch가 유일한 저장소** — State Diff, 조회, 통계 모두 OpenSearch 하나로 처리. PG 사용 안 함.
2. **Write-then-Publish** — threshold가 OpenSearch 영속 완료 후에만 이벤트 발행.
3. **OpenSearch는 1-row update** — incident당 1문서. Open 시 INDEX, Resolve 시 UPDATE.
4. **단순 search** — collapse 불필요. status 필드로 직접 필터.
5. **Group 변경 시 resolve + re-open** — 기존 incident resolve 후 새 그룹에서 새 incident 생성.
6. **Deactivated, Deleted, Activation Off 설정 시 resolve 처리** — 기존 incident resolve 처리.
7. **instop, 앱삭제, reset** — 기존 incident resolve 처리.

### Write-then-Publish 패턴

```
threshold 처리 순서:
1. State Diff → NEW / EXISTING / RESOLVED 판정 (OpenSearch status=open 조회)
2. OpenSearch incidents 영속 (INDEX / UPDATE)
3. OpenSearch 영속 성공 → dm.incident.new / dm.incident.resolved 이벤트 publish
4. alert이 이벤트 소비 → 알림 발송
```

---

## 3. threshold 모듈

### 역할

| 항목 | 설명 |
|------|------|
| State Diff | OpenSearch `device_id + status=open` 조회 → 리포트와 비교 → NEW/EXISTING/RESOLVED 판정 |
| OpenSearch 쓰기 | incidents INDEX (Open) / UPDATE (Resolve) |
| 이벤트 발행 | dm.incident.new / dm.incident.resolved (write 완료 후) |
| Group 변경 처리 | dm.device.group.changed 수신 → resolve + re-open |

### issue_instance_id 생성 규칙

- NEW 판정 시 UUID 생성 → OpenSearch 문서 ID로 사용
- RESOLVED 판정 시 해당 issue_instance_id로 OpenSearch UPDATE
- 동일 이슈의 OPEN → RESOLVE lifecycle을 하나의 ID로 추적

### threshold의 영속 처리

**NEW 인시던트**:
1. OpenSearch `incidents` INDEX (status: "open", open_snapshot 고정)
2. OpenSearch 영속 성공 → `dm.incident.new` publish

**EXISTING 인시던트**:
1. OpenSearch 쓰기 없음 (EXISTING은 OpenSearch에 반영하지 않음)
2. 이벤트 발행 없음

**RESOLVED 인시던트**:
1. OpenSearch `incidents` UPDATE (status→"resolved", resolved_at, duration_ms, resolved_reason, group_path 덮어쓰기)
2. OpenSearch 영속 성공 → `dm.incident.resolved` publish

---

## 4. Group 변경 시 Resolve + Re-Open (신규)

### 배경

디바이스가 그룹을 이동하면, 해당 디바이스의 Open incident는 이전 그룹 컨텍스트에서 발생한 것이므로:
1. 기존 Open incident를 **auto-resolve** (reason: group_changed)
2. 동일 이슈를 새 그룹에서 **새 incident로 re-open** (새 issue_instance_id 발급)

이렇게 하는 이유: Group 필터로 조회 시, 이전 그룹의 이력과 새 그룹의 이력이 명확히 분리됨.

### 이벤트 소스

| 토픽 | 발신자 | 수신자 |
|------|--------|--------|
| `dm.device.group.changed` | device-core Pizza | threshold |

### 메시지 구조

```typescript
{
  device_id: string,
  workspace_id: string,
  previous_group_id: string,
  previous_group_path: string,
  new_group_id: string,
  new_group_path: string,
  changed_at: string  // ISO 8601
}
```

### threshold 처리

1. `dm.device.group.changed` 수신
2. OpenSearch에서 해당 device_id의 모든 Open incident 조회 (`device_id + status=open`)
3. 각 incident에 대해:

**Step A — 기존 incident Resolve**:
- OpenSearch `incidents` UPDATE:
  - `status` → "resolved"
  - `resolved_at` → changed_at
  - `resolved_reason` → "group_changed"
  - `group_path` → previous_group_path (resolve 시점 = 이동 전 그룹 유지)
  - `duration_ms` → changed_at - occurred_at
- `dm.incident.resolved` publish (resolved_reason: "group_changed")

**Step B — 새 그룹에서 Re-Open**:
- 새 `issue_instance_id` 생성 (UUID)
- OpenSearch `incidents` INDEX (새 문서):
  - `status`: "open"
  - `group_id`: new_group_id
  - `group_path`: new_group_path
  - `occurred_at`: changed_at (새 그룹에서의 시작 시점)
  - `open_snapshot`: 기존 diagnostics_data 유지 + 새 group 정보
- `dm.incident.new` publish

### 결과

```
[이전 그룹 이력]
incident-001: Open 04-20 → Resolved 04-25 (reason: group_changed) — Group A

[새 그룹 이력]
incident-002: Open 04-25 → (진행 중) — Group B
```

- Group A 필터: incident-001 보임 (Resolved)
- Group B 필터: incident-002 보임 (Open)
- 동일 이슈가 그룹 이동으로 인해 2개의 독립 incident로 분리됨

### resolved_reason 값

| 값 | 설명 |
|----|------|
| normal | 이슈 조건이 정상으로 복귀 |
| activation_off | 해당 이슈 코드의 모니터링 비활성화 |
| deactivated | 디바이스 비활성화 |
| deleted | 디바이스 삭제 |
| **group_changed** | **(신규)** 디바이스 그룹 이동 → resolve 후 새 그룹에서 re-open |

---

## 5. OpenSearch 문서 설계 (1-Row Update)

### 인덱스: `incidents`

```json
{
  "mappings": {
    "_routing": { "required": true },
    "properties": {
      "issue_instance_id":   { "type": "keyword" },
      "status":              { "type": "keyword" },
      "device_id":           { "type": "keyword" },
      "workspace_id":         { "type": "keyword" },
      "group_id":            { "type": "keyword" },
      "group_path":          { "type": "keyword" },
      "issue_code":          { "type": "keyword" },
      "issue_type":          { "type": "keyword" },
      "device_type":         { "type": "keyword" },
      "os_type":             { "type": "keyword" },
      "model_name":          { "type": "keyword" },
      "serial_number":       { "type": "keyword" },
      "severity":            { "type": "keyword" },
      "is_alertable":        { "type": "boolean" },
      "description":         { "type": "text", "index": false },
      "diagnostics_data":    { "type": "object", "enabled": false },
      "threshold_value":     { "type": "float" },
      "occurred_at":         { "type": "date" },
      "resolved_at":         { "type": "date" },
      "resolved_reason":     { "type": "keyword" },
      "duration_ms":         { "type": "long" },
      "event_time":          { "type": "date" },
      "open_snapshot": {
        "type": "object",
        "properties": {
          "group_id":        { "type": "keyword" },
          "group_path":      { "type": "keyword" },
          "severity":        { "type": "keyword" },
          "diagnostics_data": { "type": "object", "enabled": false },
          "threshold_value": { "type": "float" },
          "description":     { "type": "text", "index": false }
        }
      }
    }
  }
}
```

### 문서 lifecycle

**Open 시 (INDEX)**:
```json
{
  "issue_instance_id": "uuid-001",
  "status": "open",
  "device_id": "device-001",
  "workspace_id": "PROP-001",
  "group_id": "grp-a",
  "group_path": "root/region-a/floor-1/",
  "issue_code": "TEM-01",
  "severity": "Critical",
  "occurred_at": "2026-04-23T10:00:00Z",
  "event_time": "2026-04-23T10:00:00Z",
  "open_snapshot": {
    "group_id": "grp-a",
    "group_path": "root/region-a/floor-1/",
    "severity": "Critical",
    "diagnostics_data": { "value": 82, "threshold": 75 },
    "threshold_value": 75,
    "description": "온도 초과 (82°C)"
  },
  "resolved_at": null,
  "resolved_reason": null,
  "duration_ms": null
}
```

**Resolve 시 (UPDATE)**:
```json
{
  "doc": {
    "status": "resolved",
    "resolved_at": "2026-04-25T10:00:00Z",
    "resolved_reason": "normal",
    "duration_ms": 172800000,
    "group_id": "grp-a",
    "group_path": "root/region-a/floor-1/",
    "event_time": "2026-04-25T10:00:00Z"
  }
}
```

- `open_snapshot`은 Open 시 고정되며 **Resolve 시 건드리지 않음**
- `group_path`는 Resolve 시점의 값으로 덮어씀 (Group 필터가 각 시점 값에 자연 적용)
- 문서 ID = `issue_instance_id`
- routing = `workspace_id`

### 의사결정 사항: resolved_diagnostics_data 포함 여부

**배경**: DIF(Difference) 이슈 등에서 Open 시점과 Resolve 시점의 diagnostics 데이터를 비교하여 보여주는 씬이 존재할 수 있음.

**Option A — resolved_diagnostics_data 저장**:
- Resolve 시 `resolved_diagnostics_data` 필드를 OpenSearch에 추가 저장
- 장점: BE에서 open_snapshot.diagnostics_data vs resolved_diagnostics_data 비교 데이터를 직접 제공 가능
- 단점: 저장 용량 증가, UPDATE payload 증가

**Option B — FE에서 추론**:
- Open 시점 데이터(open_snapshot.diagnostics_data)만 저장
- Resolve 시점 데이터는 "정상 복귀"이므로 FE가 threshold 기준값 기반으로 추론
- 장점: 저장 최소화, OpenSearch 문서 크기 절감
- 단점: FE 로직 복잡도 증가, 추론이 불가능한 케이스 존재 가능

**현재 상태**: 미결정. 구현 시 확정 필요.

---

## 6. Domain Event 설계

### 토픽

| 토픽 | 용도 |
|------|------|
| `dm.incident.new` | 새 인시던트 발생 알림 |
| `dm.incident.resolved` | 인시던트 해소 알림 |

### 파티션 키

`issue_instance_id` — 동일 이슈의 OPEN/RESOLVE 순서 보장.

### 이벤트 스키마

```typescript
// dm.incident.new
{
  event_id: string,
  issue_instance_id: string,
  event_type: "IssueOpened",
  device_id: string,
  issue_code: string,
  workspace_id: string,
  occurred_at: string,
  event_time: string,
  snapshot: {
    group_id: string,
    group_path: string,
    device_type: string,
    os_type: string,
    model_name: string,
    serial_number: string,
    license_grade: string,
    severity: string,
    issue_type: string,
    is_alertable: boolean,
    diagnostics_data: Record<string, unknown>,
    threshold_value?: number,
    description: string
  }
}

// dm.incident.resolved
{
  event_id: string,
  issue_instance_id: string,
  event_type: "IssueResolved",
  device_id: string,
  issue_code: string,
  workspace_id: string,
  occurred_at: string,
  resolved_at: string,
  event_time: string,
  resolved_reason: string,  // "normal" | "activation_off" | "deactivated" | "deleted" | "group_changed"
  snapshot: {
    group_id: string,
    group_path: string,
    device_type: string,
    os_type: string,
    model_name: string,
    serial_number: string,
    diagnostics_data: Record<string, unknown>
  }
}
```

---

## 7. 저장소 설계

### AWS OpenSearch — incidents (유일한 저장소)

**용도**: State Diff + Issue History 리스트 + Current Issues (status=open) + Summary 통계 + Device Detail Open Issue + 엑셀 다운로드

- 1 document = 1 incident (전체 lifecycle)
- Open 시 INDEX, Resolve 시 UPDATE
- 문서 ID = `issue_instance_id`
- routing = `workspace_id`
- State Diff: `device_id + status=open` 필터로 해당 디바이스의 현재 Open incident 조회

---

## 8. 조회 설계

### Issue History 리스트 (단순 search)

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "workspace_id": "PROP-001" } },
        { "prefix": { "group_path": "root/region-a/" } },
        { "range": { "occurred_at": { "gte": "2026-04-01", "lte": "2026-04-30" } } },
        { "terms": { "device_type": ["signage", "led"] } },
        { "term": { "status": "open" } }
      ]
    }
  },
  "aggs": {
    "top_issues": { "terms": { "field": "issue_code", "size": 10 } },
    "status_dist": { "terms": { "field": "status" } },
    "trend": { "date_histogram": { "field": "occurred_at", "calendar_interval": "1d" } }
  },
  "sort": [{ "occurred_at": "desc" }, { "issue_instance_id": "desc" }],
  "size": 20
}
```

- **단일 쿼리**로 리스트 + Summary 통계 동시 반환
- Status 필터: `{ "term": { "status": "open" } }` 또는 `"resolved"` 또는 필터 없음(All)
- Pagination: search_after
- Total count: 정확 (문서 수 = incident 수)

### Summary 통계

동일 쿼리의 `aggs` 결과에서 추출:
- **Top Issue Types**: `top_issues` terms agg
- **Status Distribution**: `status_dist` terms agg
- **Issue Trend**: `trend` date_histogram

도넛차트(이슈 상태 분포)는 Status 필터 무관하게 항상 전체 기준 → Status 필터 제외한 별도 aggs 쿼리 1개 추가.

---

## 9. UX 설계

### 페이지 구조

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Issue Trend                                                              │
│                                                                           │
│  ┌─ 필터 ─────────────────────────────────────────────────────────────┐  │
│  │ Category: [All▼]  Issue Type: [All▼ 다중]  Status: [All▼ 다중]     │  │
│  │ Target (Group): [▼ 다중]     Period: [최근 30일▼]                   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─ Summary ──────────────────────────────────────────────────────────┐  │
│  │ [막대: Top Issue Types] [선: 발생 추이] [도넛: Open/Resolved 분포]  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─ Issue History 테이블 ─────────────────────────────────────────────┐  │
│  │ Status│Occurred│Resolved│Group│Issue Type│Device│OS│Model│S/N│Detail│  │
│  │ Open  │04-26   │ —      │HQ/1F│Temp.    │LED   │webOS│55UH│ABC│ → │  │
│  │ Reslvd│04-23   │04-25   │HQ/2F│Temp.    │LED   │webOS│55UH│ABC│ → │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│  [◀ 1 2 3 ▶]                                    [Export: Excel│CSV│PDF]  │
└──────────────────────────────────────────────────────────────────────────┘
```

### Resolved Reason 표시

| reason | 표시 |
|--------|------|
| normal | 정상 복귀 |
| activation_off | 모니터링 비활성화 |
| deactivated | 디바이스 비활성화 |
| deleted | 디바이스 삭제 |
| **group_changed** | **그룹 이동** |

---

## 10. 안정성 설계

### Idempotency

- OpenSearch incidents: 문서 ID = `issue_instance_id`. 동일 ID로 INDEX 재실행 시 덮어쓰기 (멱등).
- OpenSearch UPDATE: `_update` API 사용. 이미 resolved 상태면 동일 내용 덮어쓰기 (멱등).

### Ordering

- threshold 내부에서 순차 처리: OpenSearch write → 이벤트 publish
- 동일 디바이스의 메시지는 Kafka 파티션 키(device_id)로 순서 보장
- 동일 incident에 대해 Open과 Resolve가 동시에 오는 것은 구조적으로 불가능

### Write 실패 처리

| 실패 지점 | 처리 |
|-----------|------|
| OpenSearch incidents 실패 | 해당 인시던트 skip + error 로그. 이벤트 publish 안 함 |
| GlobalEventBus publish 실패 | error 로그. OpenSearch에는 이미 기록됨 (알림만 누락) |

---

## 11. OpenSearch 검색 최적화

### Routing

- 인덱싱/검색 시 `routing = workspace_id`
- 같은 workspace의 문서가 같은 shard에 저장 → 검색 시 단일 shard만 조회

### ISM (Index State Management)

```json
{
  "hot":    { "max_age": "30d" },
  "warm":   { "min_age": "30d", "shrink": 1, "forcemerge": 1, "readonly": true },
  "cold":   { "min_age": "90d", "searchable_snapshot": true },
  "delete": { "min_age": "365d" }
}
```

### 필드 타입 설계

| 필드 | 타입 | 이유 |
|------|------|------|
| status | keyword | Status 필터 + terms agg |
| issue_instance_id | keyword | 문서 ID, 정확 매칭 |
| workspace_id, device_id, group_id | keyword | filter context |
| group_path | keyword | prefix query |
| issue_code, device_type, os_type, severity | keyword | filter + agg |
| model_name, serial_number | keyword | prefix 검색 |
| occurred_at, resolved_at, event_time | date | range filter + date_histogram |
| duration_ms | long | 표시용 |
| description | text (index: false) | 저장만, 검색 불가 |
| diagnostics_data | object (enabled: false) | 저장만, 인덱스 크기 절감 |
| open_snapshot.* | 각 타입 | Open 시점 스냅샷 보존 |

---

## 12. 구현 우선순위

| 순서 | 작업 |
|------|------|
| 1 | threshold: issue_instance_id 생성 + OpenSearch INDEX (Open) 구현 |
| 2 | threshold: OpenSearch UPDATE (Resolve) 구현 |
| 3 | threshold: State Diff를 OpenSearch 조회로 구현 (device_id + status=open) |
| 4 | threshold: write-then-publish 패턴 적용 |
| 5 | threshold: dm.device.group.changed 수신 → resolve + re-open 구현 |
| 6 | monitoring: Issue History 조회를 단순 search로 구현 |
| 7 | monitoring: Summary 통계를 동일 쿼리 aggs로 구현 |
| 8 | monitoring: Status 필터 (All/Open/Resolved) 구현 |
| 9 | OpenSearch ISM 정책 설정 |

---

## 13. OpenSearch 인프라 설정 (최신)

> 최종 업데이트: 2026-05-28

### 인덱스 구조: Rollover 기반

단일 인덱스가 아닌 **rollover 기반 multi-index** 구조를 사용한다.

```
incidents-000001 (2026-05 생성, rollover됨)
incidents-000002 (2026-06 생성, rollover됨)
incidents-000003 (2026-07 생성, 현재 active)

[incidents-write] alias → 현재 활성 인덱스 1개 (쓰기 전용)
[incidents]       alias → 전체 인덱스 (읽기 전용)
```

| 항목 | 값 |
|------|-----|
| Index template | `incidents-template` |
| ISM 정책 | `incidents-lifecycle` |
| 초기 인덱스 | `incidents-000001` |
| Write alias | `incidents-write` (INDEX/UPDATE 시 사용) |
| Read alias | `incidents` (SEARCH 시 사용) |
| Routing | `workspace_id` (필수) |
| 문서 ID | `issue_instance_id` |

### Rollover 조건

| 조건 | 값 | 환경변수 |
|------|-----|----------|
| 인덱스 나이 | 30일 | `OS_INCIDENTS_ROLLOVER_AGE` |
| 인덱스 크기 | 30GB | `OS_INCIDENTS_ROLLOVER_SIZE` |

둘 중 하나라도 충족하면 OpenSearch ISM이 자동으로 rollover 실행.

### ISM Lifecycle

```
hot (0~30d):
  - 활발한 write/update
  - rollover 조건 충족 시 새 인덱스 자동 생성 + write alias 전환

warm (30d~395d):
  - force_merge (1 segment) — 검색 성능 최적화
  - replica: 1 유지
  - 극소수 late resolve update 허용 (read_only 아님)

delete (395d 경과):
  - 인덱스 삭제
  - 395d = 30d(rollover 주기) + 365d(보존 기간) → 최근 1년 데이터 보존 보장
```

### Shard 설정

| 항목 | 값 | 환경변수 |
|------|-----|----------|
| Primary shards | 2 | `OS_INCIDENTS_SHARDS` |
| Replica shards | 1 | `OS_INCIDENTS_REPLICAS` |

노드 2대 기준 최적. prod 확장 시 환경변수로 조정.

### 코드에서의 사용

```typescript
// 새 문서 생성 (INDEX)
client.index({ index: INCIDENTS_WRITE_ALIAS, id, routing, body })

// 기존 문서 업데이트 (RESOLVE)
client.update({ index: INCIDENTS_READ_ALIAS, id, routing, body })

// 검색 (SEARCH)
client.search({ index: INCIDENTS_READ_ALIAS, body: { query } })
```

- Write alias로 INDEX하면 현재 활성 인덱스에 자동 라우팅
- Read alias로 UPDATE하면 OpenSearch가 routing + id로 해당 문서가 있는 인덱스를 찾아서 update
- Read alias로 SEARCH하면 전체 인덱스 대상 검색

### 설계 결정 사항

| 결정 | 이유 |
|------|------|
| cold 단계 제거 | UltraWarm 미사용 + long-lived open issue의 late resolve 허용 필요 |
| delete 395d (365d 아님) | rollover 주기(30d) 여유분 포함 → 최근 1년 보존 확실 보장 |
| read_only 미적용 | open issue가 수개월 유지될 수 있어 resolve update 차단 불가 |
| rollover 30d | monthly 분리. retention 관리 + recovery 부담 감소 |
| shard 2 | 노드 2대 = shard 2 균등 분배. 데이터 성장 시 환경변수로 조정 |
