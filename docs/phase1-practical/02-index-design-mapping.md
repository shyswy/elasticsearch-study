# 2단계: 인덱스 설계 & 매핑

## 학습 일자
2026-05-08

## 핵심 개념 정리

### 1. 매핑(Mapping) — 왜 존재하는가

**정의**: 인덱스 내 문서의 각 필드가 어떤 데이터 타입이고, 어떻게 저장·색인·검색될지를 정의하는 스키마.

**문제**: ES는 "스키마리스"라고 불리지만, 매핑 없이 문서를 넣으면 Dynamic Mapping이 자동으로 타입을 추론한다. 이게 문제를 만든다:
- string 필드 → text + keyword 둘 다 생성 → 역색인 2개 + doc_values 1개 → 디스크/메모리 낭비
- 의도하지 않은 검색 결과 (text로 토큰화되어 "in"으로 검색해도 "In Progress" 매칭)
- 나중에 타입을 바꾸려면 reindex 필요

**핵심 규칙**: 매핑은 한번 설정하면 기존 필드의 타입을 변경할 수 없다. 새 필드 추가는 가능하지만, text → keyword 변경은 불가. 바꾸려면 새 인덱스 생성 + reindex + alias 전환이라는 비용이 큰 작업이 필요. 그래서 처음에 제대로 설계하는 게 중요하다.

---

### 2. ES의 3가지 저장소 구조

ES는 하나의 문서를 3가지 독립적인 저장소에 동시에 저장한다. 이 구조를 이해해야 매핑 옵션의 의미를 정확히 알 수 있다.

**저장소 1: _source (원본 JSON 보관)**
- 원본 JSON을 압축해서 그대로 보관하는 저장소
- 용도: 검색 결과 반환 시 문서 내용 제공
- 방향: 없음 (단순 보관함)
- 검색에는 사용 안 됨. "이 문서의 원본 내용을 보여줘"할 때만 사용

**저장소 2: Inverted Index (역색인)**
- "값 → 그 값을 포함하는 문서 ID 목록"의 매핑 테이블
- 용도: 검색 ("이 값을 포함하는 문서가 뭐야?")
- 방향: 값 → 문서
- text 타입: Analyzer를 거쳐 토큰화된 각 단어가 역색인에 등록
- keyword 타입: 원본 값 전체가 하나의 토큰으로 역색인에 등록
- 역방향(문서 → 값) 조회는 불가

**저장소 3: Doc Values (열 지향 저장소)**
- "문서 ID → 값"의 columnar 매핑
- 용도: 집계/정렬 ("status별 건수 세줘", "created_at 순으로 정렬해줘")
- 방향: 문서 → 값
- keyword, numeric, date, boolean에 기본 활성화
- text 타입은 기본 비활성화 (text는 집계/정렬 대상이 아니므로)

**핵심**: 검색은 역색인을 사용하고, 결과 반환은 _source를 사용하고, 집계/정렬은 doc_values를 사용한다. 이 3가지가 독립적이라서 각각 끄고 켤 수 있다.

---

### 3. text vs keyword — 가장 중요한 구분

둘 다 역색인을 사용하지만, 역색인에 저장되는 방식이 다르다.

**text 타입**:
- 정의: 풀텍스트 검색을 위해 Analyzer를 거쳐 토큰화되어 역색인에 저장되는 타입
- 내부 동작: `"In Progress"` → standard analyzer → lowercase → `["in", "progress"]`
- 역색인: `"in" → [doc1]`, `"progress" → [doc1]`
- "in"으로 검색해도 매칭됨 (토큰 단위 매칭)
- 집계/정렬 불가 (원본 값이 보존되지 않으므로, doc_values도 기본 비활성)
- 적합: 로그 메시지, 이슈 설명, 제목 등 "문장 안에서 단어를 찾아야 하는" 필드

**keyword 타입**:
- 정의: 분석 없이 원본 값 그대로 저장되어, 정확한 값 매칭·집계·정렬에 사용되는 타입
- 내부 동작: `"In Progress"` → 분석 없음 → `"In Progress"` 그대로
- 역색인: `"In Progress" → [doc1, doc3]`
- doc_values: `doc1→"In Progress"`, `doc2→"Open"`
- 정확히 "In Progress"로 검색해야 매칭. "in"이나 "progress"로는 안 됨
- 집계/정렬 가능 (doc_values 활성)
- 적합: 상태값, ID, 태그, 서비스명, 호스트명 등 "정확한 값으로 필터링/그룹핑하는" 필드

