# 3단계: CRUD & 검색 쿼리 기초

## 학습 일자
2026-05-08

## 핵심 개념 정리

### 색인(Indexing) — "저장"이 아니라 "검색 가능하게 만드는 것"

색인이란 JSON 문서를 특정 인덱스에 저장하면서, 동시에 해당 문서의 텍스트 필드를 분석(Analysis)하여 역색인을 생성하는 과정이다. RDB의 INSERT에 해당하지만, 단순 저장이 아니라 "검색 가능한 상태로 만드는 것"까지 포함한다. ES에서 "색인한다(index)"라고 표현하는 이유는 저장하는 순간 역색인(Inverted Index)이 업데이트되기 때문이다.

**두 가지 색인 방식**:
- `POST /index/_doc` (ID 자동 생성): 로그처럼 개별 문서를 나중에 업데이트할 필요가 없는 경우에 적합
- `PUT /index/_doc/{id}` (ID 직접 지정): Issue History처럼 특정 문서를 나중에 조회/업데이트해야 하는 경우에 적합

**PUT의 덮어쓰기 특성**: 같은 `_id`로 PUT을 다시 호출하면 기존 문서를 완전히 덮어쓴다(replace). 명시하지 않은 필드는 사라진다. 내부적으로는 기존 문서를 삭제 마킹(.del) → 새 문서를 새 Segment에 추가.

**`_create` 엔드포인트**: `PUT /index/_create/{id}`로 호출하면 이미 존재하는 문서를 실수로 덮어쓰는 것을 방지한다 (409 Conflict 반환).

### GET by ID vs Search — 두 가지 조회 방식의 근본적 차이

| | GET by ID | Search |
|--|-----------|--------|
| 경로 | `/_doc/{id}` | `/_search` |
| 동작 | Version Map → Translog 또는 Segment에서 _id 기반 직접 조회 | 역색인 탐색 + 스코어링 |
| 실시간성 | 즉시 (Refresh 불필요) | Near Real-Time (Refresh 후) |
| 내부 구조 | _id를 키로 한 O(1) lookup | 역색인(단어→문서 목록) 탐색 |

**GET by ID가 즉시 되는 이유**: ES는 내부적으로 Version Map(메모리)을 유지한다. 최근 색인된 문서가 아직 Segment에 없으면 Version Map이 이를 감지하고, Translog에서 해당 문서의 _source를 읽어 반환한다. 역색인을 거치지 않으므로 Refresh를 기다릴 필요가 없다.

**Search가 Refresh를 기다려야 하는 이유**: Search는 역색인을 탐색해야 한다. 역색인은 Segment 안에만 존재한다. Translog는 순차적인 문서 원본 목록일 뿐 "단어 → 문서 목록" 매핑이 아니므로, Translog에서 Search를 하려면 전체 문서를 스캔해야 한다(O(n), 비현실적). 역색인은 불변(Immutable) Segment 구조이기 때문에 "기존 역색인에 한 줄 추가"가 불가능하고, 새 Segment를 통째로 만들어야 한다. 이걸 문서 1건마다 하면 성능이 붕괴하므로 Buffer에 모아뒀다가 1초마다 한 번에 Segment를 만드는 것(Refresh)이다.

### Update — 부분 업데이트의 내부 동작

`_update` API는 기존 문서의 일부 필드만 변경하는 것처럼 보이지만, 내부적으로는 "기존 _source 읽기 → 필드 병합 → 전체 문서 재색인"을 수행한다.

**PUT vs _update 내부 동작 차이**:
- PUT: 클라이언트가 전체 문서를 보내므로 ES가 기존 문서를 읽을 필요 없음. 바로 삭제 마킹 + 새 문서 색인.
- _update: ES가 기존 _source를 읽어서 병합해야 하므로 읽기 단계가 추가됨. 약간 더 느림.
- _update의 noop 최적화: 변경사항이 없으면 `result: "noop"`으로 재색인을 생략. PUT은 이 최적화가 없다.

**`doc_as_upsert: true`**: 문서가 없으면 생성, 있으면 업데이트하는 패턴.

### Delete — 문서 삭제 vs 인덱스 삭제

- 문서 단위 삭제(`DELETE /_doc/{id}`): .del 파일에 마킹만. 실제 디스크 회수는 Segment Merge 시점에 일어남.
- 인덱스 삭제(`DELETE /index`): 모든 Segment 파일을 즉시 삭제. 디스크 바로 회수.

