## D7단계: Data Stream 설계 실습

> **학습 목표**: 시계열 데이터에 적합한 Data Stream을 처음부터 설계하고,
> Index Template + Rollover 조건을 포함하여 완성할 수 있다.
>
> **참조**: D2(Data Stream 이론), D4(템플릿 구성)

### D7-1. 실습 과제

```
과제: 다음 요구사항을 만족하는 Data Stream을 설계하라.

[요구사항]
- 서비스: API Gateway 액세스 로그
- 데이터 특성:
  - append-only (수정/삭제 없음)
  - 일간 약 500만 건, 평균 문서 크기 800B
  - 일간 raw 데이터량: ~4GB
  - 피크 시간(10시~12시)에 60% 집중
  
- 필드:
  - @timestamp: 요청 시간
  - method: HTTP 메서드 (GET, POST 등)
  - path: 요청 경로 (정확 매칭 + 집계)
  - status_code: HTTP 상태 코드 (숫자, 집계)
  - response_time_ms: 응답 시간 (숫자, 집계 + 백분위)
  - client_ip: 클라이언트 IP (keyword)
  - user_agent: UA 문자열 (저장만, 검색 드뭄)
  - request_body: 요청 본문 (저장만, 검색/집계 불필요)
  - error_message: 에러 메시지 (풀텍스트 검색)
  
- 운영 요구:
  - 검색 대상: 최근 3일은 빠른 검색 필요
  - 보존: 14일 후 삭제
  - 클러스터: r6g.large 3노드, EBS 500GB 각

[산출물]
1. Component Template(s) JSON
2. Index Template JSON (data_stream 포함)
3. Rollover 조건 설정 및 근거
4. 용량 계획 (14일 기준 총 디스크 사용량 예측)
```

### D7-2. 설계 체크포인트

```
✓ @timestamp 필드가 매핑에 포함되었는가?
✓ data_stream: {} 가 Index Template에 설정되었는가?
✓ Rollover 조건이 샤드 크기 최적 범위(10~50GB)를 유지하도록 설정되었는가?
✓ Component Template으로 재사용 가능한 부분을 분리했는가?
✓ 14일 보존 시 총 디스크 사용량이 가용 용량의 85% 이내인가?
✓ 피크 시간의 색인 부하를 고려한 refresh_interval 설정이 있는가?
```

### D7-3. 학습 체크

- [ ] Data Stream용 Index Template을 작성할 수 있다
- [ ] 데이터 규모에 맞는 Rollover 조건을 설계할 수 있다
- [ ] Component Template을 활용한 매핑 분리를 적용할 수 있다
- [ ] 총 디스크 사용량을 예측하고 용량 계획을 수립할 수 있다

---
