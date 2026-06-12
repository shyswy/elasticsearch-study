## 2단계: 인덱스 설계 & 매핑

> **학습 목표**: 로그와 Issue History에 적합한 인덱스를 설계하고,
> 매핑(스키마)을 정의하여 검색/집계 성능을 최적화할 수 있다.

### 2-1. 매핑(Mapping)이란?

매핑 = 문서의 필드별 데이터 타입과 처리 방식을 정의하는 스키마

```json
PUT /issue-history
{
  "mappings": {
    "properties": {
      "issue_id":    { "type": "keyword" },
      "title":       { "type": "text", "analyzer": "standard" },
      "status":      { "type": "keyword" },
      "assignee":    { "type": "keyword" },
      "created_at":  { "type": "date" },
      "resolved_at": { "type": "date" },
      "resolution_time_minutes": { "type": "integer" },
      "tags":        { "type": "keyword" },
      "severity":    { "type": "keyword" },
      "description": { "type": "text" }
    }
  }
}
```

### 2-2. 핵심 필드 타입

| 타입 | 용도 | 검색 방식 | 집계 가능 |
|------|------|---------|---------|
| `text` | 풀텍스트 검색용 (로그 메시지, 설명) | 분석됨 (토큰화) | ✗ (직접 불가) |
| `keyword` | 정확한 값 매칭 (상태, ID, 태그) | 분석 안 됨 (원본 그대로) | ✓ |
| `date` | 날짜/시간 | 범위 검색 | ✓ |
| `integer/long` | 숫자 | 범위 검색 | ✓ |
| `boolean` | true/false | 필터 | ✓ |
| `nested` | 객체 배열 (독립 검색 필요 시) | 중첩 쿼리 | ✓ |
| `object` | 일반 객체 | 평탄화됨 | △ |

**text vs keyword 핵심 차이**:
```
"status": "In Progress"

text 타입으로 저장 시:
  → 토큰화: ["in", "progress"]
  → "in"으로 검색해도 매칭됨 (원치 않는 결과)

keyword 타입으로 저장 시:
  → 원본 그대로: "In Progress"
  → 정확히 "In Progress"로 검색해야 매칭
  → 집계(Aggregation)에서 그룹핑 가능
```

### 2-3. 로그 인덱스 설계

**시계열 인덱스 패턴** — 날짜별로 인덱스 분리:

```
logs-2025-07-01
logs-2025-07-02
logs-2025-07-03
...
```

**왜 날짜별로 분리하나?**
- 오래된 로그 삭제가 쉬움 (인덱스 통째로 삭제)
- 검색 범위를 특정 날짜로 한정 가능 (성능 향상)
- 저렴한 티어에서 스토리지 관리 필수

```json
PUT /logs-2025-07-15
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "@timestamp":  { "type": "date" },
      "level":       { "type": "keyword" },
      "service":     { "type": "keyword" },
      "message":     { "type": "text" },
      "trace_id":    { "type": "keyword" },
      "host":        { "type": "keyword" },
      "response_time_ms": { "type": "integer" }
    }
  }
}
```

> **저렴한 티어 팁**: `number_of_replicas: 0` — 단일 노드에서는 replica 불가.
> 노드가 2개 이상이면 1로 설정.

### 2-4. Issue History 인덱스 설계

```json
PUT /issue-history
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "issue_id":       { "type": "keyword" },
      "title":          { "type": "text", "fields": {
                            "keyword": { "type": "keyword" }
                          }},
      "status":         { "type": "keyword" },
      "priority":       { "type": "keyword" },
      "assignee":       { "type": "keyword" },
      "team":           { "type": "keyword" },
      "created_at":     { "type": "date" },
      "updated_at":     { "type": "date" },
      "resolved_at":    { "type": "date" },
      "resolution_time_minutes": { "type": "integer" },
      "tags":           { "type": "keyword" },
      "description":    { "type": "text" },
      "environment":    { "type": "keyword" },
      "affected_service": { "type": "keyword" }
    }
  }
}
```

**Multi-field 패턴** (`title` 필드):
```json
"title": {
  "type": "text",           // 풀텍스트 검색용
  "fields": {
    "keyword": { "type": "keyword" }  // 정확한 매칭 + 집계용
  }
}
```
→ `title`로 검색, `title.keyword`로 집계 가능

### 2-5. 인덱스 템플릿 (Index Template)

매일 생성되는 로그 인덱스에 매핑을 자동 적용:

```json
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "service":    { "type": "keyword" },
        "message":    { "type": "text" },
        "host":       { "type": "keyword" }
      }
    }
  },
  "priority": 100
}
```