시계열 로그에서 "30일 지난 로그 삭제"를 날짜별 인덱스 삭제로 처리하는 이유가 바로 이것이다.

### Bulk API — 대량 작업의 핵심

**왜 필요한가**: 문서를 1건씩 색인하면 매번 HTTP 연결 → 라우팅 → 응답의 오버헤드가 발생한다. Bulk API는 여러 작업을 하나의 HTTP 요청으로 묶어서 보낸다.

**형식 규칙**:
- 각 줄은 `\n`으로 끝나야 함 (마지막 줄 포함)
- JSON을 pretty-print하면 안 됨 (한 줄에 하나의 JSON)
- 액션 라인 + 문서 라인이 한 쌍 (delete만 예외: 액션 라인만)

**4가지 액션**:
- `index`: 색인 (있으면 덮어쓰기, 없으면 생성) — 2줄
- `create`: 생성 (이미 있으면 409 에러) — 2줄
- `update`: 부분 업데이트 (`{"doc": {...}}` 형태) — 2줄
- `delete`: 삭제 — 1줄 (문서 라인 없음)

**부분 실패 가능**: Bulk 요청은 "전체 성공 or 전체 실패"가 아니다. 개별 작업이 독립적으로 성공/실패한다. 응답의 `errors: true` 필드를 확인하고, items 배열에서 실패한 항목만 재시도해야 한다.

**적절한 배치 크기**: 5~15MB 또는 1000~5000 문서. 저렴한 티어(t3.small)에서는 보수적으로 1000문서 / 5MB 이하 권장.

**검색은 Bulk로 못 한다**: Bulk는 쓰기 작업(index, create, update, delete)만 지원. 여러 검색을 한 번에 하려면 `_msearch` API를 사용.

### 검색 쿼리의 핵심 — bool 쿼리와 must vs filter

**bool 쿼리의 4가지 절**:

| 절 | 논리 | 점수 영향 | 캐시 | 용도 |
|----|------|---------|------|------|
| `must` | AND | ✓ 반영 | ✗ | 관련도 순위가 필요한 조건 |
| `filter` | AND | ✗ 무시 | ✓ | Yes/No 판단만 필요한 조건 |
| `should` | OR | ✓ 반영 | ✗ | 매칭 시 점수 가산 (선택적) |
| `must_not` | NOT | ✗ 무시 | ✓ | 제외 조건 |

**must vs filter — 가장 중요한 구분**:
- `must`: "이 조건에 얼마나 잘 맞는가?"를 계산 (relevance scoring). TF, IDF, field length 등을 고려하므로 비용이 든다.
- `filter`: "이 조건에 맞는가? Yes/No"만 판단. 점수 계산 없음 + 결과를 비트셋으로 캐시 가능 → 같은 필터 반복 시 즉시 반환.

**실전 판단 기준**: "이 조건이 검색 결과의 순서(랭킹)에 영향을 줘야 하는가?" → Yes면 must, No면 filter.

**점수(score)의 의미**: 검색 결과의 순서를 결정하는 관련도 수치. sort를 지정하지 않으면 `_score` 내림차순이 기본 정렬. match 쿼리에서 검색어 토큰이 많이 매칭될수록, 해당 단어가 전체 문서에서 드물수록, 필드가 짧을수록 점수가 높다.

**term/range 같은 정확한 매칭 쿼리에서의 점수**: status가 "open"인 문서는 전부 동일한 점수를 받는다 (매칭이거나 아니거나, 중간이 없음). 이런 경우 must를 쓰면 불필요한 점수 계산만 하는 셈이므로 filter가 적합하다.

**should의 동작 — 상황에 따라 달라진다**:
- must/filter가 없을 때: should가 OR 조건 (최소 1개 매칭 필수)
- must/filter가 있을 때: should는 선택적 (매칭 시 점수 가산만, 안 매칭돼도 결과에 포함)

**OR 조건을 필수로 만들려면**: should를 별도의 bool 안에 넣고, 그 bool을 filter에 넣는다.
```json
"filter": [
  { "bool": { "should": [조건1, 조건2] } }
]
```

### match vs term — 필드 타입에 맞는 쿼리 선택

