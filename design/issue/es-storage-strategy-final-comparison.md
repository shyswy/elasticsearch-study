# ES 저장 전략 최종 비교: 1-Row Update vs 2-Row Append-Only

## 배경: 시스템 구조와 Incident Lifecycle

### 이 시스템이 하는 일

IoT 디바이스(~10만 대)가 2분 주기로 상태 리포트를 보냄. 리포트에 이상 징후(이슈)가 포함되면 **incident**로 관리됨. incident는 "발생(Open) → 해소(Resolved)"의 lifecycle을 가짐.

### Incident Lifecycle (핵심 — 반드시 이해 필요)

```
Open ──────────→ Resolved ──────────→ 끝 (영구 불변)
 │                  │
 │ 1회 발생         │ 1회 해소
 │                  │
 └── 이후 상태 변경 없음 ──────────────┘
```

**불변 규칙:**
- 각 incident는 **Open 1회 → Resolved 1회 → 끝**. 이것이 전부.
- **Reopen 없음.** 한번 Resolved되면 상태가 다시 바뀌지 않음.
- 같은 이슈가 다시 발생하면 **새로운 incident**(새 ID)로 생성됨.
- 따라서 하나의 ES 문서는 생애 동안 **최대 1번만 수정될 수 있음** (Update 방식 채택 시).
- Open 상태에서 Resolved 없이 영원히 남는 incident도 존재 가능 (디바이스가 계속 이상 상태).

### 데이터 흐름

```
디바이스 리포트 (Kafka) → threshold 모듈 (EKS)
                              │
                              ├─ State Diff: 기존 Open incident와 비교
                              │   → NEW: 새 incident 생성
                              │   → EXISTING: diagnostics만 갱신 (ES 쓰기 없음)
                              │   → RESOLVED: incident 해소
                              │
                              ├─ PG current_issues: 현재 Open인 것만 (INSERT/DELETE)
                              ├─ ES incident_events: 이력 저장 (이 문서의 비교 대상)
                              └─ Kafka 이벤트 발행 → alert 모듈 (알림 발송)
```

### ES에 저장하는 목적

- Issue History 리스트 조회 (복합 필터 7~8개 + 기간 범위 + prefix 검색)
- Summary 통계 (Top Issue Types, Status Distribution, Trend Chart)
- 엑셀 다운로드

### 왜 ES인가 (PG가 아닌 이유)

- 복합 필터 조합 폭발 문제: PG에서는 인덱스 조합이 기하급수적으로 늘어남
- ES는 역색인 + bitset 캐싱으로 어떤 필터 조합이든 인덱스 설계 고민 없이 처리
- 연 수억 건 누적 + 1년 조회 범위 → ES의 ILM(Hot/Warm/Cold/Delete) 자동 lifecycle 관리

### 동시 쓰기가 발생하지 않는 이유

- threshold가 Kafka consumer로 동작하며, **device_id가 파티션 키**
- 같은 device의 메시지는 같은 파티션 → 같은 consumer → **순차 처리 보장**
- 하나의 incident에 대해 Open과 Resolve가 동시에 처리되는 것은 구조적으로 불가능
- 따라서 ES Update 방식을 채택하더라도 **version conflict는 발생하지 않음**

---

## Overview

Issue History(인시던트 이력)를 Elasticsearch에 저장할 때 두 가지 방식의 비용을 3가지 관점에서 비교한다.

| | 1-Row Update | 2-Row Append-Only |
|---|---|---|
| Open 시 | INDEX 새 문서 (status: open) | INDEX 새 문서 (event_type: IssueOpened) |
| Resolve 시 | 기존 문서 UPDATE | INDEX 새 문서 (event_type: IssueResolved) |
| incident당 문서 수 | 1개 | 2개 |
| 조회 | 단순 search | collapse + inner_hits |

### 시스템 특성 (전제)

- 각 incident는 Open 1회 → Resolve 1회 → 끝. **이후 해당 문서는 영원히 수정되지 않음.**
- 동시 쓰기 불가능 (threshold가 device_id 파티션 기반 순차 처리)
- 규모: 초당 최대 ~16,600 인시던트 변경, 월 ~1,000만 incident, 연 수억 건
- EXISTING(diagnostics 갱신)은 ES에 쓰지 않음 (두 방식 모두)

