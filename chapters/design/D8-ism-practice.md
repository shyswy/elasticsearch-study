## D8단계: ISM 정책 설계 실습

> **학습 목표**: 주어진 운영 요구사항을 ISM 정책으로 변환하고,
> 상태 머신을 설계하여 완전한 정책 JSON을 작성할 수 있다.
>
> **참조**: D3(ISM 이론), D2(Rollover)

### D8-1. 실습 과제

```
과제: 다음 운영 요구사항을 만족하는 ISM 정책을 설계하라.

[요구사항]
- 대상: API Gateway 액세스 로그 (D7에서 설계한 Data Stream)
- 수명주기:
  - Hot (0~3일): 활발한 읽기/쓰기, replica 1
  - Warm (3~7일): 읽기 전용, replica 0, 세그먼트 병합
  - Cold (7~14일): 읽기 전용, 최소 리소스
  - Delete (14일 후): 삭제
  
- 추가 요구:
  - Hot에서 rollover: 1일 경과 또는 20GB 초과 시
  - Warm 전환 시: force_merge (max_num_segments: 1)
  - Cold 전환 시: index priority 낮추기 (50 → 1)
  - 삭제 실패 시: Slack 알림
  - 에러 시 재시도: 3회, 지수 백오프

[산출물]
1. ISM 정책 JSON (전체)
2. 상태 전환 다이어그램 (텍스트)
3. 각 Action/Transition 선택 근거
4. ism_template 설정 (어떤 인덱스에 자동 적용?)
```

### D8-2. 설계 체크포인트

```
✓ 상태 머신이 순환하지 않는가? (단방향: hot → warm → cold → delete)
✓ Rollover가 hot 상태의 action에 포함되었는가?
✓ Warm 전환 조건이 min_rollover_age인가? (rollover 후 경과 시간)
✓ force_merge를 read-only 전환 후에 실행하는가?
✓ 에러 핸들링(retry, notification)이 포함되었는가?
✓ ism_template의 index_patterns이 Data Stream의 backing index 패턴과 매칭되는가?
✓ 저렴한 티어/일반 티어별 조건이 적절한가?
```

### D8-3. 학습 체크

- [ ] 운영 요구사항을 상태 머신 다이어그램으로 변환할 수 있다
- [ ] 각 상태의 Action과 Transition을 적절히 설정할 수 있다
- [ ] Rollover + ISM 연동을 올바르게 구성할 수 있다
- [ ] 에러 핸들링과 알림을 포함한 완전한 ISM 정책을 작성할 수 있다

---
