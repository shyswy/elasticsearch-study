# 1단계: Elasticsearch 핵심 개념 & 아키텍처

## 학습 일자
2026-05-06

## 핵심 개념 정리

### 1. 왜 Elasticsearch인가 — RDB의 한계에서 출발

**문제 정의**: RDB에 로그를 저장하고 `SELECT * FROM logs WHERE message LIKE '%connection timeout%'`를 실행하면?
- B+Tree 인덱스는 값의 prefix 기준으로 정렬되어 있음
- `LIKE 'keyword%'`는 B+Tree leaf에서 시작점 찾고 순차 탐색 가능 → O(log n)
- `LIKE '%keyword%'`는 앞에 와일드카드 → B+Tree 무력화 → 전체 행 풀스캔 O(n)
- 로그 1억 건이면 매 검색마다 1억 행 스캔 → 현실적으로 불가능

**ES를 쓰는 3가지 핵심 이유**:

1. **역색인(Inverted Index) 기반 풀텍스트 검색**
   - "단어 → 문서 ID 리스트"를 미리 만들어두는 HashMap 유사 구조
   - 문서가 1억 건이든 10억 건이든 검색 시간 거의 동일 (O(1)에 가까움)

2. **샤드 기반 분산 처리**
   - 인덱스를 여러 Shard로 분할 → 여러 노드에 분산 저장
   - 검색 시 모든 샤드에 병렬 쿼리, 쓰기도 서로 다른 샤드끼리 병렬
   - 노드 장애 시 Replica가 승격 → failover

3. **내장 Aggregation 엔진**
   - SQL의 GROUP BY + 집계 함수에 해당
   - "최근 1시간 에러 건수", "서비스별 평균 응답시간" 등을 별도 ETL 없이 실시간 처리
   - (상세 동작은 4단계에서 학습)

---

### 2. Lucene과 Elasticsearch의 관계

**Lucene이란**: Java로 작성된 오픈소스 검색 라이브러리. 단일 머신에서 역색인 생성, 텍스트 분석, 검색을 수행한다.

**관계**:
```
Lucene (검색 엔진 라이브러리)
  → 단일 머신에서 역색인 생성/검색 수행
  → 분산 처리, REST API, 클러스터 관리 기능 없음

Elasticsearch / OpenSearch (분산 검색 시스템)
  → Lucene을 내부 엔진으로 사용
  → 그 위에 분산 처리, REST API, 클러스터 관리, 집계 등을 얹은 것
```

비유: Lucene = 자동차 엔진, Elasticsearch = 엔진 + 차체 + 네비 + 자율주행이 합쳐진 완성차

**핵심**: Shard 1개 = Lucene Index 1개. ES가 인덱스를 3개 샤드로 나누면, 내부적으로 3개의 독립적인 Lucene 인스턴스가 각자 역색인을 관리하고 로컬 검색을 수행한다.

---

### 3. 역색인 (Inverted Index) — ES의 핵심 자료구조

**정의**: "단어(term)" → "그 단어가 포함된 문서 ID 리스트(posting list)"를 매핑하는 자료구조.

**구조 (HashMap과 유사)**:
```
색인 시점에 한 번 계산:
  Doc 1: "서버 응답 지연 발생"
  Doc 2: "서버 재시작 완료"
  Doc 3: "응답 시간 정상화"

역색인:
  "서버"   → [Doc 1, Doc 2]
  "응답"   → [Doc 1, Doc 3]
  "지연"   → [Doc 1]
  "재시작" → [Doc 2]
  "정상화" → [Doc 3]

검색 "서버 응답":
  → "서버" 목록 ∩ "응답" 목록 = [Doc 1]  ← 밀리초 단위
```

**vs RDB B+Tree**:

| 상황 | RDB (B+Tree) | ES (역색인) |
|------|-------------|------------|
| `WHERE id = 123` | O(log n) — 빠름 | 불필요 (이건 RDB가 적합) |
| `WHERE message LIKE '%timeout%'` | O(n) — 풀스캔 | O(1) — key lookup |
| `WHERE message LIKE 'error%'` | O(log n) — prefix 가능 | O(1) |
| 1억 건에서 텍스트 검색 | 수십 초~분 | 수 밀리초 |

