## D2단계: Data Stream & Rollover 설계

> **학습 목표**: Data Stream의 내부 메커니즘을 이해하고,
> rollover 조건을 적절히 설정하여 자동 인덱스 관리를 구현할 수 있다.

### D2-1. Data Stream 내부 동작 상세

```
Data Stream 생성 흐름:

1. Index Template 생성 (data_stream: {} 포함)
2. 첫 문서 색인 시 Data Stream 자동 생성
3. Backing Index 자동 생성: .ds-<stream-name>-<date>-000001
4. 이후 문서는 모두 현재 write index에 저장
5. Rollover 조건 충족 시:
   - 현재 write index → read-only
   - 새 backing index 생성 (000002, 000003, ...)
   - write index 포인터 이동
```

### D2-2. Backing Index 네이밍 전략

```
기본 네이밍: .ds-<stream-name>-<generation>
  예: .ds-logs-000001, .ds-logs-000002

날짜 포함: .ds-<stream-name>-<YYYY.MM.dd>-<generation>
  예: .ds-logs-2025.07.15-000001

커스텀은 불가 — Data Stream이 자동 관리
→ 네이밍은 ES가 결정, 사용자는 stream 이름만 지정

검색 시:
  GET /logs/_search  ← stream 이름으로 (모든 backing index 대상)
  GET /.ds-logs-000003/_search  ← 특정 backing index 직접 (비권장)
```

### D2-3. Rollover 조건 설계 기준

```
Rollover 트리거 조건 (OR 로직 — 하나라도 충족 시 실행):

1. max_age: 시간 경과
   - 선택 기준: 데이터 신선도 요구사항
   - 예: "7d" → 최대 7일치 데이터가 하나의 backing index에
   - 팁: ISM의 min_index_age와 맞춰야 수명주기 계산 깔끔

2. max_size: 인덱스 전체 크기
   - 선택 기준: 샤드 크기 최적화
   - 예: "30gb" → shard 1개일 때 30GB 이하 유지
   - 주의: replica 포함 전체 크기 (primary만이 아님)

3. max_primary_shard_size: Primary Shard 단일 크기
   - 선택 기준: 개별 샤드 크기 제어 (multi-shard 인덱스일 때)
   - 예: "25gb" → 각 primary shard가 25GB 이하
   - max_size와 차이: multi-shard일 때 의미 다름

4. max_docs: 문서 수
   - 선택 기준: 문서 수 기반 관리 (크기보다 건수가 중요할 때)
   - 예: 1000000 → 100만 건마다 rollover
   - 주의: primary shard의 문서 수 기준

조건 조합 전략:
  - 일반 로그: max_age: "1d" + max_size: "20gb"
    → 하루가 지나거나 20GB 넘으면 rollover (둘 중 먼저)
  - 저렴한 티어: max_age: "7d" + max_size: "5gb"
    → 디스크 작으니까 크기 기준을 낮게
  - 대량 이벤트: max_docs: 5000000 + max_size: "30gb"
    → 검색 성능을 위해 문서 수도 제한
```

### D2-4. max_size vs max_primary_shard_size

```
인덱스: shard 3개, replica 1

max_size: "30gb"
  → 전체 인덱스 크기 (primary + replica 합산? 아니!)
  → 실제: primary shards의 합산 크기
  → 30GB = primary shard 3개 합산 (각 ~10GB)

max_primary_shard_size: "25gb"  
  → 개별 primary shard 하나의 크기
  → 어느 하나라도 25GB 넘으면 rollover

언제 뭘 쓰나?
  - shard 1개 인덱스: 둘 다 같은 효과 → max_size가 직관적
  - shard N개 인덱스: max_primary_shard_size가 개별 샤드 크기 제어에 적합
  - 저렴한 티어(shard 1개 권장): max_size 사용이 일반적
```

### D2-5. Write Index 전환 메커니즘

```
Rollover 전:
  .ds-logs-000001 (write: true, searchable: true)
  
Rollover 후:
  .ds-logs-000001 (write: false, searchable: true)  ← read-only
  .ds-logs-000002 (write: true, searchable: true)   ← 새 write index

색인 요청:
  POST /logs/_doc → 항상 현재 write index에 저장
  
검색 요청:
  GET /logs/_search → 모든 backing index 대상 (과거 + 현재)

무중단 보장:
  - Rollover 중에도 색인/검색 중단 없음
  - write index 전환은 atomic operation
  - 클라이언트는 data stream 이름만 알면 됨
```

### D2-6. Data Stream vs 수동 Rollover + Alias

```
Data Stream (자동):
  - @timestamp 필수
  - append-only (update/delete 불가)
  - rollover 자동 관리 (ISM 연동 시)
  - 네이밍 자동
  - 적합: 로그, 메트릭, 이벤트

수동 Rollover + Write Alias:
  PUT /logs-000001 { "aliases": { "logs-write": { "is_write_index": true } } }
  POST /logs-write/_rollover { "conditions": { "max_size": "20gb" } }
  
  - @timestamp 불필요
  - update/delete 가능
  - rollover 수동 또는 ISM에서 호출
  - 네이밍 커스텀 가능
  - 적합: 엔티티 데이터 + 주기적 교체가 필요한 경우

선택 기준:
  → append-only 시계열 → Data Stream
  → update/delete 필요 → 수동 Rollover + Alias
  → 두 가지 혼합 → 각각 별도 인덱스 전략
```

### D2-7. 학습 체크

- [ ] Data Stream의 backing index 생성 → write index 전환 흐름을 설명할 수 있다
- [ ] rollover 조건(max_age, max_size, max_primary_shard_size, max_docs)의 선택 근거를 설명할 수 있다
- [ ] max_size와 max_primary_shard_size의 차이를 shard 수 기준으로 설명할 수 있다
- [ ] Data Stream과 수동 Rollover + Alias의 선택 기준을 설명할 수 있다
- [ ] 주어진 데이터 규모와 제약 조건에서 적절한 rollover 조건을 설계할 수 있다

---
