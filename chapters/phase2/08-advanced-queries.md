## 8단계: 고급 쿼리 & 분석

> **학습 목표**: 복잡한 검색 요구사항을 해결하는 고급 쿼리 패턴을 익힌다.

### 8-1. 고급 검색 패턴

**Wildcard & Regex (주의: 성능 비용 높음)**:
```json
// wildcard — 패턴 매칭
{ "wildcard": { "message": "*timeout*" } }

// regexp — 정규식
{ "regexp": { "trace_id": "abc-[0-9]{4}-.*" } }

// ⚠️ 역색인을 활용하지 못함 → 전체 스캔에 가까움 → 가급적 피하기
```

**Fuzzy — 오타 허용 검색**:
```json
{ "fuzzy": { "title": { "value": "sever", "fuzziness": 1 } } }
// "sever" → "server" 매칭 (편집 거리 1)
```

**Highlight — 검색어 하이라이팅**:
```json
GET /logs-*/_search
{
  "query": { "match": { "message": "timeout error" } },
  "highlight": {
    "fields": { "message": {} }
  }
}
// 결과: "<em>timeout</em> <em>error</em> occurred at..."
```

**Multi-match — 여러 필드에서 동시 검색**:
```json
{
  "multi_match": {
    "query": "서버 장애",
    "fields": ["title^3", "description", "tags"],
    "type": "best_fields"
  }
}
// title에서 매칭 시 점수 3배 가중
```

### 8-2. Pipeline Aggregation

이전 집계 결과를 입력으로 사용하는 2차 집계:

```json
// "일별 에러 수의 이동 평균"
{
  "aggs": {
    "daily": {
      "date_histogram": { "field": "@timestamp", "calendar_interval": "day" },
      "aggs": {
        "error_count": {
          "filter": { "term": { "level": "ERROR" } }
        },
        "moving_avg_errors": {
          "moving_avg": {
            "buckets_path": "error_count._count",
            "window": 7
          }
        }
      }
    }
  }
}
```

### 8-3. Script 기반 쿼리/집계

```json
// Painless 스크립트로 런타임 계산
{
  "script_fields": {
    "resolution_hours": {
      "script": {
        "source": "doc['resolution_time_minutes'].value / 60.0"
      }
    }
  }
}

// 스크립트 기반 집계
{
  "aggs": {
    "custom_metric": {
      "scripted_metric": {
        "init_script": "state.total = 0",
        "map_script": "state.total += doc['resolution_time_minutes'].value",
        "combine_script": "return state.total",
        "reduce_script": "return states.stream().mapToLong(l -> l).sum()"
      }
    }
  }
}
```

### 8-4. 실전 쿼리 최적화 패턴

**constant_score — 점수 계산 완전 생략**:
```json
// filter만 필요하고 relevance score가 불필요할 때
GET /issue-history/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "bool": {
          "must": [
            { "term": { "status": "open" } },
            { "range": { "created_at": { "gte": "now-7d" } } }
          ]
        }
      },
      "boost": 1.0
    }
  }
}
// → 모든 매칭 문서에 동일한 점수(1.0) 부여
// → bool의 filter 절과 유사하지만, 최상위 쿼리로 명시적 사용 시 의도가 명확
```

**exists — 필드 존재 여부 필터**:
```json
// "resolved_at 필드가 있는 문서만" (해결된 이슈)
{
  "query": {
    "bool": {
      "filter": [
        { "exists": { "field": "resolved_at" } }
      ]
    }
  }
}

// "assignee가 없는 문서" (미배정 이슈)
{
  "query": {
    "bool": {
      "must_not": [
        { "exists": { "field": "assignee" } }
      ]
    }
  }
}
```

**terms lookup — 다른 인덱스의 값을 참조하는 필터**:
```json
// "team-alerts 인덱스에 등록된 서비스들의 로그만 조회"
GET /logs-*/_search
{
  "query": {
    "terms": {
      "service": {
        "index": "team-alerts",
        "id": "backend-team",
        "path": "monitored_services"
      }
    }
  }
}
// → team-alerts 인덱스의 backend-team 문서에서
//   monitored_services 필드 값을 가져와서 terms 필터로 사용
```

**minimum_should_match — should 절 최소 매칭 수 제어**:
```json
// "tags 중 최소 2개 이상 매칭되는 이슈"
{
  "query": {
    "bool": {
      "should": [
        { "term": { "tags": "performance" } },
        { "term": { "tags": "backend" } },
        { "term": { "tags": "database" } },
        { "term": { "tags": "timeout" } }
      ],
      "minimum_should_match": 2
    }
  }
}
// → 4개 조건 중 최소 2개 이상 매칭되어야 결과에 포함
```

**match_phrase — 어순 보장 검색**:
```json
// "connection timeout"이 정확히 이 순서로 나오는 문서만
{
  "query": {
    "match_phrase": {
      "message": "connection timeout"
    }
  }
}
// match: "connection" OR "timeout" (순서 무관)
// match_phrase: "connection timeout" (순서 보장, 인접해야 함)

// slop으로 단어 사이 허용 거리 설정
{
  "match_phrase": {
    "message": {
      "query": "connection timeout",
      "slop": 2
    }
  }
}
// → "connection refused timeout" 도 매칭 (사이에 1단어)
```

**index sorting — 색인 시점 정렬 (범위 쿼리 최적화)**:
```json
// 인덱스 생성 시 정렬 기준 설정
PUT /issue-history-v2
{
  "settings": {
    "index": {
      "sort.field": ["created_at", "status"],
      "sort.order": ["desc", "asc"]
    }
  },
  "mappings": { ... }
}
// → 문서가 색인될 때 created_at desc, status asc 순으로 물리적 정렬
// → "최근 N건" 같은 쿼리에서 early termination 가능 → 성능 향상
// ⚠️ 인덱스 생성 시에만 설정 가능, 이후 변경 불가
```

**named queries — 어떤 절이 매칭됐는지 확인**:
```json
GET /issue-history/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "status": { "value": "open", "_name": "is_open" } } },
        { "term": { "priority": { "value": "critical", "_name": "is_critical" } } },
        { "range": { "created_at": { "gte": "now-1d", "_name": "is_recent" } } }
      ]
    }
  }
}
// 결과의 각 문서에 matched_queries 필드 포함:
// "matched_queries": ["is_open", "is_recent"]
// → 디버깅, UI에서 매칭 이유 표시에 유용
```

### 8-5. 학습 체크

- [ ] wildcard/regex 쿼리의 성능 문제를 설명하고 대안을 제시할 수 있다
- [ ] Multi-match에서 필드 가중치(boost)를 활용할 수 있다
- [ ] Pipeline Aggregation으로 이동 평균을 계산할 수 있다
- [ ] Painless 스크립트로 런타임 필드를 생성할 수 있다
- [ ] Highlight를 활용한 검색 결과 표시를 구현할 수 있다
- [ ] `constant_score`와 `bool filter`의 차이를 설명할 수 있다
- [ ] `exists` 쿼리로 필드 유무를 필터링할 수 있다
- [ ] `terms lookup`으로 다른 인덱스의 값을 참조하는 쿼리를 작성할 수 있다
- [ ] `minimum_should_match`의 동작 방식을 설명할 수 있다
- [ ] `match`와 `match_phrase`의 차이를 설명할 수 있다
- [ ] `index sorting`이 쿼리 성능에 미치는 영향을 설명할 수 있다
- [ ] `named queries`를 활용한 매칭 디버깅을 수행할 수 있다

---
