# 4단계: Aggregation (집계)

## 학습 일자
2026-06-09

## 핵심 개념 정리

### Aggregation이란?

**정의**: 검색 결과를 그룹핑하고 통계를 계산하는 기능. SQL의 `GROUP BY` + 집계 함수(COUNT, AVG, SUM 등)에 해당.

**왜 존재하는가**:
- "status별 이슈 수", "일별 발생 추이", "issue_code별 평균 해결 시간" 같은 통계/분석 요구사항
- RDB에서는 GROUP BY + JOIN + 서브쿼리 조합이 복잡해지지만, ES는 중첩 집계(Sub-Aggregation)로 다차원 분석을 단일 쿼리에서 처리
- ES는 Doc Values(열 지향 저장소)에서 직접 읽어서 집계 → 대량 데이터에서도 빠름

**내부 동작**:
- 집계는 Doc Values를 읽어서 수행 (keyword, date, numeric, boolean 필드)
- text 타입은 Doc Values가 없으므로 직접 집계 불가 → keyword sub-field 필요
- 각 샤드에서 로컬 집계 수행 → Coordinating Node에서 합산

---

### Aggregation의 두 가지 축

**Bucket Aggregation — "어떻게 그룹을 나눌 것인가" (= SQL의 GROUP BY)**

문서를 기준에 따라 버킷(그룹)으로 나눔.

| 타입 | 설명 | SQL 비유 |
|------|------|---------|
| `terms` | 필드 값별 그룹핑 | GROUP BY field |
| `date_histogram` | 시간 구간별 그룹핑 | GROUP BY DATE_TRUNC(interval) |
| `range` | 숫자/날짜 범위별 그룹핑 | GROUP BY CASE WHEN ... |
| `filter` | 단일 조건으로 버킷 1개 | WHERE 조건 서브쿼리 |
| `filters` | 여러 조건으로 각각 버킷 | 여러 CASE WHEN |
| `histogram` | 숫자를 고정 간격으로 그룹핑 | GROUP BY FLOOR(field/interval) |

**Metric Aggregation — "각 그룹에서 뭘 계산할 것인가" (= SQL의 집계 함수)**

각 버킷(또는 전체) 내에서 통계 값을 계산.

| 타입 | 설명 | SQL 비유 |
|------|------|---------|
| `avg` | 평균 | AVG(field) |
| `sum` | 합계 | SUM(field) |
| `min` / `max` | 최솟값 / 최댓값 | MIN / MAX |
| `value_count` | 값이 있는 문서 수 | COUNT(field) |
| `cardinality` | 고유값 수 (근사치) | COUNT(DISTINCT field) |
| `stats` | count + min + max + avg + sum 한 번에 | — |
| `percentiles` | 백분위수 (p50, p90, p99) | PERCENTILE_CONT |

---

### 핵심 구조: Bucket(바깥) → Metric(안쪽) = "그룹별 통계"

**이것이 Aggregation의 가장 중요한 구조.**

```
"그룹별 통계" = Bucket Agg { Metric Agg }

SQL: SELECT issue_code, AVG(duration_ms) FROM issues GROUP BY issue_code
ES:  terms(issue_code) { avg(duration_ms) }
```

Metric만 단독으로 쓰면 "전체 통계 1개"만 나옴. "~별"이 필요하면 반드시 Bucket이 바깥을 감싸야 함:

```
avg(duration_ms) 단독:
  → 전체 평균 1개 (숫자 1개)

terms(issue_code) { avg(duration_ms) }:
  → TEM-01의 평균, DSK-01의 평균, NET-01의 평균... (그룹별)

date_histogram(일별) { avg(duration_ms) }:
  → 6/7 평균, 6/8 평균, 6/9 평균... (시간대별)
```

---

### Aggregation 기본 구조 (쿼리 형태)