### UX 기준
기존 3.0 issue trend UX 그대로 유지 ( 이전 UX 건의 사항은 우선 배제 )

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Issue Trend                                                              │
│                                                                           │
│  ┌─ 공통 필터 ────────────────────────────────────────────────────────┐  │
│  │  Target (Group): [▼ 다중 선택]       Period: [2026-04-01] ~ [04-30] │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─ Summary ──────────────────────────────────────────────────────────┐  │
│  │  ┌──────────────┐  ┌───────────────────┐  ┌────────────────────┐  │  │
│  │  │ Top Issue    │  │ Status            │  │ Issue Trend        │  │  │
│  │  │ Types        │  │ Distribution      │  │ (Timeseries)       │  │  │
│  │  │              │  │                   │  │                    │  │  │
│  │  │ 1. TEM-01 45 │  │  🔴 Open    120  │  │  ╭─╮              │  │  │
│  │  │ 2. DSK-01 32 │  │  ✅ Resolved 380 │  │ ╭╯  ╰╮  ╭╮       │  │  │
│  │  │ 3. NET-01 18 │  │                   │  │╯     ╰──╯ ╰──     │  │  │
│  │  └──────────────┘  └───────────────────┘  └────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─ Scene 1: Issues History ──────────────────────────────────────────┐  │
│  │  ┌────────────────────────────────────────────────────────────┐    │  │
│  │  │ Issue │ Device │ Model │ S/N │ Group │ Severity │ Occurred │    │  │
│  │  │ TEM-01│ d1     │ 55UH  │ ABC │ HQ/1F │ Critical │ 04-26   │    │  │
│  │  │ DSK-01│ d2     │ 49UM  │ DEF │ HQ/2F │ Warning  │ 04-24   │    │  │
│  │  └────────────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

공통 필터(Group, Period)가 Summary 통계와 Issue History 리스트 모두에 적용됨.

---

## 1. 쓰기 비용 비교: dead docs vs 인덱스 2배

### 1-Row Update의 비용

ES(Lucene)에서 update의 실제 동작:

```
1. 기존 문서의 _source를 shard 내부에서 읽어옴
2. 변경 적용
3. 새 문서를 새 segment에 INDEX
4. 기존 문서를 soft-delete 마킹 (segment 내 .del 비트맵)
5. 이후 merge 시 물리 삭제
```

이 시스템에서의 결과:
- 각 incident의 원본 Open 문서가 dead doc으로 남음
- merge 전까지 segment 내 dead docs 비율 상승
- merge 시 dead docs 정리 → I/O + CPU 소모

**하지만**: 각 문서가 생애 1회만 update되고 끝이므로:
- dead docs는 각 문서당 최대 1개
- "같은 문서를 반복 update"하는 패턴이 아님
- merge가 충분히 따라잡을 수 있는 수준
- steady state에서 dead docs 비율은 관리 가능

### 2-Row Append-Only의 비용

- 문서 수 2배 → 역색인 posting 2배, doc_values entry 2배
- 모든 indexed 필드에 대해 인덱스 크기 2배
- Heap 메모리 (term dictionary FST): ~1.5배 (term 종류는 동일, posting list만 길어짐)

### 수치 비교 (월 1,000만 incident 기준)

| | 1-Row Update | 2-Row Append-Only |
|---|---|---|
| 쓰기 횟수 | 2,000만 (INDEX 1,000만 + UPDATE 1,000만) | 2,000만 (INDEX 1,000만 + INDEX 1,000만) |
| 유효 문서 수 | 1,000만 | 2,000만 |
| Dead docs (merge 전) | ~수백만 (merge lag 의존) | 0 |
| 역색인 크기 | 1x | 2x |
| 월별 인덱스 크기 | ~12GB | ~20GB |
| Merge 부하 | dead docs 정리 비용 있음 | 없음 (합치기만) |

### 판정

| 관점 | 유리한 쪽 | 설명 |
|------|-----------|------|
| 저장 효율 | Update | 인덱스 크기 절반 |
| Segment 건강도 | Append-Only | dead docs 0%, merge 단순 |
| Merge I/O | Append-Only | dead docs 정리 불필요 |
| 쓰기 latency | Append-Only | INDEX만 vs _source GET + INDEX |