B+Tree가 `%keyword%`에 무력한 이유: B+Tree는 값의 prefix 기준으로 정렬. leaf 노드에서 시작점을 찾아 next로 이동하는 구조이므로, "중간에 있는 단어"를 찾으려면 전체를 봐야 함. 역색인은 단어를 key로 바로 lookup하므로 단어가 문장 어디에 있든 즉시 조회.

---

### 4. 핵심 용어 — 정확한 정의

| 용어 | 정의 | 비고 |
|------|------|------|
| Cluster | 하나 이상의 노드로 구성된, 동일한 클러스터 이름을 공유하는 전체 시스템 | AWS에서 "도메인" 생성 = 클러스터 1개 |
| Node | 클러스터에 참여하는 단일 ES 인스턴스 (= 하나의 JVM 프로세스) | Master Node(클러스터 관리), Data Node(데이터 저장+검색) 역할 구분 |
| Index | 관련 문서들의 **논리적** 모음. 동일한 매핑을 공유하는 문서 컬렉션 | 물리적으로는 Shard들로 분산됨. RDB의 "테이블"에 가까움 |
| Document | 인덱스에 저장되는 하나의 JSON 객체. 검색의 최소 단위 | `_source`가 실제 데이터, `_id`가 고유 식별자 |
| Shard | 인덱스를 물리적으로 분할한 독립적인 Lucene Index 단위 | 1 Shard = 1 Lucene Index. 샤드당 수십MB JVM 힙 오버헤드 |
| Replica | Primary Shard의 완전한 복제본 | 읽기 분산 + 장애 시 승격. 반드시 Primary와 다른 노드에 배치 |

**"색인(Indexing)"이라는 용어**:
- 명사 "Index": 문서들의 논리적 모음
- 동사 "색인하다(Indexing)": 문서를 저장 + 역색인 생성하는 행위 (RDB의 INSERT에 해당하지만, 역색인 생성이 포함되므로 "색인"이라 부름)

---

### 5. Primary Shard vs Replica Shard — 상세

**정의**:
- **Primary Shard**: 원본 데이터를 가진 샤드. 모든 쓰기(색인/수정/삭제)는 반드시 Primary에서 먼저 수행
- **Replica Shard**: Primary의 완전한 복제본. Primary에 쓰기 완료 후 동기적으로 복제됨

**쓰기 흐름**:
```
클라이언트 → 색인 요청
      ↓
Coordinating Node: hash(doc_id) % number_of_shards → 대상 Primary 결정
      ↓
Primary Shard: Translog 기록 + Buffer 추가
      ↓
Replica Shard: 같은 작업 수행 (동기 복제)
      ↓
클라이언트에 "성공" 응답
```

**쓰기 병렬성**:
- 서로 다른 Primary Shard끼리는 **완전 병렬** (각각 다른 노드에서 동시 처리)
- 같은 Shard 내에서는 Primary가 순서 보장
- 예: Primary 3개, 문서 A→Shard0, B→Shard1, C→Shard2 → 3개 노드에서 동시 쓰기

**읽기 분산**:
- 검색 요청은 Primary든 Replica든 아무 데서나 처리 가능
- Replica가 3개면 읽기를 4곳(Primary 1 + Replica 3)에 분산

**설정**:
```json
{
  "number_of_shards": 3,    // Primary 3개 (생성 후 변경 불가!)
  "number_of_replicas": 2   // 각 Primary마다 Replica 2개 (언제든 변경 가능)
}
// 총 샤드 수 = 3 × (1 + 2) = 9개
```

**Primary-Replica ≠ Master-Slave 노드** (중요한 차이):
- Master-Slave는 프로세스(노드) 단위. 서버 1대 = Master, 서버 1대 = Slave
- Primary-Replica는 **샤드 단위**. 한 노드 안에 여러 샤드가 있고, 어떤 건 Primary이고 어떤 건 Replica
- 한 노드가 Shard 0의 Primary이면서 동시에 Shard 1의 Replica일 수 있음
- Master Node(클러스터 메타데이터 관리)와 Primary Shard(데이터 쓰기 담당)는 완전히 별개 개념
- ES는 기본이 **동기 복제** → 전통적 Master-Slave(비동기, replication lag)보다 데이터 유실 위험 낮음