```json
GET /issue-history/_search
{
  "size": 0,          // 문서 본문 필요 없음 (집계 결과만)
  "query": { ... },   // 집계 대상 범위를 query로 제한
  "aggs": {
    "my_agg_name": {  // 집계 이름 (자유 지정)
      "terms": {      // 집계 타입
        "field": "status"
      }
    }
  }
}
```

**`"size": 0`을 쓰는 이유**:
- 집계 결과만 필요할 때 문서 hits를 아예 안 가져옴
- 네트워크 전송량 감소 + 응답 속도 향상
- 대시보드 차트 데이터를 뽑을 때는 거의 항상 `size: 0`
- 안 쓰면 기본 10개의 문서 hits가 같이 반환됨 (불필요한 데이터)

**query + aggs 조합**:
- query로 범위를 좁히고, 그 범위 내에서 집계
- query 없이 aggs만 쓰면 전체 인덱스 대상으로 집계

---

### terms aggregation

필드의 고유값별로 문서 수를 세고 그룹핑.

```json
"aggs": {
  "by_status": {
    "terms": { "field": "status" }
  }
}
```

**응답 구조 (디폴트)**:
```json
"by_status": {
  "doc_count_error_upper_bound": 0,
  "sum_other_doc_count": 0,
  "buckets": [
    { "key": "resolved", "doc_count": 158 },
    { "key": "open", "doc_count": 42 }
  ]
}
```

- `key`: 필드의 고유값 (자동)
- `doc_count`: 그 값을 가진 문서 수 (자동)
- `sum_other_doc_count`: size 한계로 잘려서 반환 안 된 나머지 버킷들의 문서 합계
  - **0이면 "전부 반환됐다"는 보장**
  - 0이 아니면 size를 늘려야 함

**핵심 옵션**:
- `size`: 반환할 최대 버킷 수. **기본 10, 최대 10,000**. 고유값이 10개 초과면 잘림.
- `order`: 정렬 기준. 기본 `doc_count` desc. sub-agg 기준 정렬도 가능.

```json
"terms": {
  "field": "issue_code",
  "size": 50,
  "order": { "avg_duration": "desc" }  // sub-agg 기준 정렬
}
```

**가짓수를 모르는 경우의 전략**:
- 가짓수 예측 가능 (수십 개): `size`를 넉넉히 (예: 100)
- 가짓수 예측 불가 (수천~수만): **composite aggregation** 사용 (4.5단계)
- terms의 size 최대값은 10,000. 이를 초과하면 terms로 전부 가져오기 불가능.

---

### date_histogram aggregation

시간 구간별로 문서를 그룹핑. 추이 차트(선그래프)의 핵심.

```json
"aggs": {
  "daily_trend": {
    "date_histogram": {
      "field": "occurred_at",
      "calendar_interval": "day"
    }
  }
}
```

**interval 옵션**:
- `calendar_interval`: "day", "week", "month", "quarter", "year" (달력 기준, 월마다 일수 다름)
- `fixed_interval`: "1h", "30m", "7d" (고정 간격)

**빈 구간 처리 (차트에서 0건인 날도 표시)**:
```json
"date_histogram": {
  "field": "occurred_at",
  "calendar_interval": "day",
  "min_doc_count": 0,
  "extended_bounds": {
    "min": "2025-05-10",
    "max": "2025-06-09"
  }
}
```

- `min_doc_count: 0`: 중간에 0건인 날도 버킷 생성
- `extended_bounds`: 데이터가 없는 양 끝 날짜까지 강제로 범위 확장
- **둘 다 함께 써야 완전한 빈 구간 처리**. min_doc_count만으로는 양 끝에 데이터 없는 날이 빠질 수 있음.

---

### filter / filters aggregation

**filter (단일 조건 → 버킷 1개)**:
```json
"aggs": {
  "open_only": {
    "filter": { "term": { "status": "open" } },
    "aggs": {
      "count": { "value_count": { "field": "issue_instance_id" } }
    }
  }
}
```

**filters (여러 조건 → 각각 버킷)**:
```json
"aggs": {
  "status_distribution": {
    "filters": {
      "filters": {
        "open": { "term": { "status": "open" } },
        "resolved": { "term": { "status": "resolved" } }
      }
    }
  }
}
```