**판단 기준**:
```
"문장 안에서 단어를 검색한다" → text
"정확한 값으로 필터링한다"   → keyword
"이 필드로 집계(GROUP BY)한다" → keyword
"이 필드로 정렬한다"         → keyword
"검색도 하고 집계도 한다"    → multi-field (text + keyword)
```

**긴 문자열의 타입 선택**: 길이가 아니라 용도가 기준.
- 로그 메시지 500자, 에러 스택트레이스 2000자 → text (문장 안 단어 검색)
- URL, 파일 경로, 긴 ID → keyword (정확 매칭)
- `ignore_above: 256`: 256자 초과 값은 keyword 역색인에 색인하지 않음 (메모리 절약). _source에는 저장됨.

---

### 4. Multi-field 패턴

**문제**: `title` 필드를 풀텍스트 검색도 하고 싶고, 집계(정확한 값 기준 그룹핑)도 하고 싶다면?

**해결**: 하나의 필드를 두 가지 방식으로 동시에 색인:

```json
"title": {
  "type": "text",
  "fields": {
    "keyword": { "type": "keyword", "ignore_above": 256 }
  }
}
```

**사용법**:
- 검색(풀텍스트): `{ "match": { "title": "서버" } }` ← text 필드 사용
- 집계/정렬: `{ "terms": { "field": "title.keyword" } }` ← keyword sub-field 사용

내부적으로 역색인 2개(text용 + keyword용) + doc_values 1개(keyword용)가 생성됨.

---

### 5. date 타입

**정의**: 날짜/시간 값을 내부적으로 epoch milliseconds(long)로 변환하여 저장하는 타입.

**왜 별도 타입이 필요한가**:
- 내부적으로 숫자(밀리초)로 저장 → 범위 검색이 숫자 비교 → 빠름
  - 숫자 비교: 단일 CPU 명령어로 끝남 (1번의 연산)
  - 문자열 비교: 한 글자씩 순차 비교 (N번의 연산)
- `date_histogram` 집계(일별/주별/월별 그룹핑) 가능 — integer에 epoch millis를 넣어도 date_histogram은 동작 안 함
- `now-7d` 같은 날짜 수학(date math) 표현 사용 가능
- 다양한 포맷 자동 파싱 ("2025-07-15T09:00:00Z", epoch millis 등)

**시점 vs 양의 구분**:
- 시점(언제 발생?) → date: `@timestamp`, `created_at`, `resolved_at`
- 양(얼마나?) → integer: `response_time_ms`, `resolution_time_minutes`
- `response_time_ms`는 API 요청의 응답 시간(밀리초). "이 요청이 얼마나 걸렸나"라는 지속 시간(duration)이지 시점이 아님.

---

### 6. 시계열 인덱스 패턴

로그처럼 계속 쌓이는 데이터는 날짜별(또는 주/월별)로 인덱스를 분리한다.

**분리하는 이유 3가지** (각각 독립적인 이점):
1. **삭제 효율**: 인덱스 통째 삭제 → 즉시 디스크 회수. 문서 단위 삭제는 Segment의 .del 파일에 마킹만 하고, merge가 일어나야 비로소 물리적 삭제 + 디스크 회수. merge 시점은 예측 불가.
2. **검색 범위 한정**: `GET /logs-2025-07-15/_search` → 오늘 인덱스의 샤드만 탐색. `logs-*`로 전체를 검색하면 모든 인덱스의 모든 샤드에 쿼리를 보냄. 불필요한 샤드에 쿼리를 안 보내는 것 자체가 성능 향상.
3. **ISM 정책 적용 용이**: "30일 지난 인덱스 자동 삭제" 같은 수명주기 관리를 인덱스 단위로 적용 가능.

**일별 vs 주별 vs 월별 선택 기준**:
- 일별 로그량 > 1GB → 일별 인덱스
- 일별 로그량 < 100MB → 주별 또는 월별 (너무 작은 인덱스가 많으면 샤드 오버헤드)

---

### 7. 샤드 전략

각 샤드는 존재만으로 비용을 발생시킨다:
- JVM Heap 메모리 소비 (샤드당 수십 MB의 메타데이터)
- 파일 디스크립터 점유
- 검색 시 각 샤드에 쿼리를 보내고 결과를 병합하는 오버헤드

