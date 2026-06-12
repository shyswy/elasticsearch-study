## 5단계: AWS OpenSearch Service 운영

> **학습 목표**: AWS 콘솔과 API를 통해 OpenSearch 도메인을 관리하고,
> 접근 제어, 백업, 업그레이드 등 운영 작업을 수행할 수 있다.

### 5-1. OpenSearch Service 도메인 구성 요소

```
AWS OpenSearch Service Domain
├── 클러스터 설정
│   ├── 인스턴스 타입 (t3.small.search 등)
│   ├── 인스턴스 수
│   ├── 스토리지 (EBS gp3)
│   └── AZ 배치 (단일 AZ / 다중 AZ)
├── 접근 제어
│   ├── Fine-Grained Access Control (FGAC)
│   ├── IAM 정책
│   └── VPC / Public 접근
├── 보안
│   ├── 전송 중 암호화 (TLS)
│   ├── 저장 시 암호화 (KMS)
│   └── 노드 간 암호화
└── 모니터링
    ├── CloudWatch 메트릭
    └── 감사 로그
```

### 5-2. 접근 방법

**OpenSearch Dashboards (UI)**:
- URL: `https://<domain-endpoint>/_dashboards`
- Kibana와 거의 동일한 UI
- Dev Tools 콘솔에서 REST API 직접 실행 가능

**REST API (CLI/코드)**:
```bash
# curl로 직접 호출
curl -XGET "https://<domain-endpoint>/issue-history/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"match_all": {}}}'

# AWS 서명 필요 시 (IAM 인증)
aws opensearch-serverless ... # 또는 awscurl 사용
```

**SDK (애플리케이션에서)**:
```python
# Python - opensearch-py
from opensearchpy import OpenSearch

client = OpenSearch(
    hosts=[{'host': '<domain-endpoint>', 'port': 443}],
    http_auth=('username', 'password'),
    use_ssl=True
)

response = client.search(
    index='issue-history',
    body={'query': {'match_all': {}}}
)
```

### 5-3. 저렴한 티어 운영 핵심

**인스턴스 제약 이해**:
```
t3.small.search:
  - vCPU: 2
  - 메모리: 2GB
  - JVM Heap: ~1GB (메모리의 50%)
  - 네트워크: 저~중간

→ 동시 검색/색인 부하에 취약
→ 대량 집계 시 메모리 부족 가능
→ 인덱스/샤드 수 최소화 필수
```

**비용 최적화 전략**:
```
1. 불필요한 replica 제거 (단일 노드면 replica 의미 없음)
2. 오래된 인덱스 삭제 (ISM 정책으로 자동화)
3. 불필요한 필드 색인 제외 ("enabled": false)
4. refresh_interval 늘리기 (1s → 30s, 실시간성 불필요 시)
5. 로그 보존 기간 최소화 (7일? 30일?)
```

### 5-4. ISM (Index State Management) — 인덱스 수명주기 관리

```json
PUT /_plugins/_ism/policies/log-retention-policy
{
  "policy": {
    "description": "30일 후 로그 인덱스 자동 삭제",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [],
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
        "index_patterns": ["logs-*"],
        "priority": 100
      }
    ]
  }
}
```

→ `logs-*` 패턴의 인덱스가 30일 지나면 자동 삭제

### 5-5. 스냅샷 (백업)

```
AWS OpenSearch Service 자동 스냅샷:
  - 매일 자동 생성 (14일 보존)
  - 수동 스냅샷도 S3에 저장 가능

수동 스냅샷 등록:
  1. S3 버킷 생성
  2. IAM Role 생성 (OpenSearch → S3 접근)
  3. 스냅샷 리포지토리 등록
  4. 스냅샷 생성/복원
```

```json
// 스냅샷 리포지토리 등록
PUT /_snapshot/my-s3-repo
{
  "type": "s3",
  "settings": {
    "bucket": "my-opensearch-snapshots",
    "region": "ap-northeast-2",
    "role_arn": "arn:aws:iam::123456789:role/opensearch-snapshot-role"
  }
}

// 스냅샷 생성
PUT /_snapshot/my-s3-repo/snapshot-2025-07-15

// 스냅샷 복원
POST /_snapshot/my-s3-repo/snapshot-2025-07-15/_restore
```

### 5-6. CloudWatch 모니터링 핵심 메트릭

| 메트릭 | 의미 | 위험 임계값 |
|--------|------|-----------|
| `ClusterStatus.red` | 일부 Primary Shard 미할당 | > 0 |
| `ClusterStatus.yellow` | 일부 Replica 미할당 | 단일노드면 정상 |
| `FreeStorageSpace` | 남은 디스크 공간 | < 20% |
| `JVMMemoryPressure` | JVM 힙 사용률 | > 80% |
| `CPUUtilization` | CPU 사용률 | > 80% 지속 |
| `SearchLatency` | 검색 응답 시간 | 증가 추세 |
| `IndexingLatency` | 색인 응답 시간 | 증가 추세 |

### 5-7. 학습 체크

- [ ] OpenSearch Dashboards의 Dev Tools에서 쿼리를 실행할 수 있다
- [ ] ISM 정책으로 오래된 로그 인덱스를 자동 삭제하는 설정을 작성할 수 있다
- [ ] 저렴한 티어에서 비용/성능 최적화를 위해 해야 할 것 3가지를 설명할 수 있다
- [ ] CloudWatch에서 클러스터 상태를 모니터링하는 핵심 메트릭을 알고 있다
- [ ] 스냅샷으로 인덱스를 백업/복원하는 흐름을 설명할 수 있다
- [ ] Fine-Grained Access Control의 역할을 설명할 수 있다

---