- `match`: 검색어를 Analyzer로 분석(토큰화)한 뒤 역색인에서 찾음. text 필드용.
- `term`: 검색어를 분석하지 않고 그대로 역색인에서 찾음. keyword/numeric/date/boolean 필드용.
- `terms` (복수): 여러 값 중 하나라도 매칭. SQL의 IN절. 같은 필드에 대한 OR이면 terms가 간결하고 최적화됨.

**흔한 실수**: text 필드에 term 쿼리 → 매칭 안 됨 (역색인에는 소문자화된 토큰만 있는데 원본 문자열로 찾으려 하므로).

### sort와 점수 계산의 관계

- sort를 지정해도 must/should는 여전히 점수를 계산한다 (정렬에 사용되지 않을 뿐).
- filter만 사용하면 점수 계산 자체가 안 일어난다 (`_score: null`).
- 로그 검색의 최적 패턴: `bool.filter` + `sort: [@timestamp desc]` → 점수 계산 없음, 캐시 활용, 가장 효율적.

### 검색 결과의 total.value와 정확한 카운트

- 매칭 문서 < 10,000건: `total.value`는 정확한 값, `relation: "eq"`
- 매칭 문서 ≥ 10,000건: `total.value: 10000`, `relation: "gte"` (정확하지 않음)
- 정확한 카운트가 필요하면: `track_total_hits: true` 또는 `_count` API
- 타입별 개수 같은 집계는 Aggregation(4단계)이 더 적합


### ES를 단일 Source of Truth로 쓰면 위험한 이유

1. **트랜잭션 없음**: 문서 단위로만 원자적(한 문서의 translog, source, 역색인 등은 모두 원자적으로 반영됨). 여러 문서에 걸친 ACID 트랜잭션 불가. A 성공 B 실패 시 롤백 방법 없음.
2. **동시 업데이트 시 last-write-wins**: 두 클라이언트가 같은 문서를 동시에 업데이트하면 나중에 쓴 쪽이 이김. Optimistic concurrency control로 감지는 가능하지만 자동 보호는 안 됨.
3. **참조 무결성 없음**: FOREIGN KEY 같은 관계형 무결성 검증 불가. 아무 값이나 넣을 수 있음.
4. **내구성 보장 수준**: 기본 설정(`request` durability)이면 RDB와 비슷하지만, 성능을 위해 `async`로 바꾸면 크래시 시 유실 가능.

**일반적인 아키텍처 패턴**: RDB를 Source of Truth로, ES는 검색/집계 전용 복제본으로 사용.

**현재 프로젝트 적용**:
- 로그: ES가 Source of Truth여도 괜찮음 (append-only, 유실 허용 가능)
- Issue History: 이벤트 소싱 방식이면 비교적 안전 (수정이 없으므로 동시성/트랜잭션 문제 해당 없음), 단일 문서 업데이트 방식이면 RDB를 원본으로 두는 게 안전

**이벤트 소싱이 ES에서 비교적 안전한 이유**: ES의 위험 요소 대부분은 "기존 문서를 수정할 때" 발생한다 (동시성 문제, race condition, 부분 업데이트 유실). 이벤트 소싱은 수정이 없고 새 문서를 추가만 하므로 이 위험들이 해당되지 않는다.

---

## 현재 프로젝트 연결

### 로그 검색 패턴
```
bool.filter + sort by @timestamp desc
→ 점수 불필요, 시간순 정렬, 캐시 활용, 가장 효율적
```

### Issue History 목록 조회 패턴
```
bool.filter (status, priority, date range) + sort by created_at desc
→ 대시보드에서 조건별 이슈 목록 표시
```

### Issue History 키워드 검색 패턴
```
bool.must (match on title/description) + bool.filter (status, environment)
→ 제목/설명에서 키워드 검색 + 조건 필터링
```

### API 선택 기준

| 상황 | API | 이유 |
|------|-----|------|
| 로그 실시간 수집 | Bulk + POST (ID 자동) | 대량, 개별 조회 불필요 |
| Issue 생성 | PUT /_doc/{id} 또는 _create | ID 지정, 중복 방지 |
| Issue 상태 변경 | POST /_update/{id} | 부분 업데이트 |
| 30일 지난 로그 삭제 | DELETE /{index} | 인덱스 통째 삭제 (즉시 디스크 회수) |
| 특정 Issue 조회 | GET /_doc/{id} | 즉시 조회 |
| Issue 검색 | GET /_search | 조건 기반 검색 |

