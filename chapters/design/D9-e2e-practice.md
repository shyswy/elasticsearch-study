## D9단계: 통합 설계 실습 (End-to-End)

> **학습 목표**: 새로운 데이터 저장 요구사항이 주어졌을 때,
> 매핑 + 템플릿 + Data Stream/Alias + ISM + 용량 계획을 모두 포함하는
> 완전한 인덱스 전략을 설계할 수 있다.
>
> **참조**: D1~D5 전체 + Phase 1의 2, 5단계

### D9-1. 실습 과제

```
과제: 새 서비스의 전체 인덱스 전략을 설계하라.

[시나리오]
새로운 마이크로서비스 "Alert Manager"가 추가된다.
이 서비스는 두 종류의 데이터를 OpenSearch에 저장해야 한다:

데이터 A — Alert Event 로그:
  - 알림 발생/해제 이벤트 (append-only, 시계열)
  - 일간 약 10만 건, 평균 1KB
  - 검색: "최근 1시간 critical alert", "서비스별 alert 빈도"
  - 집계: 시간대별 alert 수, 서비스별 MTTR
  - 보존: 30일

데이터 B — Alert Rule 설정:
  - 알림 규칙 정의 (CRUD, 엔티티)
  - 총 ~500개 규칙, 수시 업데이트
  - 검색: "특정 서비스의 활성 규칙", "조건 키워드 검색"
  - 보존: 무기한 (삭제 안 함)

[환경]
- 기존 클러스터에 추가 (t3.medium 2노드, 현재 사용량 50%)
- 기존에 logs-* 인덱스와 issue-history 인덱스가 운영 중
- 기존 Component Template: settings-common, mappings-base 있음

[산출물]
1. 데이터 A의 전체 설계:
   - 매핑 + Component Template + Index Template + Data Stream
   - ISM 정책
   - Rollover 조건
   - 용량 계획

2. 데이터 B의 전체 설계:
   - 매핑 + Index Template
   - Alias 전략 (향후 마이그레이션 대비)
   - 백업 전략

3. 기존 환경과의 호환성:
   - 기존 Template과의 priority 충돌 없는지
   - 기존 ISM 정책과의 ism_template 패턴 충돌 없는지
   - 추가 후 총 샤드 수/디스크 사용량 검증
```

### D9-2. 설계 체크포인트

```
✓ 데이터 A와 B의 특성 차이(시계열 vs 엔티티)를 반영한 전략인가?
✓ 기존 Component Template을 재사용했는가?
✓ Priority 충돌이 없는가?
✓ ISM의 ism_template 패턴이 기존 정책과 겹치지 않는가?
✓ 추가 후 총 샤드 수가 노드당 한도 내인가?
✓ 추가 후 디스크 사용량이 85% watermark 이내인가?
✓ 향후 스케일링 계획(데이터 증가 시)이 포함되었는가?
```

### D9-3. 학습 체크

- [ ] 시계열 데이터와 엔티티 데이터에 각각 적합한 인덱스 전략을 선택할 수 있다
- [ ] Component Template 재사용으로 일관된 설계를 유지할 수 있다
- [ ] 기존 환경과의 호환성(priority, 패턴 충돌, 용량)을 검증할 수 있다
- [ ] end-to-end 설계 산출물(매핑 + 템플릿 + ISM + 용량계획)을 완성할 수 있다
- [ ] 설계 의사결정의 근거를 다른 사람에게 설명할 수 있다

---
