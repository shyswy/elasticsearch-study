## 3단계: CRUD & 검색 쿼리 기초

> **학습 목표**: 문서의 생성/조회/수정/삭제를 수행하고,
> 기본 검색 쿼리를 작성하여 원하는 데이터를 찾을 수 있다.

### 3-1. 문서 CRUD

**Create (색인)**:
```json
// 단일 문서 색인 (ID 자동 생성)
POST /issue-history/_doc
{
  "issue_id": "ISS-001",
  "title": "서버 응답 지연",
  "status": "open",
  "created_at": "2025-07-15T09:00:00Z"
}

// 단일 문서 색인 (ID 지정)
PUT /issue-history/_doc/ISS-001
{
  "issue_id": "ISS-001",
  "title": "서버 응답 지연",
  "status": "open",
  "created_at": "2025-07-15T09:00:00Z"
}
```

**Read (조회)**:
```json
// ID로 단일 문서 조회
GET /issue-history/_doc/ISS-001

// 전체 문서 조회 (기본 10개)
GET /issue-history/_search
{
  "query": { "match_all": {} }
}
```

**Update (수정)**:
```json
// 부분 업데이트
POST /issue-history/_update/ISS-001
{
  "doc": {
    "status": "resolved",
    "resolved_at": "2025-07-15T11:30:00Z"
  }
}
```

**Delete (삭제)**:
```json
// 단일 문서 삭제
DELETE /issue-history/_doc/ISS-001

// 인덱스 전체 삭제
DELETE /logs-2025-07-01
```

**Bulk API (대량 작업)**:
```json
POST /_bulk
{"index": {"_index": "logs-2025-07-15"}}
{"@timestamp": "2025-07-15T09:00:00Z", "level": "ERROR", "message": "Connection timeout"}
{"index": {"_index": "logs-2025-07-15"}}
{"@timestamp": "2025-07-15T09:01:00Z", "level": "INFO", "message": "Retry successful"}
```

### 3-2. 검색 쿼리 구조

```json
GET /issue-history/_search
{
  "query": { ... },      // 어떤 문서를 찾을지
  "sort": [ ... ],       // 정렬 기준
  "from": 0,             // 페이지네이션 시작점
  "size": 10,            // 반환할 문서 수
  "_source": ["field1", "field2"]  // 반환할 필드 제한
}
```

### 3-3. 주요 쿼리 종류

**match — 풀텍스트 검색 (text 필드용)**:
```json
GET /logs-*/_search
{
  "query": {
    "match": {
      "message": "connection timeout"
    }
  }
}
// "connection" OR "timeout" 포함 문서 검색 (기본: OR)
```

**term — 정확한 값 매칭 (keyword 필드용)**:
```json
GET /issue-history/_search
{
  "query": {
    "term": {
      "status": "open"
    }
  }
}
```

**range — 범위 검색 (날짜, 숫자)**:
```json
GET /logs-*/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2025-07-15T00:00:00Z",
        "lt": "2025-07-16T00:00:00Z"
      }
    }
  }
}
```

**bool — 복합 조건 (AND/OR/NOT)**:
```json
GET /issue-history/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "status": "open" } },
        { "range": { "created_at": { "gte": "2025-07-01" } } }
      ],
      "must_not": [
        { "term": { "priority": "low" } }
      ],
      "should": [
        { "term": { "assignee": "홍길동" } }
      ],
      "filter": [
        { "term": { "environment": "production" } }
      ]
    }
  }
}
```

**bool 쿼리 구성요소**:

| 절 | 역할 | 점수 영향 |
|----|------|---------|
| `must` | AND 조건 (반드시 매칭) | ✓ 점수에 반영 |
| `must_not` | NOT 조건 (반드시 제외) | ✗ |
| `should` | OR 조건 (매칭 시 점수 가산) | ✓ 점수에 반영 |
| `filter` | AND 조건 (반드시 매칭) | ✗ 점수 무관 (캐시됨, 더 빠름) |

> **팁**: 점수(relevance)가 필요 없는 필터링은 `filter`를 쓰면 성능이 좋다.
> 로그 검색에서 날짜 범위, 서비스명 필터는 `filter`가 적합.

### 3-4. 실전 쿼리 예시

**"오늘 발생한 ERROR 로그 중 swagger-hub 서비스 것만"**:
```json
GET /logs-2025-07-15/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "term": { "service": "swagger-hub" } },
        { "range": { "@timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "size": 50
}
```

**"이번 달 미해결 이슈 중 priority가 high인 것"**:
```json
GET /issue-history/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "open" } },
        { "term": { "priority": "high" } },
        { "range": { "created_at": { "gte": "2025-07-01" } } }
      ]
    }
  },
  "sort": [{ "created_at": { "order": "desc" } }]
}
```

### 3-5. 학습 체크

- [ ] 문서를 색인(Create)하고 ID로 조회(Read)할 수 있다
- [ ] `match`와 `term` 쿼리의 차이를 설명할 수 있다
- [ ] `bool` 쿼리의 `must` / `filter` / `should` / `must_not`의 차이를 설명할 수 있다
- [ ] `filter`가 `must`보다 성능이 좋은 이유를 설명할 수 있다
- [ ] 날짜 범위 + 필드 조건을 조합한 검색 쿼리를 작성할 수 있다
- [ ] Bulk API로 여러 문서를 한 번에 색인할 수 있다

---