---

### 6. Failover — Replica와 장애 내성

**핵심 원리**: Primary가 있는 노드가 죽어도 데이터를 잃지 않고 서비스를 계속할 수 있게 하는 것.

**규칙**: Primary와 그 Replica는 반드시 다른 노드에 배치 (같은 노드에 두면 노드 죽을 때 둘 다 사라짐)

**Replica 수에 따른 장애 내성**:

| Replicas | 동시 노드 장애 허용 | 최소 필요 노드 수 | 상황 |
|----------|-------------------|-----------------|------|
| 0 | 0개 (하나라도 죽으면 데이터 유실) | 1 | 단일 노드, 비용 최소 |
| 1 | 1개 | 2 | 가장 현실적인 HA |
| 2 | 2개 | 3 | 높은 가용성 |

**자동 승격 과정**:
```
1. Master Node가 "Node A 응답 없음" 감지 (ping timeout)
2. Node A에 있던 Primary Shard들을 "미할당" 상태로 변경
3. 다른 노드에 있는 해당 Replica를 새 Primary로 승격 (즉시)
4. 클러스터 상태: GREEN → YELLOW (Replica 부족)
5. 남은 노드들에 새 Replica 생성 시작 (데이터 복제)
6. 복제 완료 → 다시 GREEN
```
전체 과정 자동, 수 초~수십 초 내 완료. 애플리케이션은 잠깐의 지연만 겪고 계속 동작.

**노드 수를 넘는 샤드/Replica 설정 시**:

| 상황 | 가능 여부 | 의미 |
|------|---------|------|
| Primary > 노드 수 | ✓ 동작함 | 한 노드에 Primary 여러 개. 쓰기 병렬성은 노드 수에 제한되지만, 데이터 크기/미래 확장 이유로 합리적일 수 있음 |
| Replica > 노드 수 - 1 | 배치 불가 (unassigned) | 클러스터 YELLOW. 리소스 낭비는 아니지만 실제 문제와 구분 어려워짐 |

---

### 7. RDBMS vs ES — 데이터 모델링 철학의 차이

**RDBMS**: "데이터를 정규화하고, 필요할 때 JOIN으로 조합한다"
- 저장 공간 효율적, 데이터 일관성 보장
- 하지만 JOIN은 비용이 큼

**Elasticsearch**: "검색/집계할 형태로 미리 비정규화해서 저장한다"
- 저장 공간은 더 쓰지만, 검색 시 JOIN 불필요 → 빠름
- ES에는 JOIN이 없음 (nested, parent-child는 있지만 제한적)

**실제 예시 — Issue History**:
```
RDB: issues 테이블 + users 테이블 + teams 테이블 → 3개 JOIN

ES: 하나의 문서에 모두 포함
{
  "issue_id": "ISS-001",
  "title": "서버 응답 지연",
  "assignee": "홍길동",      // 이미 이름으로 저장
  "team": "backend",         // 이미 팀명으로 저장
  "tags": ["performance", "backend"]
}
```

**비정규화의 단점**: 담당자 이름이 바뀌면? RDB는 users 테이블만 수정. ES는 관련 문서 전부 업데이트 필요.

**왜 ES에서 비정규화를 선택하는가**:
1. ES에 JOIN이 없으므로 사실상 유일한 선택지
2. 로그/이슈 히스토리는 "발생 시점의 스냅샷" → 나중에 이름이 바뀌어도 당시 기록은 그대로 유지하는 게 맞음
3. 검색 성능이 최우선 목적

---

### 8. 문서의 생명주기 — 색인부터 검색 가능까지

