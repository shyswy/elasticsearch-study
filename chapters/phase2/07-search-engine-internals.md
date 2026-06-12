## 7단계: 검색 엔진 내부 동작 원리

> **학습 목표**: 문서가 색인되고 검색되는 내부 메커니즘을 이해한다.
> "문서를 저장하면 실제로 무슨 일이 일어나는가?"
>
> **1단계에서 이어지는 질문**: NRT(Near Real-Time)와 Refresh의 상세 동작 원리
> — Refresh가 정확히 무엇을 하는지, Buffer→Segment 변환 과정,
> Refresh vs Flush의 차이, refresh_interval 튜닝의 트레이드오프를 여기서 깊이 다룬다.

### 7-1. 색인(Indexing) 내부 흐름

```
문서 색인 요청
      ↓
Coordinating Node (요청 수신)
      ↓
라우팅: shard = hash(document_id) % number_of_shards
      ↓
Primary Shard (해당 샤드가 있는 노드)
      ↓
1. 문서를 Translog에 기록 (내구성 보장, WAL과 유사)
2. In-Memory Buffer에 추가
      ↓
[Refresh] (기본 1초마다)
3. Buffer → Lucene Segment (검색 가능한 불변 파일)
      ↓
[Flush] (주기적)
4. Translog 비우기 + Segment를 디스크에 fsync
      ↓
Replica Shard에 복제
```

### 7-2. Lucene Segment 이해

```
Index (ES)
  └── Shard (= 1개의 Lucene Index)
       └── Segment 1 (불변 파일)
       └── Segment 2
       └── Segment 3
       └── ...

특성:
  - Segment는 한번 쓰면 변경 불가 (Immutable)
  - 삭제 = .del 파일에 마킹 (실제 삭제는 Merge 시)
  - 업데이트 = 기존 문서 삭제 마킹 + 새 문서 추가
  - Segment가 많으면 검색 시 모두 탐색 → Merge로 합침
```

**Segment Merge (세그먼트 병합)**:
```
Merge란?
  → 여러 개의 작은 Segment를 하나의 큰 Segment로 합치는 백그라운드 작업

왜 필요한가?
  1. Segment가 많으면 검색 시 모든 Segment를 탐색해야 함 → 느려짐
  2. 삭제된 문서(.del 마킹)가 실제로 디스크에서 제거되는 유일한 시점
  3. 디스크 공간 회수

동작 흐름:
  Segment A (100문서, 삭제마킹 20개)
  Segment B (80문서, 삭제마킹 5개)
  Segment C (50문서, 삭제마킹 0개)
       ↓ Merge
  Segment D (205문서, 삭제마킹 0개) ← 삭제된 25개 제거됨

특성:
  - ES가 자동으로 수행 (시점은 ES가 판단)
  - CPU/IO 부하 발생 → 저렴한 티어에서 주의
  - Force Merge: 수동으로 강제 병합 (쓰기 완료된 인덱스에만 권장)
    POST /logs-2025-07-01/_forcemerge?max_num_segments=1

문서 삭제가 비효율적인 이유:
  - 삭제 요청 → .del 파일에 마킹만 (즉시 디스크 회수 안 됨)
  - Merge가 일어나야 비로소 물리적 삭제 + 디스크 회수
  - Merge 시점은 예측 불가
  - vs 인덱스 통째 삭제: 즉시 모든 Segment 파일 삭제 → 즉시 디스크 회수
```

### 7-3. 텍스트 분석 (Analysis) 파이프라인

```
원본 텍스트: "The Quick Brown Fox jumped!"
      ↓
Character Filter: HTML 태그 제거 등
      ↓
Tokenizer: 공백/구두점 기준 분리
  → ["The", "Quick", "Brown", "Fox", "jumped!"]
      ↓
Token Filter:
  - lowercase: ["the", "quick", "brown", "fox", "jumped"]
  - stemmer: ["the", "quick", "brown", "fox", "jump"]
  - stop words 제거: ["quick", "brown", "fox", "jump"]
      ↓
역색인에 저장
```

**한국어 분석기**:
```json
PUT /korean-index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "korean": {
          "type": "custom",
          "tokenizer": "nori_tokenizer"
        }
      }
    }
  }
}
// "서버 응답 지연" → ["서버", "응답", "지연"]
```

### 7-4. 검색(Search) 내부 흐름

```
검색 요청
      ↓
Coordinating Node
      ↓
[Query Phase]
  → 모든 관련 Shard에 쿼리 전송 (병렬)
  → 각 Shard: 로컬에서 매칭 문서 ID + 점수 반환
  → Coordinating Node: 결과 병합 + 정렬 + Top N 선택
      ↓
[Fetch Phase]
  → Top N 문서의 실제 내용을 해당 Shard에서 가져옴
      ↓
최종 결과 반환
```

### 7-5. Relevance Scoring (BM25)

```
점수 = 얼마나 관련 있는 문서인가?

BM25 요소:
  - TF (Term Frequency): 검색어가 문서에 많이 나올수록 높음
  - IDF (Inverse Document Frequency): 전체 문서에서 드문 단어일수록 높음
  - Field Length: 짧은 필드에서 매칭되면 더 높음

예: "서버 에러" 검색
  Doc A (message: "서버 에러"): 짧은 필드 + 정확 매칭 → 높은 점수
  Doc B (message: "서버 점검 중 에러 발생으로 인한 서비스 중단"): 긴 필드 → 낮은 점수
```

### 7-6. 학습 체크

- [ ] 문서 색인 시 Translog → Buffer → Segment의 흐름을 설명할 수 있다
- [ ] Segment가 불변(Immutable)인 이유와 장단점을 설명할 수 있다
- [ ] Segment Merge의 역할과 문서 삭제가 비효율적인 이유를 설명할 수 있다
- [ ] Analyzer의 구성요소(Character Filter, Tokenizer, Token Filter)를 설명할 수 있다
- [ ] Query Phase와 Fetch Phase의 차이를 설명할 수 있다
- [ ] BM25 스코어링의 기본 원리를 설명할 수 있다

---
