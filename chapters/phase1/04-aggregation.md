## 4단계: Aggregation (집계)

> **학습 목표**: 로그와 Issue History 데이터에서 통계/분석 쿼리를 작성하여
> 대시보드에 필요한 데이터를 추출할 수 있다.

### 4-1. Aggregation이란?

SQL의 `GROUP BY` + 집계 함수에 해당. 검색 결과를 그룹핑하고 통계를 계산한다.

```
SQL:    SELECT status, COUNT(*) FROM issues GROUP BY status
ES:     terms aggregation on "status" field
```

### 4-2. Aggregation 기본 구조

```json
GET /issue-history/_search
{
  "size": 0,          // 문서 본문은 필요 없음 (집계 결과만)
  "aggs": {
    "my_agg_name": {  // 집계 이름 (자유롭게 지정)
      "terms": {      // 집계 타입
        "field": "status"
      }
    }
  }
}
```

### 4-3. 주요 Aggregation 종류

**Bucket Aggregation — 그룹핑**:

```json
// terms: 필드 값별 그룹핑 (GROUP BY)
"aggs": {
  "by_status": {
    "terms": { "field": "status" }
  }
}
// 결과: open: 15건, resolved: 42건, in_progress: 8건

// date_histogram: 시간 구간별 그룹핑
"aggs": {
  "daily_count": {
    "date_histogram": {
      "field": "@timestamp",
      "calendar_interval": "day"
    }
  }
}
// 결과: 7/13: 120건, 7/14: 95건, 7/15: 150건

// range: 숫자 범위별 그룹핑
"aggs": {
  "resolution_time_ranges": {
    "range": {
      "field": "resolution_time_minutes",
      "ranges": [
        { "to": 30 },
        { "from": 30, "to": 120 },
        { "from": 120 }
      ]
    }
  }
}
```

**Metric Aggregation — 통계 계산**:

```json
// avg: 평균
"aggs": {
  "avg_resolution_time": {
    "avg": { "field": "resolution_time_minutes" }
  }
}

// stats: 한 번에 count, min, max, avg, sum
"aggs": {
  "resolution_stats": {
    "stats": { "field": "resolution_time_minutes" }
  }
}

// cardinality: 고유값 수 (DISTINCT COUNT)
"aggs": {
  "unique_assignees": {
    "cardinality": { "field": "assignee" }
  }
}
```

### 4-4. 중첩 Aggregation (Sub-Aggregation)

```json
// "팀별 → 상태별 이슈 수 + 평균 해결 시간"
GET /issue-history/_search
{
  "size": 0,
  "aggs": {
    "by_team": {
      "terms": { "field": "team" },
      "aggs": {
        "by_status": {
          "terms": { "field": "status" }
        },
        "avg_resolution": {
          "avg": { "field": "resolution_time_minutes" }
        }
      }
    }
  }
}
```

결과 구조:
```
by_team:
  ├── backend팀: 25건
  │     ├── by_status: open(5), resolved(18), in_progress(2)
  │     └── avg_resolution: 95분
  └── frontend팀: 12건
        ├── by_status: open(3), resolved(8), in_progress(1)
        └── avg_resolution: 45분
```

### 4-5. 실전 집계 예시

**"최근 7일간 일별 에러 로그 수"**:
```json
GET /logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-7d" } } }
      ]
    }
  },
  "aggs": {
    "daily_errors": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "day"
      }
    }
  }
}
```

**"서비스별 평균 응답 시간 (상위 10개)"**:
```json
GET /logs-*/_search
{
  "size": 0,
  "aggs": {
    "by_service": {
      "terms": { "field": "service", "size": 10 },
      "aggs": {
        "avg_response": {
          "avg": { "field": "response_time_ms" }
        }
      }
    }
  }
}
```

**"이번 달 이슈 해결률 (resolved / total)"**:
```json
GET /issue-history/_search
{
  "size": 0,
  "query": {
    "range": { "created_at": { "gte": "2025-07-01" } }
  },
  "aggs": {
    "total": { "value_count": { "field": "issue_id" } },
    "resolved": {
      "filter": { "term": { "status": "resolved" } }
    }
  }
}
```

### 4-6. 학습 체크

- [ ] Bucket Aggregation과 Metric Aggregation의 차이를 설명할 수 있다
- [ ] `terms` aggregation으로 필드별 그룹핑을 수행할 수 있다
- [ ] `date_histogram`으로 시계열 데이터를 시간 구간별로 집계할 수 있다
- [ ] Sub-Aggregation을 사용해 다차원 집계를 작성할 수 있다
- [ ] 현재 프로젝트의 Issue History에서 "팀별 평균 해결 시간" 쿼리를 작성할 수 있다
- [ ] `size: 0`을 쓰는 이유를 설명할 수 있다

---