→ `logs-2025-07-15` 인덱스가 생성될 때 자동으로 이 매핑 적용

### 2-6. 문서의 3가지 저장소 구조 (ES 내부 저장 모델)

ES는 하나의 문서를 **3가지 독립적인 저장소**에 동시에 저장한다. 이 구조를 이해해야 매핑 옵션(`index`, `doc_values`, `enabled`)의 의미를 정확히 알 수 있다.

```
문서: { "status": "open", "title": "서버 장애", "created_at": "2025-07-15" }

┌─────────────────────────────────────────────────────────┐
│ 저장소 1: _source (원본 JSON 보관)                        │
│                                                         │
│ 원본 JSON을 압축해서 그대로 보관                           │
│ { "status": "open", "title": "서버 장애", ... }          │
│                                                         │
│ 용도: 검색 결과 반환 시 문서 내용 제공                     │
│ 방향: 없음 (단순 보관)                                    │
│ 검색에는 사용 안 됨                                       │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 저장소 2: Inverted Index (역색인)                         │
│                                                         │
│ 값 → 문서 ID 목록 매핑                                   │
│ "open"  → [doc1, doc3]                                  │
│ "서버"  → [doc1]                                        │
│ "장애"  → [doc1]                                        │
│                                                         │
│ 용도: 검색 ("이 값을 포함하는 문서가 뭐야?")               │
│ 방향: 값 → 문서                                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 저장소 3: Doc Values (열 지향 저장소)                      │
│                                                         │
│ 문서 ID → 값 매핑 (columnar)                             │
│ status:     doc1→"open", doc2→"resolved", doc3→"open"   │
│ created_at: doc1→1752537600000, doc2→1752451200000      │
│                                                         │
│ 용도: 집계/정렬 ("이 문서의 값은?", "값별 건수 세줘")      │
│ 방향: 문서 → 값                                          │
│ keyword, numeric, date, boolean에 기본 활성화             │
│ text 타입은 기본 비활성화 (집계 대상이 아니므로)            │
└─────────────────────────────────────────────────────────┘
```

**필드 매핑 옵션과 저장소의 관계**:

| 옵션 | 역색인 | Doc Values | _source | 검색 | 집계/정렬 | 조회 |
|------|--------|-----------|---------|------|---------|------|
| 기본 (전부 켜짐) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `index: false` | ✗ | ✓ | ✓ | ✗ | ✓ | ✓ |
| `doc_values: false` | ✓ | ✗ | ✓ | ✓ | ✗ | ✓ |
| `enabled: false` | ✗ | ✗ | ✓ | ✗ | ✗ | ✓ |

### 2-7. 매핑 변경 규칙 & 인덱스 설정 변경 가능 여부

**매핑 변경 불가 규칙**:
- 기존 필드의 타입은 변경 불가 (text → keyword 불가)
- 새 필드 추가는 가능
- 타입을 바꾸려면 새 인덱스 생성 + Reindex 필요

**인덱스 설정 변경 가능 여부**:

| 설정 | 변경 가능 시점 | 비고 |
|------|-------------|------|
| `number_of_shards` | 생성 시에만 | 이후 변경 불가, reindex 필요 |
| `number_of_replicas` | 언제든 | `PUT /index/_settings`로 변경 |
| `refresh_interval` | 언제든 | 동적 변경 가능 |

### 2-8. 여러 인덱스를 묶는 방법 (인덱스 패턴, Alias, Data Stream)

날짜별로 분리된 인덱스를 검색/관리할 때 사용하는 3가지 방법:

**방법 1: 인덱스 패턴 (와일드카드)**:
```json
GET /logs-*/_search
// → logs-2025-07-13, logs-2025-07-14, ... 모두 검색
// 가장 단순. 패턴에 맞는 모든 인덱스 대상.
```

**방법 2: Alias (별칭)**:
```json
POST /_aliases
{
  "actions": [
    { "add": { "index": "logs-2025-07-14", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-15", "alias": "logs-recent" } }
  ]
}
// → GET /logs-recent/_search 로 등록된 인덱스만 검색
// 특정 인덱스만 선택적으로 묶기 가능
// 무중단 인덱스 전환에도 사용 (6단계에서 상세)
```

**방법 3: Data Stream (시계열 전용 추상 레이어)**:
```
Data Stream: "logs"
  ├── .ds-logs-2025.07.13-000001  (backing index)
  ├── .ds-logs-2025.07.14-000002  (backing index)
  └── .ds-logs-2025.07.15-000003  (write index, 최신)
```

