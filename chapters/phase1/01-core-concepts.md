## 1단계: Elasticsearch 핵심 개념 & 아키텍처

> **학습 목표**: "왜 Elasticsearch/OpenSearch를 쓰는가"부터 시작하여
> 클러스터의 구성 요소와 데이터가 저장되는 방식을 설명할 수 있다.

### 1-1. 왜 Elasticsearch인가 (개념 이해)

**문제**: RDB에 로그를 저장하면 되지 않나?

| 문제 상황 | Elasticsearch의 해결책 |
|----------|----------------------|
| 대량 로그에서 텍스트 검색이 느리다 | 역색인(Inverted Index) 기반 풀텍스트 검색 |
| 로그가 쌓일수록 DB 쿼리가 느려진다 | 분산 저장 + 병렬 검색 (샤딩) |
| "최근 1시간 에러 건수" 같은 집계가 복잡하다 | 내장 Aggregation 엔진 |
| 스키마 변경이 어렵다 | 유연한 JSON 문서 구조 |
| 실시간 대시보드가 필요하다 | Near Real-Time 검색 (1초 이내 반영) |

**핵심 특성**:
- **문서 지향(Document-Oriented)**: JSON 문서를 저장하고 검색
- **분산 시스템**: 데이터를 여러 노드에 분산 저장
- **Near Real-Time**: 문서 색인 후 ~1초 내 검색 가능
- **스키마리스(Schemaless)**: 매핑 없이도 문서 저장 가능 (하지만 명시적 매핑 권장)

### 1-2. 핵심 용어 정리

```
┌─────────────────────────────────────────────────────────┐
│                    Cluster (클러스터)                     │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │    │
│  │             │  │             │  │             │    │
│  │ [Shard 0-P] │  │ [Shard 1-P] │  │ [Shard 2-P] │    │
│  │ [Shard 1-R] │  │ [Shard 2-R] │  │ [Shard 0-R] │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
│  Index: "logs-2025-07"                                  │
│    → Primary Shards: 3개 (P)                            │
│    → Replica Shards: 3개 (R) — 각 Primary의 복제본      │
└─────────────────────────────────────────────────────────┘
```

| 용어 | RDBMS 비유 | 설명 |
|------|-----------|------|
| Cluster | DB 서버 그룹 | 하나 이상의 노드로 구성된 전체 시스템 |
| Node | DB 서버 1대 | 클러스터의 구성원 (데이터 저장 + 검색 처리) |
| Index | Database (또는 Table) | 관련 문서들의 논리적 모음 |
| Document | Row | JSON 형태의 데이터 단위 |
| Field | Column | 문서 내의 키-값 쌍 |
| Shard | Partition | 인덱스를 물리적으로 분할한 단위 |
| Replica | Slave/복제본 | Primary Shard의 복제본 (HA + 읽기 성능) |

### 1-3. RDBMS vs Elasticsearch 사고방식 전환

```
RDBMS 사고:
  "테이블을 설계하고, 정규화하고, JOIN으로 데이터를 조합한다"

Elasticsearch 사고:
  "검색/집계할 형태로 문서를 비정규화해서 저장한다"
  "JOIN은 없다 — 필요한 데이터는 문서 안에 다 넣는다"
```

**예시: Issue History**

```
RDBMS 방식:
  issues 테이블 + issue_logs 테이블 + users 테이블 → JOIN

Elasticsearch 방식:
  하나의 문서에 필요한 정보를 모두 포함:
  {
    "issue_id": "ISS-001",
    "title": "서버 응답 지연",
    "status": "resolved",
    "assignee": "홍길동",
    "created_at": "2025-07-01T09:00:00Z",
    "resolved_at": "2025-07-01T11:30:00Z",
    "resolution_time_minutes": 150,
    "tags": ["performance", "backend"],
    "logs": [
      {"timestamp": "...", "message": "CPU 사용률 90% 초과"},
      {"timestamp": "...", "message": "스케일 아웃 실행"}
    ]
  }
```