적정 샤드 크기: 10~50GB/샤드.

**설정 변경 가능 여부**:
- `number_of_shards`: 인덱스 생성 시에만 설정 가능, 이후 변경 불가 (reindex 필요)
- `number_of_replicas`: 언제든 변경 가능 (`PUT /index/_settings`)
- `refresh_interval`: 언제든 변경 가능

---

### 8. 여러 인덱스를 묶는 방법

날짜별로 분리된 인덱스를 검색/관리할 때 사용하는 3가지 방법:

**방법 1: 인덱스 패턴 (와일드카드)**
```
GET /logs-*/_search
```
- 패턴에 맞는 모든 인덱스 대상. 별도 설정 불필요.
- 단점: 범위를 세밀하게 제어할 수 없음 (패턴 매칭 전부)

**방법 2: Alias (별칭)**
- 하나 이상의 인덱스에 붙이는 가상의 이름. 실제 인덱스가 아니라 포인터.
- 명시적으로 등록한 인덱스만 묶음 (선택적 제어)
- 무중단 인덱스 전환에 사용 (reindex 후 alias만 교체 → 애플리케이션 코드 변경 없음)
- 쓰기 대상 지정 가능 (`is_write_index: true`)
- 비용 없음 (이름 매핑만, 데이터 복사 아님)

**방법 3: Data Stream (시계열 전용 추상 레이어)**
- 내부적으로 여러 backing index를 자동 생성/관리하는 상위 개념
- 인덱스 생성·롤오버·삭제를 자동으로 관리 (직접 날짜별 인덱스를 만들 필요 없음)
- `@timestamp` 필드 필수
- append-only: 기존 문서 update/delete 불가
- 검색은 data stream 이름으로, 쓰기는 자동으로 최신 backing index에
- 롤오버: 조건(크기/시간/문서수) 충족 시 자동으로 새 backing index 생성

```
Data Stream: "logs"
  ├── .ds-logs-000001 (backing index, read-only)
  ├── .ds-logs-000002 (backing index, read-only)
  └── .ds-logs-000003 (write index ← 현재 쓰기 대상)
```

**비교**:

| | 와일드카드 | Alias | Data Stream |
|--|-----------|-------|-------------|
| 설정 필요 | 없음 | alias 등록 | 인덱스 템플릿 + data_stream |
| 범위 제어 | 패턴 매칭 전부 | 명시적 등록만 | 자동 (전체 backing index) |
| 쓰기 대상 | 직접 지정 | is_write_index | 자동 (최신) |
| Rollover | 수동/ISM | 수동/ISM | 내장 자동 |
| 문서 수정/삭제 | 가능 | 가능 | 불가 (append-only) |
| 운영 부담 | 인덱스 직접 관리 | 인덱스+alias 관리 | 최소 (자동화) |

**Data Stream 선택 기준**:
- 적합: append-only 시계열 데이터 (로그, 메트릭, 이벤트 스트림, 감사 로그)
- 부적합: 기존 문서를 update/delete 해야 하는 엔티티 데이터
- 단, 이벤트 소싱 방식(상태 변경마다 새 문서 추가)이면 엔티티 데이터도 Data Stream 가능

---

### 9. Dynamic Mapping 기본 동작

매핑에 정의하지 않은 새 필드가 문서에 들어오면 ES가 값을 보고 자동으로 타입을 추론한다.

**JSON 타입별 기본 매핑 결과**:

| JSON 값 | ES 기본 매핑 |
|---------|-------------|
| `"hello"` | text + keyword multi-field (낭비!) |
| `42` | long |
| `3.14` | float |
| `true` | boolean |
| `"2025-07-15"` | date (패턴 감지) |
| `{ "a": 1 }` | object (내부 필드도 재귀적 dynamic mapping) |

string의 기본 결과가 text + keyword 둘 다 생성되는 게 핵심 문제. 대부분의 필드는 둘 중 하나만 필요.

**제어 옵션** (`"dynamic"` 설정):
- `true` (기본): 새 필드 자동 매핑 생성 + 색인
- `false`: _source에 저장만, 매핑/색인 안 함 (검색 불가, 조회는 가능)
- `strict`: 매핑에 없는 필드가 오면 에러 (거부)
- `runtime`: 새 필드를 runtime field로 자동 생성 (색인 안 함, 쿼리 시 계산)

