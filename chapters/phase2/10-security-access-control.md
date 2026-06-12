## 10단계: 보안 & 접근 제어

> **학습 목표**: OpenSearch의 보안 모델을 이해하고 적절한 접근 제어를 설정할 수 있다.

### 10-1. AWS OpenSearch 보안 레이어

```
레이어 1: 네트워크 접근 제어
  → VPC 내 배치 (Private Subnet)
  → Security Group으로 IP/포트 제한
  → 또는 Public Access + IP 기반 정책

레이어 2: IAM 정책 (AWS 레벨)
  → 도메인 접근 권한 (es:ESHttp*)
  → 리소스 기반 정책 (Domain Access Policy)

레이어 3: Fine-Grained Access Control (FGAC)
  → 인덱스/문서/필드 레벨 접근 제어
  → Internal User Database 또는 SAML/Cognito 연동
  → Role Mapping
```

### 10-2. Fine-Grained Access Control (FGAC)

```json
// 역할 생성 예시 (OpenSearch Dashboards → Security)
{
  "cluster_permissions": ["cluster_composite_ops_ro"],
  "index_permissions": [
    {
      "index_patterns": ["logs-*"],
      "allowed_actions": ["read", "search"]
    },
    {
      "index_patterns": ["issue-history"],
      "allowed_actions": ["crud"]
    }
  ]
}
```

**역할 분리 예시**:

| 역할 | 접근 범위 | 용도 |
|------|---------|------|
| log-reader | logs-* 읽기 전용 | 개발자 로그 조회 |
| issue-admin | issue-history CRUD | Issue 관리 앱 |
| dashboard-user | 전체 읽기 | Dashboards 시각화 |
| admin | 전체 | 클러스터 관리 |

### 10-3. IAM 기반 접근 정책

```json
// Domain Access Policy (리소스 기반 정책)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/app-role"
      },
      "Action": "es:ESHttp*",
      "Resource": "arn:aws:es:ap-northeast-2:123456789:domain/my-domain/*"
    }
  ]
}
```

### 10-4. 학습 체크

- [ ] VPC 접근과 Public 접근의 트레이드오프를 설명할 수 있다
- [ ] FGAC로 인덱스별 읽기/쓰기 권한을 분리할 수 있다
- [ ] IAM 정책과 FGAC의 관계를 설명할 수 있다
- [ ] 애플리케이션별 최소 권한 역할을 설계할 수 있다

---