**filters vs terms**:
- terms: 필드의 모든 고유값을 자동으로 그룹핑
- filters: 직접 정의한 조건별로 그룹핑 (복합 조건 가능, 이름 지정 가능)

---

### range aggregation

숫자/날짜를 범위별로 그룹핑:
```json
"aggs": {
  "resolution_ranges": {
    "range": {
      "field": "duration_ms",
      "ranges": [
        { "key": "under_1h", "to": 3600000 },
        { "key": "1h_to_6h", "from": 3600000, "to": 21600000 },
        { "key": "over_6h", "from": 21600000 }
      ]
    }
  }
}
```

---

### cardinality — 고유값 수 (근사치)

```json
"aggs": {
  "unique_devices": {
    "cardinality": { "field": "device_id" }
  }
}
```

- HyperLogLog++ 알고리즘 기반의 **근사치** (정확한 DISTINCT COUNT가 아님)
- 고유값 수천 이하면 거의 정확, 수백만 이상이면 오차 있음
- `precision_threshold` (기본 3000)로 정확도 조절 가능

---

### 중첩 Aggregation (Sub-Aggregation)

Bucket 안에 또 다른 Bucket이나 Metric을 넣어서 다차원 분석:

```json
// "issue_code별 → 건수 + 평균 해결 시간 + 일별 추이"
{
  "size": 0,
  "aggs": {
    "by_issue_code": {
      "terms": { "field": "issue_code", "size": 20 },
      "aggs": {
        "avg_duration": {
          "avg": { "field": "duration_ms" }
        },
        "max_duration": {
          "max": { "field": "duration_ms" }
        },
        "daily": {
          "date_histogram": {
            "field": "occurred_at",
            "calendar_interval": "day"
          }
        }
      }
    }
  }
}
```

결과 구조:
```
by_issue_code:
  ├── TEM-01: 45건
  │     ├── avg_duration: 3600000ms
  │     ├── max_duration: 86400000ms
  │     └── daily: 6/7(3건), 6/8(5건), 6/9(2건)...
  ├── DSK-01: 30건
  │     ├── avg_duration: 7200000ms
  │     ├── max_duration: 43200000ms
  │     └── daily: 6/7(1건), 6/8(4건)...
  └── ...
```

**중첩 규칙**:
- Bucket 안에 Metric: "각 그룹의 통계" (가장 흔함)
- Bucket 안에 Bucket: "다차원 그룹핑" (issue_code별 → 일별)
- Metric 안에 다른 agg: 불가 (Metric은 말단 노드)

---

## 현재 프로젝트 연결

### Issue Trend Summary 차트 — 단일 쿼리로 3개 차트 데이터 추출

```json
GET /issue-history/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "workspace_id": "ws-123" } },
        { "range": { "occurred_at": { "gte": "now-30d" } } }
      ]
    }
  },
  "aggs": {
    "top_issue_types": {
      "terms": { "field": "issue_code", "size": 10 }
    },
    "daily_trend": {
      "date_histogram": {
        "field": "occurred_at",
        "calendar_interval": "day",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "now-30d",
          "max": "now"
        }
      }
    },
    "status_distribution": {
      "filters": {
        "filters": {
          "open": { "term": { "status": "open" } },
          "resolved": { "term": { "status": "resolved" } }
        }
      }
    }
  }
}
```

매핑:
- 막대차트 (Top Issue Types) → `top_issue_types`
- 선그래프 (발생 추이) → `daily_trend`
- 도넛차트 (Open/Resolved 분포) → `status_distribution`

### Issue Trend에서 추가로 필요한 집계 패턴

- "issue_code별 평균 해결 시간" → terms + avg sub-agg
- "그룹(group_path)별 이슈 수" → terms on group_path
- "일별 추이를 status별로 분리" → date_histogram + filters sub-agg

---