```json
// Data Stream 생성을 위한 인덱스 템플릿
PUT /_index_template/logs-ds-template
{
  "index_patterns": ["logs"],
  "data_stream": {},
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text" }
      }
    }
  }
}

// Data Stream에 문서 색인 (자동으로 최신 backing index에 저장)
POST /logs/_doc
{
  "@timestamp": "2025-07-15T09:00:00Z",
  "message": "서버 시작"
}

// Data Stream 전체 검색
GET /logs/_search
```

**Data Stream 핵심 특성**:
- `@timestamp` 필드 필수
- append-only: 기존 문서 update/delete 불가 (로그에 적합)
- 자동 rollover: 조건(크기/시간/문서수) 충족 시 새 backing index 생성
- 검색은 data stream 이름으로 (내부 인덱스 이름 몰라도 됨)

**비교**:

| | 와일드카드 | Alias | Data Stream |
|--|-----------|-------|-------------|
| 범위 제어 | 패턴 매칭 전부 | 명시적 등록 | 자동 관리 |
| 쓰기 대상 | 직접 지정 | 쓰기 alias 설정 가능 | 자동 (최신 index) |
| Rollover | 수동/ISM | 수동/ISM | 내장 자동 |
| 문서 수정/삭제 | 가능 | 가능 | 불가 (append-only) |
| 적합한 데이터 | 범용 | 범용 | 시계열 (로그, 메트릭) |

**현재 프로젝트 적용**:
- 로그 → Data Stream 적합 (append-only, 자동 rollover)
- Issue History → 설계 방식에 따라 다름:
  - 이벤트 소싱 (상태 변경마다 새 문서 추가) → Data Stream 가능
  - 단일 문서 업데이트 (이슈당 1문서, status 필드 수정) → 일반 인덱스 + Alias

**Data Stream 선택 기준 (범용)**:
```
Data Stream이 적합한 경우:
  - 데이터가 append-only (한번 쓰면 수정 안 함)
  - @timestamp 기반 시계열 데이터
  - 자동 rollover + 수명주기 관리가 필요
  - 예: 로그, 메트릭, 이벤트 스트림, 감사 로그

일반 인덱스가 적합한 경우:
  - 기존 문서를 update/delete 해야 함
  - 시계열이 아닌 엔티티 데이터
  - 예: 사용자 프로필, 상품 카탈로그, 설정 데이터
```

### 2-9. Dynamic Mapping 기본 동작

매핑에 정의하지 않은 새 필드가 문서에 들어오면 ES가 자동으로 타입을 추론한다:

**JSON 타입별 기본 매핑 결과**:

| JSON 값 | 감지 타입 | ES 기본 매핑 |
|---------|---------|-------------|
| `"hello"` | string | text + keyword multi-field |
| `42` | integer | long |
| `3.14` | float | float |
| `true` | boolean | boolean |
| `"2025-07-15"` | date 패턴 감지 | date |
| `{ "a": 1 }` | object | object (내부 필드도 재귀적 dynamic mapping) |

**string의 기본 Dynamic Mapping 결과**:
```json
// 매핑에 없는 "unknown_field": "hello" 가 들어오면:
"unknown_field": {
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword",
      "ignore_above": 256
    }
  }
}
// → text + keyword 둘 다 생성 → 역색인 2개 + doc_values 1개
// → 대부분의 필드는 둘 중 하나만 필요 → 디스크/메모리 낭비
```

**Dynamic Mapping 제어 옵션**:
```json
PUT /my-index
{
  "mappings": {
    "dynamic": "strict"  // 또는 "true" (기본), "false", "runtime"
  }
}
```

| 값 | 동작 |
|----|------|
| `true` (기본) | 새 필드 자동 매핑 생성 + 색인 |
| `false` | 새 필드 _source에 저장만, 매핑/색인 안 함 |
| `strict` | 매핑에 없는 필드가 오면 에러 (거부) |
| `runtime` | 새 필드를 runtime field로 자동 생성 (색인 안 함) |

> **저렴한 티어 권장**: `dynamic: "strict"` 또는 `dynamic_templates`로 제어.
> 무분별한 dynamic mapping은 매핑 폭발 + 디스크 낭비의 원인.

### 2-10. 샤드 전략 (저렴한 티어)

```
규칙: 샤드 1개당 10~50GB가 적정

저렴한 티어 (t3.small, 10~20GB EBS):
  → 인덱스당 Primary Shard 1개로 충분
  → 일별 로그가 1GB 미만이면 주간/월간 인덱스도 고려

과도한 샤드의 문제:
  → 샤드 1개당 메모리 오버헤드 존재
  → 소규모 클러스터에서 샤드가 많으면 성능 저하
  → "1000개의 작은 인덱스" < "적절히 큰 소수의 인덱스"
```