### 1-4. 역색인 (Inverted Index) — ES의 핵심 자료구조

```
문서 저장:
  Doc 1: "서버 응답 지연 발생"
  Doc 2: "서버 재시작 완료"
  Doc 3: "응답 시간 정상화"

역색인 생성:
  "서버"   → [Doc 1, Doc 2]
  "응답"   → [Doc 1, Doc 3]
  "지연"   → [Doc 1]
  "재시작" → [Doc 2]
  "정상화" → [Doc 3]

검색: "서버 응답" → "서버"∩"응답" = [Doc 1]  (밀리초 단위)
```

- RDB의 LIKE '%서버%'는 전체 행을 스캔 → O(n)
- 역색인은 단어 → 문서 목록을 즉시 조회 → O(1)에 가까움

### 1-5. 클러스터 아키텍처 (AWS OpenSearch 기준)

```
┌─────────────────────────────────────────────────────┐
│              AWS OpenSearch Service                  │
│                                                     │
│  ┌──────────────────────┐                          │
│  │   Master Node(s)     │  ← 클러스터 상태 관리     │
│  │   (Dedicated 선택시) │     인덱스 생성/삭제 결정  │
│  └──────────────────────┘                          │
│                                                     │
│  ┌──────────────────────┐  ┌────────────────────┐  │
│  │   Data Node 1        │  │   Data Node 2      │  │
│  │   (t3.small 등)      │  │   (t3.small 등)    │  │
│  │                      │  │                    │  │
│  │  - 데이터 저장       │  │  - 데이터 저장     │  │
│  │  - 검색/집계 처리    │  │  - 검색/집계 처리  │  │
│  │  - EBS 볼륨 연결     │  │  - EBS 볼륨 연결   │  │
│  └──────────────────────┘  └────────────────────┘  │
│                                                     │
│  ┌──────────────────────┐                          │
│  │  OpenSearch Dashboards│  ← 시각화 UI            │
│  │  (Kibana 호환)       │                          │
│  └──────────────────────┘                          │
└─────────────────────────────────────────────────────┘
         ↑
    VPC Endpoint 또는 Public Access
         ↑
    [애플리케이션 / 개발자]
```

**저렴한 티어에서의 현실**:
- Dedicated Master Node 없을 수 있음 (Data Node가 Master 역할 겸임)
- 단일 AZ 배포일 수 있음 (HA 제한)
- → 인덱스 설계와 데이터 라이프사이클 관리가 더 중요

### 1-6. 문서의 생명주기

```
1. 색인 (Indexing)
   POST /logs-2025-07/_doc
   {"message": "에러 발생", "level": "ERROR", ...}
        ↓
2. 분석 (Analysis) — 텍스트 필드만 해당
   "에러 발생" → 토큰화 → ["에러", "발생"] → 역색인에 추가
        ↓
3. 저장 (Storage)
   Primary Shard에 저장 → Replica에 복제
        ↓
4. Refresh (기본 1초)
   메모리 버퍼 → 검색 가능한 세그먼트로 전환
        ↓
5. 검색 가능 (Searchable)
   GET /logs-2025-07/_search?q=에러
```

### 1-7. 학습 체크

- [ ] "왜 로그 저장에 RDB 대신 Elasticsearch를 쓰는가"를 3가지 이유로 설명할 수 있다
- [ ] Index / Document / Shard / Replica의 역할을 각각 설명할 수 있다
- [ ] 역색인(Inverted Index)이 왜 텍스트 검색에 빠른지 설명할 수 있다
- [ ] RDBMS와 Elasticsearch의 데이터 모델링 사고방식 차이를 설명할 수 있다
- [ ] AWS OpenSearch Service가 관리해주는 것과 직접 해야 하는 것을 구분할 수 있다
- [ ] Near Real-Time의 의미와 Refresh 주기의 관계를 설명할 수 있다

---