## 실전 쿼리/설정 예시

### issue_code별 건수 + 평균/최대 해결 시간

```json
GET /issue-history/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "workspace_id": "ws-123" } },
        { "term": { "status": "resolved" } }
      ]
    }
  },
  "aggs": {
    "by_issue_code": {
      "terms": { "field": "issue_code", "size": 50 },
      "aggs": {
        "avg_duration": { "avg": { "field": "duration_ms" } },
        "max_duration": { "max": { "field": "duration_ms" } }
      }
    }
  }
}
```

### 시간대별(일별) 평균 해결 시간

```json
GET /issue-history/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "workspace_id": "ws-123" } },
        { "term": { "status": "resolved" } },
        { "range": { "occurred_at": { "gte": "now-30d" } } }
      ]
    }
  },
  "aggs": {
    "daily_trend": {
      "date_histogram": {
        "field": "occurred_at",
        "calendar_interval": "day"
      },
      "aggs": {
        "avg_duration": { "avg": { "field": "duration_ms" } }
      }
    }
  }
}
```

### 해결 시간 분포 (범위별)

```json
GET /issue-history/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "workspace_id": "ws-123" } },
        { "term": { "status": "resolved" } }
      ]
    }
  },
  "aggs": {
    "resolution_ranges": {
      "range": {
        "field": "duration_ms",
        "ranges": [
          { "key": "under_1h", "to": 3600000 },
          { "key": "1h_to_6h", "from": 3600000, "to": 21600000 },
          { "key": "over_6h", "from": 21600000 }
        ]
      }
    }
  }
}
```

### 고유 디바이스 수

```json
GET /issue-history/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "workspace_id": "ws-123" } },
        { "range": { "occurred_at": { "gte": "now-30d" } } }
      ]
    }
  },
  "aggs": {
    "unique_devices": {
      "cardinality": { "field": "device_id" }
    }
  }
}
```

---

## 핵심 명령어 / API

```
# 기본 집계 구조
GET /index/_search { "size": 0, "aggs": { "name": { "type": { "field": "..." } } } }

# terms (필드 값별 그룹핑)
"terms": { "field": "status", "size": 10 }

# date_histogram (시간대별)
"date_histogram": { "field": "occurred_at", "calendar_interval": "day" }

# 빈 구간 포함
"date_histogram": { ..., "min_doc_count": 0, "extended_bounds": { "min": "...", "max": "..." } }

# range (범위별)
"range": { "field": "duration_ms", "ranges": [{ "to": 3600000 }, { "from": 3600000 }] }

# filter (단일 조건 버킷)
"filter": { "term": { "status": "open" } }

# filters (다중 조건)
"filters": { "filters": { "name1": { query }, "name2": { query } } }

# Metric aggs
"avg": { "field": "duration_ms" }
"max": { "field": "duration_ms" }
"stats": { "field": "duration_ms" }    // count+min+max+avg+sum 한 번에
"cardinality": { "field": "device_id" }

# Sub-Aggregation (중첩)
"aggs": { "bucket_name": { "terms": {...}, "aggs": { "metric_name": { "avg": {...} } } } }
```

---

## 헷갈렸던 점 & 해결

### 착각: Metric Agg를 단독으로 쓰면 "그룹별" 결과가 나온다
- **실제**: Metric만 단독으로 쓰면 "전체 통계 1개"만 나옴.
- **이유**: "~별"이 필요하면 반드시 Bucket Agg가 바깥을 감싸야 함. Bucket = GROUP BY 역할.
- **올바른 구조**: `terms(issue_code) { avg(duration_ms) }` — Bucket이 그룹을 만들고, 그 안에 Metric이 각 그룹의 통계를 계산.

### 착각: 중첩 집계에서 Bucket과 Metric을 같은 레벨에 병렬로 놓으면 "그룹별 통계"가 된다
- **실제**: 같은 레벨에 놓으면 각각 독립적으로 수행됨 (전체 대상 terms + 전체 대상 avg).
- **올바른 구조**: Bucket 안에 Metric을 넣어야 "해당 그룹 내에서의 통계"가 됨.
- **기억법**: `aggs: { bucket: { terms: {...}, aggs: { metric: { avg: {...} } } } }` — 안쪽 aggs가 "이 버킷 내에서" 계산됨.

