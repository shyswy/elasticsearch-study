## 9단계: 성능 최적화

> **학습 목표**: 검색/색인 성능을 측정하고 병목을 찾아 최적화할 수 있다.

### 9-1. 성능 측정

```json
// 쿼리 실행 시간 확인
GET /logs-*/_search
{
  "profile": true,
  "query": { "match": { "message": "error" } }
}

// Slow Log 설정
PUT /logs-*/_settings
{
  "index.search.slowlog.threshold.query.warn": "5s",
  "index.search.slowlog.threshold.query.info": "2s",
  "index.indexing.slowlog.threshold.index.warn": "10s"
}
```

### 9-2. 검색 성능 최적화

```
1. filter 활용 (캐시됨, 점수 계산 안 함)
   → 날짜 범위, 상태 필터는 반드시 filter 절에

2. 필요한 필드만 반환
   "_source": ["field1", "field2"]

3. 인덱스 범위 좁히기
   GET /logs-2025-07-15/_search  (전체 logs-* 대신)

4. 적절한 size 설정
   기본 10개면 충분한데 10000개 요청하지 않기

5. keyword 필드 활용
   정확한 매칭은 term + keyword (text 필드에 term 쓰지 않기)

6. 불필요한 집계 제거
   size: 0 + 필요한 aggs만
```

### 9-3. 색인 성능 최적화

```
1. Bulk API 사용 (단건 색인 반복 금지)
   → 배치 크기: 5~15MB 또는 1000~5000 문서

2. refresh_interval 늘리기
   → 대량 색인 시 "30s" 또는 "-1" (수동 refresh)

3. replica 일시 제거 후 색인, 완료 후 복원
   → 대량 초기 데이터 로딩 시

4. 매핑 최적화
   → 불필요한 필드: "enabled": false
   → 검색 불필요한 필드: "index": false
```

### 9-4. 메모리 최적화 (저렴한 티어 필수)

```
JVM Heap 사용처:
  - Segment 메타데이터 (샤드 수에 비례)
  - Fielddata (text 필드 집계 시 — 피해야 함)
  - Query Cache (filter 결과 캐시)
  - Request Cache (집계 결과 캐시)

최적화:
  1. 샤드 수 최소화 (인덱스당 1개)
  2. text 필드에 집계하지 않기 (keyword 사용)
  3. 오래된 인덱스 삭제 (ISM 정책)
  4. 불필요한 doc_values 비활성화
```

### 9-5. 학습 체크

- [ ] `profile` API로 쿼리 병목을 분석할 수 있다
- [ ] Slow Log를 설정하고 느린 쿼리를 식별할 수 있다
- [ ] Bulk API의 적절한 배치 크기를 설명할 수 있다
- [ ] 저렴한 티어에서 JVM Heap 사용을 줄이는 방법을 3가지 이상 설명할 수 있다
- [ ] filter vs must의 성능 차이를 캐시 관점에서 설명할 수 있다

---