---

## 실전 쿼리/설정 예시

### 로그 검색 — 전형적 패턴
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

### 로그 검색 — 메시지 키워드 포함 (점수 불필요, 성능 중요)
```json
GET /logs-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-7d/d" } } },
        { "term": { "environment": "production" } },
        { "term": { "level": "ERROR" } },
        { "match": { "message": "timeout" } }
      ]
    }
  },
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "size": 20
}
```
참고: match를 filter 안에 넣으면 점수 계산 없이 "이 토큰이 포함되어 있는가?"만 판단한다.

### Issue History — 복합 조건 필터링
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

### Issue History — 키워드 검색 + 우선순위 부스팅
```json
GET /issue-history/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "서버 응답 지연" } }
      ],
      "filter": [
        { "term": { "environment": "production" } }
      ],
      "should": [
        { "term": { "priority": "critical" } },
        { "term": { "priority": "high" } }
      ]
    }
  }
}
```

### OR 조건을 필수로 만드는 패턴
```json
// "status가 open이면서, priority가 high이거나 critical인 이슈"
GET /issue-history/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "open" } },
        { "terms": { "priority": ["high", "critical"] } }
      ]
    }
  }
}
// 또는 bool 중첩 방식:
{
  "bool": {
    "filter": [
      { "term": { "status": "open" } },
      { "bool": {
          "should": [
            { "term": { "priority": "high" } },
            { "term": { "priority": "critical" } }
          ]
        }
      }
    ]
  }
}
```

### Bulk API 예시
```json
POST /_bulk
{"index": {"_index": "logs-2025-07-15"}}
{"@timestamp": "2025-07-15T09:00:00Z", "level": "ERROR", "message": "Connection timeout"}
{"index": {"_index": "logs-2025-07-15"}}
{"@timestamp": "2025-07-15T09:01:00Z", "level": "INFO", "message": "Retry successful"}
{"create": {"_index": "issue-history", "_id": "ISS-002"}}
{"issue_id": "ISS-002", "title": "DB 커넥션 풀 고갈", "status": "open"}
{"update": {"_index": "issue-history", "_id": "ISS-001"}}
{"doc": {"status": "resolved"}, "doc_as_upsert": true}
{"delete": {"_index": "issue-history", "_id": "ISS-003"}}
```

---

## 핵심 명령어 / API

```json
// 색인 (ID 자동)
POST /issue-history/_doc
{ "issue_id": "ISS-001", "title": "서버 응답 지연", "status": "open" }

// 색인 (ID 지정)
PUT /issue-history/_doc/ISS-001
{ ... }

// 중복 방지 색인
PUT /issue-history/_create/ISS-001
{ ... }

// ID로 조회
GET /issue-history/_doc/ISS-001

// 부분 업데이트
POST /issue-history/_update/ISS-001
{ "doc": { "status": "resolved" } }

// 삭제
DELETE /issue-history/_doc/ISS-001
DELETE /logs-2025-07-01

// 검색
GET /issue-history/_search
{ "query": { ... }, "sort": [...], "size": 10, "_source": [...] }

// 정확한 카운트
GET /issue-history/_count
{ "query": { "term": { "status": "open" } } }

// 여러 검색 한 번에
POST /_msearch
{"index": "logs-*"}
{"query": {"term": {"level": "ERROR"}}}
{"index": "issue-history"}
{"query": {"term": {"status": "open"}}}
```

---

## 헷갈렸던 점 & 해결

### 1. sort가 있으면 점수 계산이 안 된다?
- 착각: sort를 지정하면 자동으로 점수 계산이 생략된다
- 실제: sort가 있어도 must/should는 여전히 점수를 계산한다. 다만 정렬에 사용되지 않을 뿐.
- 이유: 점수 계산 생략은 `filter`를 써야 일어난다. sort는 정렬 기준만 바꾸는 것이지 쿼리 실행 방식을 바꾸지 않는다.

### 2. must_not의 부정 조건을 filter에 넣으면 같은 거 아닌가?
- 착각: filter에 "부정 조건"을 넣으면 must_not과 동일하다
- 실제: filter는 "포함 조건"이고 must_not은 "제외 조건"이다. ES에는 "부정 조건을 filter에 넣는 문법"이 없다.
- 결론: 둘 다 점수에 영향 안 주고 캐시된다는 공통점이 있지만, 역할이 다르다.

