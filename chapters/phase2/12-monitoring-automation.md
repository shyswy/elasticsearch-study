## 12단계: 모니터링 & 운영 자동화

> **학습 목표**: 클러스터 상태를 지속적으로 모니터링하고,
> 알림 설정과 운영 자동화를 구축할 수 있다.

### 12-1. CloudWatch 기반 모니터링

```
핵심 알림 설정:
  - ClusterStatus.red > 0 → 즉시 알림 (P1)
  - FreeStorageSpace < 5GB → 경고 (디스크 부족 임박)
  - JVMMemoryPressure > 80% → 경고 (OOM 위험)
  - CPUUtilization > 90% (5분 지속) → 경고
  - 4xx/5xx 에러 수 급증 → 경고
```

### 12-2. OpenSearch Alerting (내장 알림)

```json
// 모니터 생성 (OpenSearch Dashboards → Alerting)
{
  "name": "High Error Rate Alert",
  "schedule": { "period": { "interval": 5, "unit": "MINUTES" } },
  "inputs": [{
    "search": {
      "indices": ["logs-*"],
      "query": {
        "bool": {
          "filter": [
            { "term": { "level": "ERROR" } },
            { "range": { "@timestamp": { "gte": "now-5m" } } }
          ]
        }
      }
    }
  }],
  "triggers": [{
    "name": "error_spike",
    "condition": { "script": { "source": "ctx.results[0].hits.total.value > 100" } },
    "actions": [{
      "name": "notify_slack",
      "destination_id": "slack-webhook-id",
      "message_template": {
        "source": "최근 5분간 ERROR 로그 {{ctx.results[0].hits.total.value}}건 발생"
      }
    }]
  }]
}
```

### 12-3. 운영 자동화 체크리스트

```
일간:
  - 클러스터 상태 확인 (green/yellow/red)
  - 디스크 사용량 확인

주간:
  - ISM 정책 동작 확인 (오래된 인덱스 삭제됐는지)
  - 느린 쿼리 로그 리뷰

월간:
  - 인덱스 크기 추이 확인
  - 스냅샷 복원 테스트
  - 버전 업그레이드 검토
```

### 12-4. 학습 체크

- [ ] CloudWatch 알림으로 클러스터 이상을 감지하는 설정을 구성할 수 있다
- [ ] OpenSearch Alerting으로 비즈니스 알림을 설정할 수 있다
- [ ] 일간/주간/월간 운영 체크리스트를 실행할 수 있다
- [ ] 알림 피로(Alert Fatigue)를 줄이는 임계값 설정 전략을 설명할 수 있다

---
