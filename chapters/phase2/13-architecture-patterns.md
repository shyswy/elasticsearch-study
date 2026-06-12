## 13단계: 대규모 운영 패턴 & 아키텍처

> **학습 목표**: 데이터 증가와 요구사항 변화에 대응하는 아키텍처 패턴을 이해한다.

### 13-1. Hot-Warm-Cold 아키텍처

```
Hot (최근 데이터):
  → 빠른 인스턴스 (SSD)
  → 활발한 읽기/쓰기
  → 최근 1~7일 로그

Warm (중간 데이터):
  → 저렴한 인스턴스 (HDD)
  → 읽기 위주, 쓰기 없음
  → 7~30일 로그

Cold (오래된 데이터):
  → 최소 리소스 또는 S3 (UltraWarm)
  → 드문 읽기
  → 30일+ 로그

ISM 정책으로 자동 전환:
  Hot (7일) → Warm (30일) → Cold (90일) → Delete
```

**AWS OpenSearch UltraWarm**:
```
- S3 기반 스토리지 (매우 저렴)
- 읽기 전용
- 검색 가능하지만 느림
- 저렴한 티어에서는 사용 불가할 수 있음 → ISM으로 삭제가 현실적
```

### 13-2. 멀티 테넌시 패턴

```
패턴 1: 인덱스 분리
  logs-team-a-2025-07
  logs-team-b-2025-07
  → 격리 완벽, 관리 복잡

패턴 2: 필드 기반 분리 + FGAC
  logs-2025-07 (team 필드로 구분)
  → 관리 단순, FGAC로 접근 제어

패턴 3: 클러스터 분리
  → 완전 격리, 비용 높음
```

### 13-3. 인덱스 전략 패턴

```
시계열 데이터 (로그):
  → 날짜별 인덱스 + ISM + Rollover
  → logs-000001, logs-000002, ... (크기/시간 기반 롤오버)

엔티티 데이터 (Issue History):
  → 단일 인덱스 + Alias
  → 매핑 변경 시 Reindex + Alias 전환

하이브리드:
  → 월별 인덱스 + Alias (issue-history-2025-07)
  → 검색은 alias로 (issue-history-current)
```

**Index Rollover**:
```json
// 크기 또는 문서 수 기반으로 새 인덱스 자동 생성
POST /logs-000001/_rollover
{
  "conditions": {
    "max_size": "10gb",
    "max_age": "7d",
    "max_docs": 1000000
  }
}
```

### 13-4. 스케일링 전략

```
수직 스케일링 (Scale Up):
  t3.small → t3.medium → r6g.large
  → 단순하지만 한계 있음

수평 스케일링 (Scale Out):
  노드 추가 → 샤드 재분배
  → 저렴한 티어에서는 비용 증가

데이터 축소:
  → 보존 기간 단축
  → 불필요한 필드 제거
  → 샘플링 (모든 로그를 저장하지 않기)
```

### 13-5. OpenSearch vs Elasticsearch 차이점 정리

| 항목 | Elasticsearch | OpenSearch |
|------|--------------|-----------|
| 라이선스 | Elastic License 2.0 / SSPL | Apache 2.0 |
| 보안 | X-Pack (유료) | Security Plugin (무료 내장) |
| 알림 | Watcher (유료) | Alerting Plugin (무료 내장) |
| ISM | ILM (Index Lifecycle Management) | ISM (Index State Management) |
| 시각화 | Kibana | OpenSearch Dashboards |
| API 호환 | ES 7.10 기준 대부분 호환 | `_plugins/` 경로 추가 |
| 관리형 서비스 | Elastic Cloud | AWS OpenSearch Service |

### 13-6. 학습 체크

- [ ] Hot-Warm-Cold 아키텍처의 각 단계 역할을 설명할 수 있다
- [ ] Index Rollover의 동작 방식과 사용 시점을 설명할 수 있다
- [ ] 현재 프로젝트의 데이터 증가에 대응하는 스케일링 전략을 제안할 수 있다
- [ ] OpenSearch와 Elasticsearch의 주요 차이점을 설명할 수 있다
- [ ] 멀티 테넌시 패턴 중 현재 환경에 적합한 것을 선택하고 이유를 설명할 수 있다

---