### 3. must에서 점수가 왜 의미 있는가? (어차피 매칭되는 것들은 전부 100점 아닌가?)
- 착각: must 조건을 충족하는 문서는 모두 동일한 점수를 받는다
- 실제: match 쿼리에서는 매칭 정도가 다양하다. "database connection timeout"으로 검색 시 3개 토큰 모두 매칭되는 문서 > 1개만 매칭되는 문서.
- 보완: term/range 같은 정확한 매칭 쿼리에서는 맞다 (매칭이거나 아니거나). 이런 경우 filter가 적합.

### 4. Q6 체크 세션에서 쿼리 조건 누락
- 착각: range 조건 하나만 쓰면 된다
- 실제: 요구사항의 모든 조건을 빠짐없이 나열해야 한다 (environment, level, message, timestamp)
- 교훈: 실전 쿼리 작성 시 "조건을 빠짐없이 나열 → 점수 불필요하면 전부 filter → sort와 size 명시"

### 5. filter의 캐시 언급 누락
- 착각: filter와 must의 차이는 점수 계산 유무뿐
- 실제: filter는 결과를 비트셋으로 캐시한다. 같은 filter 조건이 반복되면 캐시에서 즉시 반환. must는 캐시하지 않는다.

---

## 복습 포인트

### 색인의 본질
색인(Indexing)은 단순 저장이 아니라 "역색인 생성까지 포함하는 과정"이다. 문서가 Translog + Buffer에 기록되고, Refresh 시 Segment로 변환되면서 _source, 역색인, doc_values가 모두 한꺼번에 생성된다. Segment 생성은 원자적(all or nothing)이므로 "_source만 성공하고 역색인 실패" 같은 부분 실패는 발생하지 않는다.

### GET by ID vs Search의 근본적 차이
GET by ID는 Version Map + Translog에서 _id 기반 O(1) lookup이므로 Refresh 없이 즉시 가능하다. Search는 역색인을 탐색해야 하는데, 역색인은 Segment 안에만 존재하고, Translog는 순차적 문서 목록일 뿐 "단어→문서" 매핑이 아니므로 Search에 사용할 수 없다. 역색인이 불변 Segment 구조라서 in-place update가 불가능한 것이 NRT의 근본 원인이다.

### must vs filter 판단법
"이 조건이 결과의 순서(랭킹)에 영향을 줘야 하는가?" → Yes면 must, No면 filter. 로그 검색에서 날짜/레벨/서비스명은 전부 filter. Issue 제목 검색처럼 "가장 관련 있는 것을 먼저"가 필요할 때만 must. filter는 점수 계산 생략 + 비트셋 캐시로 성능 이점이 크다.

### Bulk API의 핵심 특성
액션 라인 + 문서 라인 쌍으로 구성 (delete만 1줄). 부분 실패 가능하므로 `errors` 필드 확인 필수. 검색은 지원하지 않음 (검색 대량 처리는 `_msearch`). 저렴한 티어에서는 1000문서/5MB 이하 권장.

### 인덱스 삭제 vs 문서 삭제
문서 단위 삭제는 .del 마킹만 하고 Segment Merge 시점에 물리 삭제된다 (디스크 즉시 회수 안 됨). 인덱스 삭제는 모든 Segment 파일을 즉시 삭제하므로 디스크가 바로 회수된다. 시계열 로그는 날짜별 인덱스로 분리하고 인덱스 통째 삭제하는 것이 정답.

### ES를 Source of Truth로 쓸 때의 위험
트랜잭션 없음 + 동시 업데이트 시 last-write-wins + 참조 무결성 없음. 로그(append-only)는 ES 단독 가능, Issue History(업데이트 필요)는 RDB를 원본으로 두는 게 안전. 이벤트 소싱 방식이면 수정이 없으므로 ES 단독도 비교적 안전하지만, 정말 중요한 데이터라면 Kafka/RDB에 이벤트 원본을 보관하는 게 가장 안전.

### terms vs should + 여러 term
같은 필드에 대한 OR 조건이면 `terms`가 간결하고 내부적으로 더 최적화됨. 다른 필드에 걸친 OR이면 `should` + 여러 `term`을 써야 한다.