**하지만 이 시스템에서는**: 1회 update 패턴이므로 dead docs 비율이 극단적으로 높아지지 않음. Update의 쓰기 비용 문제는 **크리티컬하지 않음**. 인덱스 2배 비용도 ILM(warm/cold tier)으로 관리 가능. **둘 다 허용 범위 내.**

---

## 2. 조회 비용 비교: collapse vs 단순 search

### 2-Row Append-Only의 조회 (collapse)

같은 incident의 Open + Resolved 문서를 그룹핑하여 대표 1개만 반환:

```json
{
  "query": { "bool": { "filter": [...] } },
  "collapse": {
    "field": "issue_instance_id",
    "inner_hits": {
      "name": "open_event",
      "size": 1,
      "sort": [{ "event_time": "asc" }]
    }
  },
  "sort": [{ "event_time": "desc" }],
  "size": 20
}
```

collapse 내부 동작:
1. filter 조건에 매칭되는 **전체 문서** 스캔 (2N개)
2. issue_instance_id별로 그룹핑
3. 각 그룹에서 sort 기준 대표 1개 선택
4. inner_hits: 각 그룹 내에서 추가 문서 반환 (Open 스냅샷용)
5. 최종 결과: N개 incident (대표 row + inner_hits)

### 1-Row Update의 조회 (단순 search)

```json
{
  "query": { "bool": { "filter": [...] } },
  "sort": [{ "event_time": "desc" }],
  "size": 20
}
```

단순 동작:
1. filter 조건에 매칭되는 문서 스캔 (N개)
2. sort 기준 상위 20개 반환
3. 끝

### 비용 차이 분석

| 단계 | 1-Row Update | 2-Row Append-Only | 차이 |
|------|---|---|---|
| Filter 스캔 대상 | N개 문서 | 2N개 문서 | 2배 스캔 |
| 그룹핑 연산 | 없음 | issue_instance_id별 그룹핑 | 추가 CPU |
| inner_hits | 없음 | 그룹당 1개 추가 fetch | 추가 I/O |
| 결과 크기 | 20 docs | 20 docs + 20 inner_hits | 응답 약간 큼 |
| Bitset 캐싱 효과 | 적용됨 | 동일하게 적용됨 | 차이 없음 |

### 실측 예상 latency (workspace당 30일 조회 기준)

- Update: ~27만 문서 스캔 → ~30~50ms
- Append-Only: ~53만 문서 스캔 + collapse → ~50~80ms
- **차이: 약 20~30ms**

### 판정

collapse 오버헤드는 존재하지만:
- 절대값으로 수십 ms 수준
- 사용자 체감 차이 미미 (100ms 이내)
- bitset 캐싱으로 반복 조회 시 더 줄어듦

**조회 성능 차이는 크리티컬하지 않음.** 차이는 "쿼리 복잡도(구현 난이도)"에 있지 "성능"에 있지 않음.

---

## 3. 필터 + 통계 + 리스트 연동 시 쿼리 구조

### Group 필터 시점 처리

두 방식 모두 자연스럽게 처리됨:
- **Update**: resolve 시 group_path를 resolve 시점 값으로 덮어쓰면, 각 문서의 group_path가 곧 해당 상태 시점의 group. 필터 시 자연 적용.
- **Append-Only**: 각 이벤트 문서에 해당 시점의 group이 기록되어 있으므로 필터 시 자연 적용.

**비교 대상 아님 — 둘 다 동일하게 동작.**

### Summary 통계 쿼리

| | 1-Row Update | 2-Row Append-Only |
|---|---|---|
| Top Issue Types | `terms` agg on `issue_code` (같은 쿼리) | `terms` agg on `issue_code` + `event_type=IssueOpened` 필터 |
| Status Distribution | `terms` agg on `status` (같은 쿼리) | `terms` agg on `event_type` (같은 쿼리) |
| Issue Trend (timeseries) | `date_histogram` on `occurred_at` (같은 쿼리) | `date_histogram` on `event_time` + `event_type=IssueOpened` 필터 |
| 리스트와 통계 기준 일치 | ✅ 동일 쿼리 기반 | ⚠️ 별도 쿼리 (collapse 결과와 aggs 기준 불일치) |

### 쿼리 수 비교