---

### 10. 인덱스 템플릿 (Index Template)

**문제**: 매일 `logs-2025-07-15`, `logs-2025-07-16`... 인덱스가 생성되는데, 매번 매핑을 수동으로 정의할 수 없다.

**해결**: 인덱스 템플릿을 등록하면, 패턴에 맞는 인덱스가 생성될 때 자동으로 매핑/설정이 적용된다.

- `index_patterns`: 어떤 이름의 인덱스에 적용할지 (글로브 패턴)
- `priority`: 여러 템플릿이 같은 패턴에 매칭될 때 우선순위 (높은 숫자 우선)
- Data Stream 생성 시에도 인덱스 템플릿이 필수 (`"data_stream": {}` 옵션 추가)

---

### 11. 매핑 심화: dynamic_templates

매핑에 명시하지 않은 새 필드에 대한 자동 매핑 규칙을 정의:

**문법 구조**:
```json
"dynamic_templates": [
  {
    "규칙_이름(자유롭게_변경_가능)": {
      "match_mapping_type": "string",   ← ES 예약 키 (변경 불가)
      "match": "*_description",          ← ES 예약 키
      "unmatch": "*.raw",                ← ES 예약 키
      "mapping": { ... }                 ← ES 예약 키
    }
  }
]
```

- 규칙 이름: 사람이 읽기 위한 라벨. 아무 이름이나 가능.
- 예약 키 4개: `match_mapping_type`, `match`, `unmatch`, `mapping`
- 여러 규칙이 있으면 배열 순서대로 첫 번째 매칭되는 규칙 적용

---

### 12. 매핑 심화: copy_to

여러 필드의 값을 하나의 검색 전용 필드로 합침:

```json
"title":       { "type": "text", "copy_to": "search_all" },
"description": { "type": "text", "copy_to": "search_all" },
"search_all":  { "type": "text" }
```

- 역색인에는 search_all 토큰이 등록됨 → 검색 가능
- _source에는 포함 안 됨 → 결과 반환 시 안 보임
- 이게 가능한 이유: 검색은 역색인 사용, 결과 반환은 _source 사용 → 독립적

---

### 13. 매핑 심화: runtime fields

색인 시점이 아닌 쿼리 시점에 동적으로 계산되는 가상 필드:

- 매핑 변경 없이 새 필드 즉시 사용 가능
- 디스크 공간 절약 (색인하지 않으니까)
- "이 필드가 정말 필요한지" 검증 후 정식 매핑으로 승격 가능
- 단점: 매 쿼리마다 계산 → 자주 쓰는 필드라면 정식으로 색인하는 게 성능상 유리

---

### 14. 매핑 심화: flattened 타입

**문제**: 동적 키가 무한히 늘어날 수 있는 객체 (사용자 정의 메타데이터, HTTP 헤더 등). Dynamic Mapping이 각 키마다 필드를 생성하면 매핑 폭발(mapping explosion) → 클러스터 불안정.

**해결**: `"type": "flattened"` — 객체 전체를 하나의 필드로 취급.
- 매핑에 1개 필드만 등록됨 (키가 아무리 늘어도)
- 모든 값이 keyword처럼 동작 (정확 매칭만, 범위/풀텍스트 불가)

**flattened ≠ "dynamic_templates로 모든 string을 keyword로"**:
- dynamic_templates: 새 키마다 매핑에 필드 추가됨 → 매핑 폭발 위험 여전
- flattened: 매핑 1개 고정 → 매핑 폭발 없음. 핵심 차이는 매핑 수가 증가하느냐 vs 고정이냐.

---

### 15. 필드 매핑 옵션 (저장소 제어)

3가지 저장소를 각각 끄는 옵션:

| 옵션 | 끄는 저장소 | 검색 | 집계/정렬 | 조회 | 사용 예 |
|------|-----------|------|---------|------|--------|
| 기본 (전부 켜짐) | — | ✓ | ✓ | ✓ | — |
| `index: false` | 역색인 | ✗ | ✓ | ✓ | 검색 불필요, 집계만 하는 필드 |
| `doc_values: false` | Doc Values | ✓ | ✗ | ✓ | 검색만 하고 집계/정렬 안 하는 필드 |
| `enabled: false` | 역색인 + Doc Values + 파싱 | ✗ | ✗ | ✓ | 저장만, 검색/집계 모두 불필요 |