**전체 흐름**:
```
1. 색인 요청 도착 → Coordinating Node 수신
2. 라우팅: hash(doc_id) % number_of_shards → 대상 Primary Shard 결정
3. Primary Shard에서:
   a. Translog에 기록 (디스크, 장애 복구용) ← 반드시 먼저!
   b. In-Memory Buffer에 추가
4. Replica Shard에 동기 복제 (같은 과정)
5. 클라이언트에 "성공" 응답
   ─── 여기까지가 "쓰기 완료" (검색은 아직 불가) ───
6. [Refresh] (기본 1초마다)
   Buffer에 쌓인 문서들 → 새 Lucene Segment 생성 (파일 시스템 캐시)
   → 이 시점부터 검색 가능!
7. [Flush] (주기적)
   Segment를 디스크에 fsync + Translog 비움 → 완전한 영속화
```

**"쓰기 완료" = Segment 생성이 아님!**
- 쓰기 완료 = Translog 기록 + Buffer 추가까지
- Segment 생성(Refresh)을 기다렸다가 응답하면 매 요청마다 최대 1초 대기 → 너무 느림
- Translog에 기록되면 장애 시에도 복구 가능하므로, 그 시점에서 "안전하게 저장됨"으로 간주

**Translog가 Buffer보다 먼저인 이유**:
- Buffer는 메모리 → 노드 죽으면 날아감
- Translog는 디스크 → 노드 죽어도 복구 가능 (RDB의 WAL과 동일한 역할)
- 먼저 Translog에 써서 내구성 확보 → 그 다음 Buffer에 추가

---

### 9. Refresh — 왜 바로 저장 안 하고 버퍼를 쓰는가

**문제**: 문서가 들어올 때마다 즉시 디스크에 쓰고 역색인을 업데이트하면?
- 디스크 I/O는 메모리 대비 수천~수만 배 느림
- 초당 수천 건 로그 → 매 건마다 디스크 쓰기 → 병목
- 역색인 파일을 매번 수정하면 파일 구조 깨지기 쉬움

**해결: Segment라는 불변(Immutable) 파일**

Lucene의 핵심 설계: 한번 쓴 Segment 파일은 절대 수정하지 않는다.
- 수정이 없으면 동시 읽기가 안전 (락 불필요)
- 파일 시스템 캐시를 최대한 활용 가능
- 손상 위험 없음
- 새 데이터는 항상 새 Segment로 생성

**Refresh의 실제 동작**:
```
[0.0초] 문서 A 도착 → Buffer: [A]         검색 불가
[0.3초] 문서 B 도착 → Buffer: [A, B]      검색 불가
[0.7초] 문서 C 도착 → Buffer: [A, B, C]   검색 불가
[1.0초] ★ Refresh ★
         Buffer [A, B, C] → 새 Segment 1 생성
         Buffer 비워짐
         → A, B, C 검색 가능!
[1.2초] 문서 D 도착 → Buffer: [D]         D는 아직 검색 불가
[2.0초] ★ Refresh ★
         Buffer [D] → 새 Segment 2 생성
         → D도 검색 가능!
```

**왜 이렇게 하면 좋은가**:
1. 쓰기 성능: 매 문서마다 디스크 쓰기 대신, 1초치를 모아서 한 번에 Segment 생성 → 효율적
2. 검색 성능: Segment는 불변이라 락 없이 병렬 읽기 가능
3. 거의 실시간: 최대 1초 지연이면 대부분의 모니터링에 충분

**Refresh vs Flush 핵심 구분**:
- **Refresh**: Buffer → Segment (파일 시스템 캐시에 존재). 검색 가능해지는 시점. 아직 디스크에 fsync 안 됨.
- **Flush**: Segment를 디스크에 fsync + Translog 정리. 완전한 영속화 시점.
- Refresh 후 Flush 전에 노드가 죽어도 Translog가 있으므로 데이터 복구 가능.

**트레이드오프**:
- Refresh 자주 (1초) → 작은 Segment 많이 생김 → 나중에 Merge 필요 → 검색 시 여러 Segment 탐색
- Refresh 덜 자주 (30초) → Segment 수 줄어듦 + 색인 성능 향상, 하지만 검색 반영 느려짐

---

### 10. Near Real-Time (NRT)