### 착각: terms에 size 안 쓰면 전부 나온다
- **실제**: 기본 size는 10. 고유값이 15개여도 상위 10개만 반환.
- **확인법**: 응답의 `sum_other_doc_count`가 0이면 전부 반환된 것, 0 초과면 잘린 것.
- **한계**: terms의 size 최대값은 10,000. 초과하면 composite aggregation 필요.

### 착각: date_histogram에서 min_doc_count: 0만 쓰면 빈 날짜도 다 나온다
- **실제**: min_doc_count: 0은 중간의 빈 날짜만 채움. 양 끝에 데이터가 없는 날은 빠질 수 있음.
- **해결**: `extended_bounds`로 시작/끝 날짜를 강제 지정해야 완전한 빈 구간 처리.

### 체크 세션에서 몰랐던 것: sum_other_doc_count의 의미
- **정의**: terms의 size 한계로 반환되지 못한 나머지 버킷들의 문서 합계.
- **예시**: 고유값 15개인데 size:10 → 상위 10개 버킷 반환, 나머지 5개 버킷의 문서가 총 35건 → sum_other_doc_count: 35
- **대응**: 0이 아니면 size를 늘려야 함. 0이면 "전부 반환됐다"는 보장.

---

## 복습 포인트

1. **Aggregation의 핵심 구조는 "Bucket(바깥) → Metric(안쪽)"**이다. Bucket은 SQL의 GROUP BY, Metric은 AVG/SUM 등 집계 함수. "그룹별 통계"를 구하려면 반드시 Bucket이 Metric을 감싸야 한다. Metric만 단독으로 쓰면 전체 통계 1개만 나옴.

2. **`size: 0`은 집계 전용 쿼리의 필수 설정**이다. 문서 hits를 안 가져오므로 네트워크/메모리 절약. 대시보드 차트 데이터는 항상 size: 0.

3. **terms aggregation의 기본 size는 10, 최대 10,000**이다. 고유값이 10개 초과면 잘리며, 응답의 `sum_other_doc_count`로 잘림 여부를 확인. 0이면 전부 반환, 0 초과면 size를 늘려야 함. 10,000개 초과하면 composite 필요.

4. **date_histogram의 빈 구간 처리는 min_doc_count: 0 + extended_bounds 둘 다 필요**하다. min_doc_count만으로는 양 끝에 데이터 없는 날이 빠질 수 있음. extended_bounds로 시작/끝 날짜를 강제 지정.

5. **중첩 Aggregation은 Bucket 안에 또 다른 Agg를 넣는 구조**이다. terms(issue_code별) { avg(duration_ms) + date_histogram(일별) } 처럼 한 번의 쿼리로 다차원 분석 가능. 같은 레벨에 여러 agg를 병렬로 놓을 수도 있고(독립 실행), 안에 넣을 수도 있음(해당 그룹 내 계산).

6. **filters vs terms**: terms는 필드의 모든 고유값을 자동 그룹핑하고, filters는 직접 정의한 복합 조건별로 그룹핑(이름 지정 가능). Issue Trend의 도넛차트(Open/Resolved)처럼 명시적 조건이 필요하면 filters.

7. **cardinality는 근사치**이다. HyperLogLog++ 알고리즘 기반. 고유값 수천 이하면 거의 정확하지만, 수백만 이상이면 오차. 정확한 DISTINCT COUNT가 아님에 주의.

8. **Issue Trend Summary를 단일 쿼리로 구성**: query로 workspace + 기간 필터 → aggs 안에 terms(막대), date_histogram(선), filters(도넛)를 병렬 배치. 3번 쿼리할 필요 없이 1번으로 3개 차트 데이터 추출.