| 화면 요소 | 1-Row Update | 2-Row Append-Only |
|-----------|---|---|
| Issue History 리스트 | 1 쿼리 | 1 쿼리 (collapse) |
| Summary 통계 | 같은 쿼리에 aggs 추가 가능 | 별도 쿼리 (collapse와 aggs 기준 불일치이므로 분리) |
| **총 쿼리 수** | **1개** (search + aggs 통합) | **2개** (collapse 쿼리 + aggs 쿼리) |

### 판정

| 관점 | 1-Row Update | 2-Row Append-Only |
|------|---|---|
| Group 필터 시점 처리 | 자연 적용 | 자연 적용 |
| 리스트 + 통계 일관성 | ✅ 동일 쿼리 기반 | ⚠️ 별도 쿼리 필요 |
| 쿼리 수 | 1개 | 2개 |

---

## 종합 비교

| 관점 | 1-Row Update | 2-Row Append-Only | 심각도 |
|------|:---:|:---:|------|
| **1. 쓰기 비용** | | | |
| └ 저장 효율 (인덱스 크기) | ◎ (1x) | ○ (2x) | 낮음 — ILM으로 관리 가능 |
| └ Segment 건강도 | ○ (dead docs 있음) | ◎ (0%) | 낮음 — 1회 update라 관리 가능 |
| └ Merge 부하 | ○ | ◎ | 낮음 — 크리티컬하지 않음 |
| **2. 조회 비용** | | | |
| └ 검색 latency | ◎ (~40ms) | ○ (~70ms) | 낮음 — 체감 차이 미미 |
| └ 쿼리 구현 복잡도 | ◎ (단순 search) | ○ (collapse + inner_hits) | 중간 — 한번 구현하면 패턴화 |
| **3. 필터 + 통계 연동** | | | |
| └ 리스트-통계 일관성 | ◎ (단일 쿼리) | ○ (별도 쿼리 2개) | 중간 |
| └ Group 필터 시점 처리 | 동일 | 동일 | 비교 대상 아님 — 둘 다 자연스럽게 처리됨 |
| └ 쿼리 수 | ◎ (1개) | ○ (2개) | 낮음 — 네트워크 비용 미미 |
| **기타** | | | |
| └ Open 스냅샷 보존 | ○ (별도 필드 설계) | ◎ (자연 보존) | 중간 |
| └ 멱등성 | ○ (upsert 로직) | ◎ (자연 멱등) | 낮음 |
| └ ES 엔진 궁합 | △ (mutable 패턴) | ◎ (immutable 패턴) | 낮음 — 1회 update라 실질 영향 적음 |

---

## 최종 결론

### 이 시스템 특성에서의 현실적 평가

이 시스템은 "각 문서가 생애 1회만 update되고 끝"이라는 특수한 패턴. 이로 인해:

1. **쓰기 비용**: 둘 다 크리티컬하지 않음. Update의 dead docs도, Append-Only의 인덱스 2배도 모두 허용 범위.
2. **조회 비용**: 성능 차이 수십 ms. 체감 불가. 차이는 구현 복잡도에 있음.
3. **필터+통계 연동**: Update가 단일 쿼리로 리스트+통계 일관성 확보 가능. Append-Only는 2개 쿼리 분리 필요.

### 각 방식의 실질적 이점

**1-Row Update를 선택하면 얻는 것:**
- 조회 단순성 (collapse 불필요)
- 리스트 + 통계 단일 쿼리 (일관된 기준)
- 저장 효율 (인덱스 절반)
- 직관적 디버깅 (1 doc = 1 incident 전체 상태)

**2-Row Append-Only를 선택하면 얻는 것:**
- Open 스냅샷 자연 보존 (추가 설계 0)
- 자연 멱등성 (event_id = doc_id)
- Segment 100% 유효 (merge 부하 최소)
- ES immutable 철학 부합

### 판단

두 방식 모두 이 시스템에서 안전하게 동작함. "1회 update" 패턴이므로 ES update의 전통적 문제점(segment fragmentation, version conflict)이 크리티컬하지 않음.

선택은 **트레이드오프 영역**:
- "조회 단순성 + 통계 일관성"을 우선 → **1-Row Update**
- "ES 본래 패턴 + 스냅샷 자연 보존 + 멱등성"을 우선 → **2-Row Append-Only**