**정의**: 문서 색인 후 즉시 검색 가능한 게 아니라, Refresh(기본 1초) 후에 검색 가능해지는 특성.

"Near" Real-Time인 이유: 최대 1초의 지연이 존재. 완전한 실시간(0ms)은 아니지만, 대부분의 로그/이슈 모니터링에는 충분.

**refresh_interval 튜닝**:
- 기본값: 1초
- 대량 색인 시: 30초 또는 -1(수동)로 설정 → 색인 부하 크게 감소
- 실시간 대시보드가 아니라 "최근 몇 분 내 로그"를 보는 용도라면 30초 지연은 문제없음

---

## 현재 프로젝트 연결

| 개념 | 프로젝트 적용 |
|------|-------------|
| 역색인 | 로그 메시지에서 "timeout", "error" 등 키워드 검색 → 밀리초 단위 |
| 샤드 | 단일 노드(t3.small)이므로 인덱스당 Primary 1개, Replica 0개 |
| 비정규화 | Issue History에 assignee 이름, team명을 문서에 직접 포함 |
| NRT | 로그 발생 후 1초 내 대시보드에서 검색 가능 |
| Aggregation | "최근 1시간 에러 건수", "서비스별 응답시간" 등 실시간 집계 |

**저렴한 티어(t3.small) 제약과 대응**:
- JVM Heap ~1GB → 샤드 수 최소화 필수 (샤드당 수십MB 오버헤드)
- 단일 노드 → Replica 배치 불가 (replicas: 0, 클러스터 YELLOW는 정상)
- Dedicated Master Node 없음 → Data Node가 Master 겸임 → 대량 색인 중 클러스터 관리 느려질 수 있음
- 단일 AZ → 노드 장애 시 서비스 중단 가능 → 자동 스냅샷(매일)으로 최악의 경우 하루치 유실
- **핵심 운영 전략**: 인덱스 설계 최적화 + ISM으로 오래된 인덱스 자동 삭제 + refresh_interval 늘리기

**AWS OpenSearch Service가 관리해주는 것**:
- 노드 프로비저닝, OS 패치, 소프트웨어 업그레이드
- 자동 스냅샷 (매일, 14일 보존)
- 모니터링 (CloudWatch 연동)

**직접 해야 하는 것**:
- 인덱스 설계 (매핑, 샤드 수)
- 데이터 수명주기 관리 (ISM 정책)
- 쿼리 최적화
- 접근 제어 설정
- 용량 계획 (디스크 부족 전에 대응)

---

## 실전 쿼리/설정 예시

```json
// 클러스터 상태 확인
GET /_cluster/health

// 인덱스 생성 (단일 노드 최적 설정)
PUT /logs-2025-07-15
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}

// 문서 색인 (ID 자동 생성)
POST /issue-history/_doc
{
  "issue_id": "ISS-001",
  "title": "서버 응답 지연",
  "status": "open",
  "assignee": "홍길동",
  "team": "backend",
  "created_at": "2025-07-15T09:00:00Z"
}

// 문서 색인 (ID 지정)
PUT /issue-history/_doc/ISS-001
{ ... }

// ID로 단일 문서 조회
GET /issue-history/_doc/ISS-001
```

---

## 헷갈렸던 점 & 해결

1. **Translog vs Buffer 순서**
   - 착각: Buffer가 먼저인 줄 알았음
   - 실제: Translog(디스크)가 먼저. 이유: 메모리만 쓰면 노드 죽을 때 유실 → Translog로 내구성 먼저 확보

2. **Refresh vs Flush 혼동**
   - 착각: 둘 다 "디스크에 쓰는 것"
   - 실제: Refresh = Buffer → Segment (파일 시스템 캐시, 검색 가능해짐). Flush = Segment → 디스크 fsync + Translog 정리 (완전 영속화)
   - Refresh 후 Flush 전에 죽어도 Translog로 복구 가능

3. **Primary-Replica ≠ Master-Slave 노드**
   - 착각: 노드 단위로 Master/Slave가 고정
   - 실제: 샤드 단위. 한 노드가 Shard 0의 Primary이면서 Shard 1의 Replica일 수 있음
   - Master Node(클러스터 관리)와 Primary Shard(데이터 쓰기)는 완전히 별개 개념

