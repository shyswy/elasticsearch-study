## D5단계: Alias 전략 & 마이그레이션

> **학습 목표**: Alias를 활용한 무중단 인덱스 전환, 필터 alias,
> write/read alias 분리 패턴을 설계할 수 있다.

### D5-1. Alias의 본질

```
Alias = 하나 이상의 인덱스를 가리키는 포인터 (가상 이름)

특성:
  - 인덱스 이름 변경 불가 → alias로 간접 참조
  - 클라이언트는 alias 이름만 알면 됨
  - 실제 인덱스 교체 시 클라이언트 코드 변경 불필요
  - 하나의 alias가 여러 인덱스를 가리킬 수 있음 (읽기)
  - write alias는 하나의 인덱스만 가리켜야 함
```

### D5-2. Write Alias vs Read Alias 분리 패턴

```json
// 초기 설정
PUT /issue-history-v1
{
  "aliases": {
    "issue-history-write": { "is_write_index": true },
    "issue-history-read": {}
  }
}

// 애플리케이션 코드:
//   쓰기: POST /issue-history-write/_doc
//   읽기: GET /issue-history-read/_search

// 매핑 변경 필요 시 (v1 → v2 마이그레이션):
// Step 1: 새 인덱스 생성
PUT /issue-history-v2 { "mappings": { ... } }

// Step 2: reindex
POST /_reindex
{ "source": { "index": "issue-history-v1" }, "dest": { "index": "issue-history-v2" } }

// Step 3: alias 원자적 전환
POST /_aliases
{
  "actions": [
    { "remove": { "index": "issue-history-v1", "alias": "issue-history-write" } },
    { "remove": { "index": "issue-history-v1", "alias": "issue-history-read" } },
    { "add": { "index": "issue-history-v2", "alias": "issue-history-write", "is_write_index": true } },
    { "add": { "index": "issue-history-v2", "alias": "issue-history-read" } }
  ]
}
// → 원자적(atomic) 실행: 클라이언트는 중단 없이 계속 동작
```

### D5-3. Filter Alias — 뷰(View)처럼 사용

```json
// 팀별로 자기 데이터만 보이는 alias
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "issue-history-v2",
        "alias": "issues-backend",
        "filter": { "term": { "team": "backend" } }
      }
    },
    {
      "add": {
        "index": "issue-history-v2",
        "alias": "issues-frontend",
        "filter": { "term": { "team": "frontend" } }
      }
    }
  ]
}

// 사용:
GET /issues-backend/_search  ← 자동으로 team=backend 필터 적용
GET /issues-frontend/_search ← 자동으로 team=frontend 필터 적용

// 장점: 클라이언트가 필터 조건을 몰라도 됨
// 활용: FGAC 대안, 멀티 테넌시 간이 구현
```

### D5-4. 다중 인덱스 Alias (시계열 읽기)

```json
// 최근 7일 로그만 묶는 alias
POST /_aliases
{
  "actions": [
    { "add": { "index": "logs-2025-07-09", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-10", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-11", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-12", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-13", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-14", "alias": "logs-recent" } },
    { "add": { "index": "logs-2025-07-15", "alias": "logs-recent" } }
  ]
}

// ISM 또는 cron으로 매일 업데이트:
// - 가장 오래된 인덱스 remove
// - 새 인덱스 add

GET /logs-recent/_search  ← 항상 최근 7일 대상
```

### D5-5. 무중단 마이그레이션 시나리오

```
시나리오: 매핑 변경 (text → keyword, 새 필드 추가 등)

방법 1: Reindex + Alias 전환 (소규모 데이터)
  1. 새 인덱스 생성 (새 매핑)
  2. _reindex로 데이터 복사
  3. alias 원자적 전환
  4. 기존 인덱스 삭제
  → 다운타임: 0 (alias 전환은 atomic)
  → 주의: reindex 중 새로 들어오는 데이터 처리 필요

방법 2: Dual-Write + Alias 전환 (대규모/실시간 데이터)
  1. 새 인덱스 생성
  2. 앱에서 dual-write 시작 (v1 + v2 동시 쓰기)
  3. 기존 데이터 reindex (v1 → v2)
  4. read alias를 v2로 전환
  5. write alias를 v2로 전환
  6. dual-write 중단, v1 삭제
  → 다운타임: 0
  → 복잡도: 높음

방법 3: Data Stream의 경우
  - backing index는 read-only이므로 매핑 변경 불필요
  - 새 매핑은 다음 rollover된 backing index부터 적용
  - template만 업데이트하면 됨
  → 과거 데이터 매핑은 그대로 유지됨 (필요 시 reindex)
```

### D5-6. 학습 체크

- [ ] Write Alias와 Read Alias를 분리하는 이유를 설명할 수 있다
- [ ] Filter Alias의 동작 방식과 활용 사례를 설명할 수 있다
- [ ] Alias 기반 무중단 마이그레이션의 단계를 순서대로 설명할 수 있다
- [ ] `_aliases` API의 원자적(atomic) 실행 특성을 설명할 수 있다
- [ ] Data Stream에서 매핑 변경 시 대응 전략을 설명할 수 있다
- [ ] 다중 인덱스 Alias로 시계열 데이터의 조회 범위를 제어할 수 있다

---
