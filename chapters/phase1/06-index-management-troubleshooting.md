## 6단계: 인덱스 관리 & 트러블슈팅

> **학습 목표**: 클러스터 상태 이상 시 원인을 진단하고 해결할 수 있으며,
> 인덱스 운영에 필요한 관리 작업을 수행할 수 있다.

### 6-1. 클러스터 상태 확인

```json
// 클러스터 전체 상태
GET /_cluster/health
// 응답: { "status": "green" | "yellow" | "red", ... }

// 노드 상태
GET /_cat/nodes?v

// 인덱스 목록 + 상태
GET /_cat/indices?v&s=index

// 샤드 할당 상태
GET /_cat/shards?v

// 미할당 샤드 이유 확인
GET /_cluster/allocation/explain
```

### 6-2. 클러스터 상태별 의미

```
GREEN:  모든 Primary + Replica 정상 할당
YELLOW: 모든 Primary 정상, 일부 Replica 미할당
        → 단일 노드 클러스터에서는 정상 (replica 배치할 다른 노드 없음)
RED:    일부 Primary Shard 미할당 → 데이터 유실 위험!
```

### 6-3. 에러별 진단 가이드

**클러스터 RED — Primary Shard 미할당**:
```json
// 원인 확인
GET /_cluster/allocation/explain

// 흔한 원인:
// 1. 디스크 공간 부족 (워터마크 초과)
//    → 오래된 인덱스 삭제 또는 디스크 증설
// 2. 노드 장애
//    → AWS 콘솔에서 노드 상태 확인

// 디스크 워터마크 확인
GET /_cluster/settings?include_defaults=true&filter_path=*.cluster.routing.allocation.disk*
```

**디스크 워터마크**:
```
low  (85%): 새 샤드 할당 중지
high (90%): 샤드를 다른 노드로 이동 시작
flood_stage (95%): 인덱스를 read-only로 전환!

→ read-only 해제:
PUT /logs-*/_settings
{ "index.blocks.read_only_allow_delete": null }
```

**JVM Memory Pressure 높음 (>80%)**:
```
원인:
  - 너무 많은 샤드 (샤드당 메모리 오버헤드)
  - 큰 집계 쿼리 (cardinality, terms 등)
  - fielddata 사용 (text 필드에 집계 시도)

해결:
  1. 불필요한 인덱스/샤드 삭제
  2. 집계 쿼리 최적화 (size 줄이기)
  3. text 필드 집계 → keyword 필드로 변경
```

**검색/색인 지연 증가**:
```
원인:
  - CPU 포화 (동시 요청 과다)
  - 대량 색인 중 검색 경합
  - 복잡한 쿼리 (wildcard, regex 등)

해결:
  1. refresh_interval 늘리기 (색인 부하 감소)
  2. 쿼리 최적화 (filter 활용, 불필요한 필드 제외)
  3. bulk 색인 시 적절한 배치 크기 사용
```

### 6-4. 인덱스 관리 작업

**인덱스 설정 변경**:
```json
// refresh_interval 변경
PUT /logs-*/_settings
{
  "index.refresh_interval": "30s"
}

// replica 수 변경
PUT /issue-history/_settings
{
  "index.number_of_replicas": 1
}
```

**Reindex (인덱스 재생성)**:
```json
// 매핑 변경이 필요할 때 (기존 인덱스 매핑은 변경 불가)
POST /_reindex
{
  "source": { "index": "issue-history" },
  "dest": { "index": "issue-history-v2" }
}

// 이후 alias로 전환
POST /_aliases
{
  "actions": [
    { "remove": { "index": "issue-history", "alias": "issue-current" } },
    { "add": { "index": "issue-history-v2", "alias": "issue-current" } }
  ]
}
```

**Alias (별칭)** — 무중단 인덱스 전환:
```json
// alias 생성
POST /_aliases
{
  "actions": [
    { "add": { "index": "issue-history-v2", "alias": "issue-history-read" } }
  ]
}

// 애플리케이션은 alias로 접근 → 실제 인덱스 변경 시 무중단
GET /issue-history-read/_search
```

**Force Merge (세그먼트 병합)**:
```json
// 더 이상 쓰기가 없는 인덱스에 대해 (읽기 성능 향상)
POST /logs-2025-07-01/_forcemerge?max_num_segments=1
```

### 6-5. 대량 작업 & Task 관리

**_update_by_query — 조건부 대량 업데이트**:
```json
// "7월 1일 이전의 open 이슈를 모두 stale로 변경"
POST /issue-history/_update_by_query
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "open" } },
        { "range": { "created_at": { "lt": "2025-07-01" } } }
      ]
    }
  },
  "script": {
    "source": "ctx._source.status = 'stale'",
    "lang": "painless"
  }
}

// 충돌 시 동작 설정
POST /issue-history/_update_by_query?conflicts=proceed
// → 버전 충돌 무시하고 계속 진행 (기본은 abort)
```

**_delete_by_query — 조건부 대량 삭제**:
```json
// "30일 이상 된 debug 레벨 로그 삭제"
POST /logs-*/_delete_by_query
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "DEBUG" } },
        { "range": { "@timestamp": { "lt": "now-30d" } } }
      ]
    }
  }
}

// 비동기 실행 (대량 작업 시 권장)
POST /logs-*/_delete_by_query?wait_for_completion=false
// → task ID 반환, 백그라운드 실행
```

**Task Management API — 장시간 작업 모니터링**:
```json
// 진행 중인 모든 작업 조회
GET /_tasks?actions=*byquery&detailed=true

// 특정 작업 상태 확인
GET /_tasks/node_id:task_id

// 작업 취소
POST /_tasks/node_id:task_id/_cancel
```

**대량 작업 시 주의사항**:
```
1. wait_for_completion=false로 비동기 실행 (타임아웃 방지)
2. conflicts=proceed로 버전 충돌 무시 (필요 시)
3. scroll_size로 배치 크기 조절 (기본 1000)
4. 저렴한 티어에서는 대량 작업이 클러스터 부하 유발
   → 업무 시간 외 실행 또는 throttle 설정:
   POST /issue-history/_update_by_query?scroll_size=500&requests_per_second=100
```

### 6-6. 트러블슈팅 체크리스트

```
문제 발생 시 순서대로 확인:

Step 1: 클러스터 상태 확인
  GET /_cluster/health

Step 2: 노드 상태 확인
  GET /_cat/nodes?v

Step 3: 디스크 공간 확인
  GET /_cat/allocation?v

Step 4: 미할당 샤드 확인
  GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

Step 5: 느린 쿼리 확인
  GET /_cat/tasks?v  (진행 중인 작업)

Step 6: JVM 상태 확인
  GET /_nodes/stats/jvm
```

### 6-7. 학습 체크

- [ ] `GET /_cluster/health`의 green/yellow/red 상태를 해석할 수 있다
- [ ] 디스크 워터마크로 인한 read-only 상태를 해제할 수 있다
- [ ] JVM Memory Pressure가 높을 때 원인과 해결책을 설명할 수 있다
- [ ] Reindex + Alias로 무중단 매핑 변경을 수행할 수 있다
- [ ] 클러스터 장애 시 트러블슈팅 순서를 따라 진단할 수 있다
- [ ] Force Merge를 언제 사용해야 하는지 설명할 수 있다
- [ ] `_update_by_query`로 조건부 대량 업데이트를 수행할 수 있다
- [ ] `_delete_by_query`를 비동기로 실행하고 Task API로 모니터링할 수 있다
- [ ] 대량 작업 시 `requests_per_second`로 throttle을 설정하는 이유를 설명할 수 있다

---