4. **Index = 논리적 모음**
   - 착각: Index가 물리적 저장 단위
   - 실제: 물리적으로는 Shard들로 분산. Index는 그걸 하나로 묶는 논리적 단위

5. **"쓰기 완료" 시점**
   - 착각: Segment 생성(Refresh) 완료 = 쓰기 완료
   - 실제: Translog + Buffer 추가 시점에 클라이언트에 성공 응답. Segment 생성은 나중(Refresh 시)

---

## 복습 포인트

1. **ES의 3가지 핵심 가치**: 역색인(O(1) 텍스트 검색, RDB LIKE '%x%'는 O(n) 풀스캔) + 샤드(분산 저장, 병렬 검색/쓰기, failover) + Aggregation(실시간 집계, 별도 ETL 불필요). 이 3가지가 로그/이슈 데이터에 ES를 쓰는 이유.

2. **Lucene = 검색 엔진 라이브러리, ES = 분산 시스템**. Shard 1개 = Lucene Index 1개. ES가 샤드 3개로 나누면 내부적으로 독립적인 Lucene 인스턴스 3개가 각자 역색인 관리. Lucene은 분산/API/클러스터 관리 못 함 → ES가 그걸 얹은 것.

3. **역색인 = HashMap("단어" → [문서 ID 리스트])**. 색인 시점에 텍스트를 토큰화하여 역색인 생성. 검색 시 단어를 key로 lookup → O(1). B+Tree는 prefix 정렬이라 중간 매칭 불가 → 풀스캔. 이게 ES가 텍스트 검색에 압도적으로 빠른 근본 이유.

4. **색인 흐름**: 요청 → Translog(내구성, 디스크) → Buffer(메모리) → 클라이언트 응답(쓰기 완료) → [Refresh 1초] Segment 생성(검색 가능) → [Flush] 디스크 fsync(영속화). Translog가 먼저인 이유: 메모리만 쓰면 노드 죽을 때 유실. Refresh ≠ Flush: Refresh는 검색 가능 시점, Flush는 영속화 시점.

5. **Primary = 쓰기 담당, Replica = 읽기 분산 + failover**. 서로 다른 Primary끼리는 쓰기 완전 병렬. 동기 복제(Primary 쓰기 → Replica 복제 → 응답). Replica 수 = 동시 노드 장애 허용 수. 노드 죽으면 Master가 감지 → Replica를 새 Primary로 자동 승격(수 초). Primary-Replica는 샤드 단위(노드 단위 아님), Master Node와 별개 개념.

6. **Segment는 불변(Immutable)**: 한번 쓰면 수정 안 함 → 락 없이 병렬 읽기 안전, 파일 시스템 캐시 활용, 손상 위험 없음. 삭제 = .del 마킹(실제 삭제는 Merge 시). 업데이트 = 기존 삭제 마킹 + 새 문서 추가. Refresh가 자주 일어나면 작은 Segment 많아짐 → Merge로 합침.

7. **NRT = 최대 1초 지연**. Buffer에 있다가 Refresh로 Segment 되어야 검색 가능. refresh_interval 늘리면(30s) 색인 성능 향상 + Segment 수 감소, 대신 검색 반영 느려짐. 저렴한 티어에서 실시간성 불필요하면 30초 권장.

8. **비정규화**: ES에 JOIN 없음 → 필요한 데이터를 문서에 다 넣음. 단점: 데이터 변경 시 관련 문서 전부 업데이트. 하지만 로그/이슈는 발생 시점 스냅샷이라 변경할 일 거의 없음 → 트레이드오프 수용 가능.

9. **저렴한 티어 핵심 전략**: 샤드 최소화(인덱스당 1개, JVM 힙 절약), replicas:0(단일 노드), refresh_interval 늘리기(색인 부하 감소), ISM으로 오래된 인덱스 자동 삭제(디스크 관리). Primary > 노드 수는 가능하지만 메모리 부담. Replica > 노드수-1은 배치 불가(unassigned, YELLOW).
