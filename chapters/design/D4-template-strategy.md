## D4단계: 템플릿 구성 전략

> **학습 목표**: Component Template과 Index Template의 조합으로
> 재사용 가능한 매핑/설정 체계를 설계할 수 있다.

### D4-1. Template 계층 구조

```
Component Template (재사용 가능한 빌딩 블록)
  ├── settings-component   ← 공통 설정 (shard 수, refresh_interval)
  ├── mappings-base        ← 공통 매핑 (@timestamp, service 등)
  └── mappings-logs        ← 로그 전용 매핑 (level, message 등)

Index Template (최종 조합)
  └── logs-template
       ├── composed_of: [settings-component, mappings-base, mappings-logs]
       ├── index_patterns: ["logs-*"]
       └── data_stream: {}

최종 인덱스 = Component Templates 합산 + Index Template 고유 설정
```

### D4-2. Component Template 상세

```json
// 공통 설정 컴포넌트
PUT /_component_template/settings-common
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "5s"
    }
  }
}

// 공통 매핑 컴포넌트 (모든 인덱스에 공통인 필드)
PUT /_component_template/mappings-base
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service": { "type": "keyword" },
        "environment": { "type": "keyword" },
        "trace_id": { "type": "keyword" }
      }
    }
  }
}

// 로그 전용 매핑 컴포넌트
PUT /_component_template/mappings-logs
{
  "template": {
    "mappings": {
      "properties": {
        "level": { "type": "keyword" },
        "message": { "type": "text" },
        "host": { "type": "keyword" },
        "response_time_ms": { "type": "integer" }
      }
    }
  }
}
```

### D4-3. Composable Index Template

```json
// 로그용 Index Template (component 조합)
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "composed_of": ["settings-common", "mappings-base", "mappings-logs"],
  "priority": 200,
  "template": {
    "settings": {
      "index.lifecycle.name": "log-retention-policy"
    }
  }
}

// Issue History용 Index Template
PUT /_index_template/issue-history-template
{
  "index_patterns": ["issue-history-*"],
  "composed_of": ["settings-common", "mappings-base"],
  "priority": 100,
  "template": {
    "mappings": {
      "properties": {
        "issue_id": { "type": "keyword" },
        "title": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
        "status": { "type": "keyword" },
        "priority": { "type": "keyword" },
        "assignee": { "type": "keyword" },
        "resolution_time_minutes": { "type": "integer" }
      }
    }
  }
}
```

### D4-4. Template 우선순위 (Priority) 규칙

```
여러 Template이 같은 index_patterns에 매칭될 때:
  → priority가 높은 Template이 우선 적용
  → 같은 priority면 에러 (명시적 해결 필요)

예:
  logs-template (priority: 200, pattern: "logs-*")
  catch-all-template (priority: 50, pattern: "*")
  
  → "logs-2025-07-15" 생성 시: logs-template 적용 (200 > 50)
  → "users" 생성 시: catch-all-template 적용 (유일 매칭)

Component Template 간 병합:
  - composed_of 배열 순서대로 병합
  - 뒤에 오는 것이 앞의 것을 override
  - Index Template 자체의 template이 최종 override

  composed_of: ["settings-common", "mappings-base", "mappings-logs"]
  → settings-common 적용 → mappings-base 병합 → mappings-logs 병합
  → Index Template의 template 필드가 최종 덮어쓰기
```

### D4-5. Legacy Template vs Composable Template

```
Legacy (_template API):
  PUT /_template/logs-template
  - 단순하지만 재사용 불가
  - component template 개념 없음
  - 여러 패턴 매칭 시 all merged (예측 어려움)
  - deprecated (7.8+)

Composable (_index_template API):
  PUT /_index_template/logs-template
  - component template으로 재사용
  - priority로 명확한 우선순위
  - 한 인덱스에 하나의 template만 적용
  - 현재 권장 방식

주의: 둘이 공존하면 composable이 우선 적용됨
→ 레거시 템플릿은 제거하고 composable로 마이그레이션 권장
```

### D4-6. 학습 체크

- [ ] Component Template과 Index Template의 관계를 설명할 수 있다
- [ ] `composed_of` 배열의 병합 순서와 override 규칙을 설명할 수 있다
- [ ] `priority`가 같은 두 Template이 충돌할 때 동작을 설명할 수 있다
- [ ] 공통 설정/매핑을 Component Template으로 분리하는 설계를 할 수 있다
- [ ] Legacy Template과 Composable Template의 차이를 설명할 수 있다
- [ ] 현재 프로젝트에 맞는 Template 계층 구조를 설계할 수 있다

---