### 2-11. 매핑 심화 패턴

**dynamic_templates — 동적 필드에 매핑 규칙 자동 적용**:
```json
PUT /logs-template
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      {
        "long_text_fields": {
          "match": "*_description",
          "mapping": {
            "type": "text",
            "fields": { "keyword": { "type": "keyword" } }
          }
        }
      }
    ],
    "properties": {
      "@timestamp": { "type": "date" }
    }
  }
}
```
→ 미리 정의하지 않은 string 필드는 자동으로 keyword로 매핑
→ `*_description` 패턴의 필드는 text + keyword multi-field로 매핑

**copy_to — 여러 필드를 하나로 합쳐서 검색**:
```json
{
  "mappings": {
    "properties": {
      "title":       { "type": "text", "copy_to": "search_all" },
      "description": { "type": "text", "copy_to": "search_all" },
      "tags":        { "type": "keyword", "copy_to": "search_all" },
      "search_all":  { "type": "text" }
    }
  }
}
// → "search_all" 필드 하나로 title + description + tags 통합 검색 가능
// → _source에는 포함되지 않음 (검색 전용)
```

**runtime fields — 색인 없이 쿼리 시점에 필드 생성**:
```json
GET /issue-history/_search
{
  "runtime_mappings": {
    "resolution_hours": {
      "type": "double",
      "script": {
        "source": "emit(doc['resolution_time_minutes'].value / 60.0)"
      }
    },
    "is_slow_resolution": {
      "type": "boolean",
      "script": {
        "source": "emit(doc['resolution_time_minutes'].value > 120)"
      }
    }
  },
  "query": {
    "term": { "is_slow_resolution": true }
  }
}
// → 디스크 공간 절약, 매핑 변경 없이 새 필드 사용
// → 단점: 매 쿼리마다 계산 → 빈번한 사용 시 정식 필드로 색인 권장
```

**flattened 타입 — 동적 키가 많은 객체 처리**:
```json
{
  "mappings": {
    "properties": {
      "metadata": { "type": "flattened" }
    }
  }
}
// 문서: { "metadata": { "any_key": "any_value", "dynamic_field": "123" } }
// → 매핑 폭발(mapping explosion) 방지
// → 모든 값이 keyword처럼 동작 (정확 매칭만 가능)
```

**필드 매핑 옵션 — 불필요한 기능 비활성화로 최적화**:
```json
{
  "properties": {
    "raw_log": {
      "type": "text",
      "index": false,        // 검색 불필요 → 색인 안 함 (저장만)
      "doc_values": false    // 집계/정렬 불필요 → 디스크 절약
    },
    "internal_id": {
      "type": "keyword",
      "doc_values": false    // 집계/정렬 안 할 필드
    },
    "debug_info": {
      "type": "object",
      "enabled": false       // 검색/집계 모두 불필요 → JSON 저장만
    }
  }
}
```

### 2-12. 학습 체크

- [ ] `text`와 `keyword` 타입의 차이를 실제 사용 예시로 설명할 수 있다
- [ ] ES의 3가지 저장소(_source, Inverted Index, Doc Values)의 역할과 방향을 설명할 수 있다
- [ ] 로그 데이터에 시계열 인덱스 패턴을 적용하는 이유를 설명할 수 있다
- [ ] Multi-field 매핑이 왜 유용한지 설명할 수 있다
- [ ] Index Template의 역할과 동작 방식을 설명할 수 있다
- [ ] 저렴한 티어에서 샤드 수를 최소화해야 하는 이유를 설명할 수 있다
- [ ] Issue History 인덱스의 매핑을 직접 작성할 수 있다
- [ ] `dynamic_templates`로 동적 필드 매핑 규칙을 설정할 수 있다
- [ ] `copy_to`를 활용한 통합 검색 필드를 설계할 수 있다
- [ ] `runtime fields`의 장단점과 사용 시점을 설명할 수 있다
- [ ] `index: false`, `doc_values: false`, `enabled: false`의 차이를 설명할 수 있다
- [ ] `flattened` 타입이 필요한 상황을 설명할 수 있다
- [ ] 매핑 변경 불가 규칙과 `number_of_shards` vs `number_of_replicas`의 변경 가능 여부를 설명할 수 있다
- [ ] 인덱스 패턴, Alias, Data Stream의 차이와 각각의 적합한 사용 시점을 설명할 수 있다
- [ ] Dynamic Mapping의 기본 동작(string → text+keyword)과 제어 옵션(`strict`, `false`)을 설명할 수 있다
- [ ] Data Stream이 로그에 적합하고 Issue History에 부적합한 이유를 설명할 수 있다

---
