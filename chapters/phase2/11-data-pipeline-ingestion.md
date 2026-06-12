## 11단계: 데이터 파이프라인 & 수집

> **학습 목표**: 다양한 소스에서 OpenSearch로 데이터를 수집하는 파이프라인을 설계할 수 있다.

### 11-1. 데이터 수집 아키텍처 패턴

```
패턴 1: 직접 색인 (애플리케이션 → OpenSearch)
  App → OpenSearch REST API
  장점: 단순, 지연 최소
  단점: 앱과 ES 결합도 높음, 장애 시 데이터 유실

패턴 2: 로그 수집기 경유
  App → 로그 파일 → Fluent Bit/Fluentd → OpenSearch
  장점: 앱과 분리, 버퍼링, 재시도
  단점: 약간의 지연

패턴 3: 메시지 큐 경유
  App → Kafka/SQS → Logstash/Lambda → OpenSearch
  장점: 높은 내구성, 백프레셔 처리
  단점: 복잡도 증가

패턴 4: AWS 네이티브
  CloudWatch Logs → Subscription Filter → Lambda → OpenSearch
  Kinesis Data Firehose → OpenSearch
```

### 11-2. Ingest Pipeline

OpenSearch 내부에서 문서를 변환/보강하는 파이프라인:

```json
PUT /_ingest/pipeline/log-pipeline
{
  "description": "로그 문서 전처리",
  "processors": [
    {
      "date": {
        "field": "timestamp_string",
        "formats": ["yyyy-MM-dd HH:mm:ss"],
        "target_field": "@timestamp"
      }
    },
    {
      "grok": {
        "field": "message",
        "patterns": ["%{LOGLEVEL:level} %{GREEDYDATA:log_message}"]
      }
    },
    {
      "remove": {
        "field": "timestamp_string"
      }
    }
  ]
}

// 파이프라인 적용하여 색인
POST /logs-2025-07-15/_doc?pipeline=log-pipeline
{
  "timestamp_string": "2025-07-15 09:30:00",
  "message": "ERROR Connection refused to database"
}
```

### 11-3. Data Prepper (OpenSearch 전용 수집기)

```
AWS 권장 수집 도구:
  Data Prepper = OpenSearch 전용 데이터 수집/변환 파이프라인

  소스 → Data Prepper → OpenSearch
  (OTel traces, logs, metrics 지원)
```

### 11-4. 학습 체크

- [ ] 4가지 데이터 수집 패턴의 장단점을 비교할 수 있다
- [ ] Ingest Pipeline으로 문서 전처리를 설정할 수 있다
- [ ] Grok 패턴으로 비정형 로그를 파싱할 수 있다
- [ ] 현재 프로젝트에 적합한 수집 아키텍처를 선택하고 이유를 설명할 수 있다

---
