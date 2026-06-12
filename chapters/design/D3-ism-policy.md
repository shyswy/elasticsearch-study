## D3단계: ISM 정책 설계

> **학습 목표**: Index State Management의 상태 머신 모델을 이해하고,
> 다단계 수명주기 정책을 설계할 수 있다.

### D3-1. ISM 상태 머신 모델

```
ISM Policy = 상태(State) + 전환(Transition) + 액션(Action)의 조합

┌─────────┐   min_index_age: 7d   ┌─────────┐   min_index_age: 30d   ┌─────────┐
│   hot   │ ───────────────────→  │  warm   │ ───────────────────→  │  delete │
│         │                        │         │                        │         │
│ 액션:   │                        │ 액션:   │                        │ 액션:   │
│ (없음)  │                        │ replica │                        │ delete  │
│         │                        │ → 0     │                        │         │
│         │                        │ force   │                        │         │
│         │                        │ merge   │                        │         │
└─────────┘                        └─────────┘                        └─────────┘

핵심 개념:
- State: 인덱스의 현재 상태 (이름은 자유롭게 지정)
- Action: 해당 상태에 진입할 때 실행되는 작업
- Transition: 다음 상태로 넘어가는 조건
- default_state: 인덱스가 처음 할당받는 상태
```

### D3-2. 사용 가능한 Action 종류

```
인덱스 설정 변경:
  - read_only: 인덱스를 읽기 전용으로 변경
  - read_write: 읽기/쓰기 가능으로 되돌림
  - replica_count: replica 수 변경
  - index_priority: 복구 우선순위 변경

인덱스 최적화:
  - force_merge: 세그먼트 병합 (max_num_segments 지정)
  - shrink: 샤드 수 줄이기 (ex: 5 → 1)
  - close: 인덱스 닫기 (검색 불가, 디스크 절약)
  - open: 닫힌 인덱스 열기

데이터 이동:
  - allocation: 특정 노드로 샤드 이동 (hot → warm 노드)
  - snapshot: 스냅샷 생성

수명주기:
  - rollover: 새 인덱스로 교체 (Data Stream / Alias 사용 시)
  - delete: 인덱스 삭제

알림:
  - notification: 상태 전환 시 알림 발송 (SNS, Slack 등)
```

### D3-3. Transition 조건 종류

```
시간 기반:
  - min_index_age: "7d"     ← 인덱스 생성 후 경과 시간
  - min_rollover_age: "1d"  ← 마지막 rollover 후 경과 시간

크기 기반:
  - min_size: "30gb"        ← 인덱스 전체 크기
  - min_primary_shard_size: "25gb"

문서 수 기반:
  - min_doc_count: 1000000

Cron 기반:
  - cron:
      expression: "0 0 * * *"   ← 매일 자정
      timezone: "Asia/Seoul"

조건 조합 (AND):
  transitions에 conditions를 여러 개 넣으면 AND로 동작
  OR이 필요하면 별도 transition을 여러 개 정의
```

### D3-4. ISM + Rollover 연동 패턴

```json
// Data Stream과 ISM 연동 — rollover를 ISM이 관리
{
  "policy": {
    "description": "로그 수명주기: rollover → warm → delete",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [
          {
            "rollover": {
              "min_size": "10gb",
              "min_index_age": "1d"
            }
          }
        ],
        "transitions": [
          {
            "state_name": "warm",
            "conditions": {
              "min_rollover_age": "7d"
            }
          }
        ]
      },
      {
        "name": "warm",
        "actions": [
          { "replica_count": { "number_of_replicas": 0 } },
          { "force_merge": { "max_num_segments": 1 } }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": {
              "min_index_age": "30d"
            }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          { "delete": {} }
        ]
      }
    ],
    "ism_template": [
      {
        "index_patterns": ["logs*"],
        "priority": 100
      }
    ]
  }
}
```

### D3-5. ISM 에러 핸들링

```
ISM 액션 실패 시 동작:

기본 동작:
  - 액션 실패 → 재시도 (기본 3회, 지수 백오프)
  - 모든 재시도 실패 → 정책이 "failed" 상태로 전환
  - 수동 개입 필요: POST /_plugins/_ism/retry/<index>

커스텀 에러 핸들링:
{
  "error_notification": {
    "destination": {
      "slack": { "url": "https://hooks.slack.com/..." }
    },
    "message_template": {
      "source": "ISM 정책 실패: {{ctx.index}} - {{ctx.error}}"
    }
  }
}

재시도 설정:
{
  "actions": [
    {
      "retry": {
        "count": 5,
        "backoff": "exponential",
        "delay": "1h"
      },
      "force_merge": { "max_num_segments": 1 }
    }
  ]
}
```

### D3-6. ism_template vs 수동 정책 연결

```
ism_template (자동 연결):
  - 정책 내에 index_patterns 정의
  - 패턴에 매칭되는 새 인덱스에 자동 적용
  - Data Stream의 backing index에도 자동 적용
  - priority로 여러 정책 간 우선순위 결정

수동 연결:
  POST /_plugins/_ism/add/logs-2025-07-15
  { "policy_id": "log-retention-policy" }
  → 기존 인덱스에 정책 적용할 때

정책 변경 시:
  - 이미 적용된 인덱스: 기존 정책 계속 사용
  - 새 인덱스: 변경된 정책 적용
  - 기존 인덱스에 새 정책 적용:
    POST /_plugins/_ism/change_policy/logs-*
    { "policy_id": "new-policy" }
```

### D3-7. 저렴한 티어를 위한 ISM 설계 원칙

```
1. Hot-Warm-Cold 중 Hot-Delete만 사용 (노드 1개니까 warm 노드 없음)
2. force_merge는 삭제 전에 불필요 (어차피 삭제할 인덱스)
3. replica_count 변경 불필요 (이미 0)
4. rollover 크기를 작게 (디스크 제한):
   - max_size: "5gb" 또는 max_primary_shard_size: "5gb"
5. 삭제 주기를 짧게 (디스크 절약):
   - 로그: 7~14일
   - Issue History: 90일 또는 무기한 (크기가 작으니)
6. 에러 알림 필수 (디스크 풀 나면 클러스터 RED)
```

### D3-8. 학습 체크

- [ ] ISM의 상태 머신 모델(State, Action, Transition)을 도식으로 설명할 수 있다
- [ ] 사용 가능한 Action 종류를 5가지 이상 나열하고 각각의 용도를 설명할 수 있다
- [ ] Transition 조건(시간/크기/문서수/cron)의 선택 기준을 설명할 수 있다
- [ ] ISM + Rollover 연동 패턴(hot → warm → delete)을 작성할 수 있다
- [ ] ISM 에러 핸들링(재시도, 알림)을 설정할 수 있다
- [ ] ism_template과 수동 정책 연결의 차이를 설명할 수 있다
- [ ] 저렴한 티어에서의 ISM 설계 원칙을 3가지 이상 설명할 수 있다

---