**주의**: `index: false`는 역색인만 끈다. doc_values는 여전히 켜져 있으므로 집계/정렬은 가능. 이 구분이 중요.

---

## 실전 쿼리/설정 예시

### 로그 인덱스 매핑
```json
PUT /logs-2025-07-15
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "@timestamp":       { "type": "date" },
      "level":            { "type": "keyword" },
      "service":          { "type": "keyword" },
      "message":          { "type": "text" },
      "trace_id":         { "type": "keyword" },
      "host":             { "type": "keyword" },
      "response_time_ms": { "type": "integer" }
    }
  }
}
```

### Issue History 인덱스 매핑
```json
PUT /issue-history
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "issue_id":                  { "type": "keyword" },
      "title":                     { "type": "text", "fields": {
                                       "keyword": { "type": "keyword" }
                                     }},
      "status":                    { "type": "keyword" },
      "priority":                  { "type": "keyword" },
      "assignee":                  { "type": "keyword" },
      "team":                      { "type": "keyword" },
      "created_at":                { "type": "date" },
      "updated_at":                { "type": "date" },
      "resolved_at":               { "type": "date" },
      "resolution_time_minutes":   { "type": "integer" },
      "tags":                      { "type": "keyword" },
      "description":               { "type": "text" },
      "environment":               { "type": "keyword" },
      "affected_service":          { "type": "keyword" }
    }
  }
}
```

### 인덱스 템플릿
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

### Data Stream 생성
```json
PUT /_index_template/logs-ds-template
{
  "index_patterns": ["logs"],
  "data_stream": {},
  "template": {
    "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "service":    { "type": "keyword" },
        "message":    { "type": "text" }
      }
    }
  }
}

// 색인 (Data Stream 자동 생성)
POST /logs/_doc
{ "@timestamp": "2025-07-15T09:00:00Z", "level": "ERROR", "message": "Connection timeout" }

// 검색
GET /logs/_search
{ "query": { "term": { "level": "ERROR" } } }
```

### Alias 생성/전환
```json
// alias 생성
POST /_aliases
{
  "actions": [
    { "add": { "index": "logs-2025-07-14", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-15", "alias": "logs-recent" } }
  ]
}

// 무중단 인덱스 전환 (v1 → v2)
POST /_aliases
{
  "actions": [
    { "remove": { "index": "issue-history-v1", "alias": "issue-current" } },
    { "add":    { "index": "issue-history-v2", "alias": "issue-current" } }
  ]
}
```

### dynamic_templates
```json
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword", "ignore_above": 256 }
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
    ]
  }
}
```

### copy_to
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
```

### runtime fields
```json
GET /issue-history/_search
{
  "runtime_mappings": {
    "resolution_hours": {
      "type": "double",
      "script": { "source": "emit(doc['resolution_time_minutes'].value / 60.0)" }
    }
  },
  "query": { "range": { "resolution_hours": { "gte": 2.0 } } }
}
```

### 필드 옵션 최적화
```json
{
  "properties": {
    "raw_log":     { "type": "text", "index": false },
    "internal_id": { "type": "keyword", "doc_values": false },
    "debug_info":  { "type": "object", "enabled": false }
  }
}
```

---

## 핵심 명령어 / API

```
# 인덱스 생성 (매핑 + 설정)
PUT /인덱스명 { "settings": {...}, "mappings": {...} }

# 매핑 확인
GET /인덱스명/_mapping

# 인덱스 설정 변경 (동적 설정만)
PUT /인덱스명/_settings { "index.number_of_replicas": 1 }

# 인덱스 템플릿 생성
PUT /_index_template/템플릿명 { "index_patterns": [...], "template": {...} }

# Alias 관리
POST /_aliases { "actions": [...] }
GET /_alias/alias명

# Data Stream
GET /_data_stream/logs
POST /logs/_rollover

# 인덱스 목록 확인
GET /_cat/indices?v&s=index
```

---

## 헷갈렸던 점 & 해결

**착각: text의 토큰화 시 대소문자가 유지된다**
→ 실제: standard analyzer는 lowercase 필터를 포함하므로 "In Progress" → ["in", "progress"]로 소문자 변환됨.
→ 이유: 검색 시 대소문자 구분 없이 매칭하기 위해 기본 analyzer가 lowercase를 적용.

**착각: `index: false`로 설정하면 집계도 안 된다**
→ 실제: `index: false`는 역색인만 끈다. Doc Values는 여전히 켜져 있으므로 집계/정렬은 가능.
→ 이유: 역색인과 doc_values는 독립적인 저장소. 각각 별도로 끄고 켤 수 있다.

**착각: flattened는 dynamic_templates로 모든 string을 keyword로 매핑한 것과 같다**
→ 실제: dynamic_templates는 새 키마다 매핑에 필드가 추가됨 (매핑 폭발 위험). flattened는 매핑 1개로 고정.
→ 이유: flattened는 객체 전체를 하나의 필드로 취급하므로 내부 키가 아무리 늘어도 매핑 수는 변하지 않음.

**체크 세션에서 빠뜨린 것: multi-field에서 검색/집계 필드명**
→ 검색(풀텍스트): `{ "match": { "title": "서버" } }` ← text 필드
→ 집계/정렬: `{ "terms": { "field": "title.keyword" } }` ← keyword sub-field

**체크 세션에서 빠뜨린 것: 날짜별 인덱스 분리 이유 3가지 중 2개 누락**
→ 삭제 효율만 답함. 검색 범위 한정(성능)과 ISM 정책 적용 용이(운영 자동화)를 빠뜨림.
→ 2번(검색 범위 한정)은 삭제와 무관한 독립적 이점. 불필요한 샤드에 쿼리를 보내지 않는 것.

---

## 복습 포인트

**매핑은 변경 불가**: 기존 필드 타입을 바꿀 수 없다. 새 필드 추가만 가능. 바꾸려면 새 인덱스 생성 + reindex + alias 전환. number_of_shards도 생성 시에만 설정 가능(변경 불가). number_of_replicas와 refresh_interval만 언제든 변경 가능.

**3가지 저장소**: _source(원본 보관, 결과 반환용), 역색인(값→문서, 검색용), Doc Values(문서→값, 집계/정렬용). 이 3개가 독립적이라서 `index: false`(역색인 끔, 집계는 가능), `doc_values: false`(Doc Values 끔, 검색은 가능), `enabled: false`(둘 다 끔)로 각각 제어 가능. copy_to가 "_source에 없지만 검색 가능"한 이유도 이 구조 때문.

**text vs keyword**: text는 Analyzer를 거쳐 토큰화(소문자 변환 포함)되어 역색인에 저장 → 부분 단어 검색 가능, 집계 불가. keyword는 원본 그대로 역색인 + doc_values에 저장 → 정확 매칭만, 집계/정렬 가능. 둘 다 역색인을 사용하지만 저장 방식이 다름. 둘 다 필요하면 multi-field 패턴. 검색은 `title`(text), 집계는 `title.keyword`(keyword).

**시계열 인덱스 분리 3가지 이점**: 삭제 효율(인덱스 통째 삭제 = 즉시 디스크 회수, 문서 단위 삭제 = .del 마킹 후 merge까지 회수 안 됨), 검색 범위 한정(해당 인덱스 샤드만 탐색 = 성능), ISM 정책 적용(자동 삭제 규칙). 1번과 3번은 같은 맥락(삭제 관련), 2번은 완전히 다른 축(검색 성능).

**여러 인덱스 묶기**: 와일드카드(`logs-*`, 가장 단순, 범위 제어 불가), Alias(명시적 등록, 무중단 전환, 비용 없음), Data Stream(시계열 전용, append-only 강제, @timestamp 필수, 자동 rollover = 기존 index readonly + 새 인덱스를 만들어 거기에 계속 쓰기 시작하는 것). Data Stream의 append-only 제약 때문에 문서 수정이 필요한 데이터에는 부적합하지만, 이벤트 소싱 방식이면 가능.

**Dynamic Mapping**: string → text + keyword multi-field가 기본 (낭비). 제어: `dynamic: strict`(미정의 필드 거부), `dynamic_templates`(패턴별 규칙 지정), `flattened`(동적 키 객체를 매핑 1개로 고정). flattened의 핵심 가치는 매핑 수 고정 = 매핑 폭발 방지.

**date vs integer**: date는 시점(언제?), integer는 양(얼마나?). date만 date_histogram 집계, date math(`now-7d`), 포맷 파싱을 지원. 내부적으로 epoch millis(숫자)로 저장되어 범위 검색이 빠름(숫자 비교 = 단일 CPU 명령어).
