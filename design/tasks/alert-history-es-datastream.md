# Task: Alert History — PG → ES Datastream 전환

> 이 문서를 새 세션에 전달하면 바로 작업 시작 가능하도록 모든 컨텍스트를 포함합니다.

---

## 목표

alert_history 저장소를 PostgreSQL에서 Elasticsearch Datastream으로 전환.
1 alert 처리 = 1 ES document로 통합 저장. 채널별 상세 데이터(email html body, snmp oid 등) 포함.

---

## 의사결정 기록

| 항목 | 결정 | 근거 |
|------|------|------|
| 저장소 | ES Datastream | 시계열 데이터 특성, 자동 rollover, ILM 적용 용이 |
| Document 구조 | 1 alert = 1 document (통합) | 현재 코드가 Promise.allSettled 후 한 번에 저장하는 구조. 조회 시 한 건으로 전체 파악 |
| Email HTML body | 포함 | 나중에 ES 부담되면 축약. 현재는 전체 저장 |
| SNMP OID/varbinds | 포함 | 가벼움 (수백 bytes) |
| Custom endpoint body | 포함 | 가벼움 (수백 bytes) |
| 설정 on/off | 불필요 | 운영 중 변경할 이유 없음. 불필요한 복잡도 제거 |

---

## ES Document 구조

```typescript
interface IAlertHistoryDocument {
  // === 공통 ===
  '@timestamp': string;              // ISO 8601, datastream 필수
  incident_id: string;
  workspace_id: string;
  event_type: 'new' | 'resolved';
  incident_count: number;            // frequency control로 묶인 건수 (resolved는 1)

  // === 대표 incident 정보 ===
  device_id: string;
  issue_code: string;
  severity?: string;
  issue_type?: string;

  // === 채널 결과 ===
  channels: {
    email: {
      status: 'success' | 'failed' | 'skipped';
      recipients?: string[];
      subject?: string;
      html_body?: string;
      error_message?: string;
    };
    snmp: {
      status: 'success' | 'failed' | 'skipped';
      servers?: { ip: string; port: number }[];
      varbinds?: { oid: string; value: string }[];
      error_message?: string;
    };
    custom_endpoint: {
      status: 'success' | 'failed' | 'skipped';
      url?: string;
      request_body?: Record<string, unknown>;
      response_status?: number;
      error_message?: string;
    };
  };
}
```

---

## ES Datastream 설정

- Index pattern: `alert-history`
- Datastream name: `alert-history`
- ILM Policy: 추후 결정 (hot → warm → delete)
- Mapping: dynamic template + explicit fields for `channels.*`

---

## 영향 범위

### 변경 대상

| 파일 | 변경 내용 |
|------|-----------|
| `src/alert/repositories/alert-history.repository.ts` | PG → ES client로 전환. `save()`, `saveAll()` → ES bulk/index |
| `src/alert/interfaces/alert-history.interface.ts` | `IAlertHistory` → `IAlertHistoryDocument` 구조 변경 |
| `src/alert/interfaces/channel-result.interface.ts` | 채널별 상세 데이터 필드 추가 |
| `src/alert/services/alert-orchestration.service.ts` | 채널 결과에 상세 데이터 포함하도록 수정 |
| `src/alert/services/email-alert.service.ts` | 반환값에 recipients, subject, html_body 추가 |
| `src/alert/services/snmp-trap.service.ts` | 반환값에 servers, varbinds 추가 |
| `src/alert/services/custom-endpoint-alert.service.ts` | 반환값에 url, request_body 추가 |

### 신규 생성

| 파일 | 내용 |
|------|------|
| `src/alert/opensearch/alert-history-setup.ts` | Datastream index template + ILM 생성 |
| `src/alert/opensearch/alert-history-client.ts` | ES client wrapper (index document) |

### 로컬 테스트 변경

| 파일 | 변경 내용 |
|------|-----------|
| `scripts/local-test/001-create-tables.sql` | `alert_history` 테이블 제거 (또는 유지 — 다른 용도 있으면) |
| E2E 테스트 | alert_history 검증을 PG query → ES query로 변경 |

---

## 구현 순서

1. `IChannelResult` 인터페이스 확장 — 채널별 상세 데이터 필드 추가
2. 각 채널 서비스 수정 — 반환값에 상세 데이터 포함
3. ES client + datastream setup 구현
4. `AlertHistoryRepository` → ES 기반으로 재작성
5. `AlertOrchestrationService` — 새 repository 연동
6. 로컬 테스트 환경 업데이트 (ES datastream 생성, E2E 검증 변경)
7. 단위 테스트 업데이트

---

## 참조 파일

| 파일 | 용도 |
|------|------|
| `src/alert/services/alert-orchestration.service.ts` | 현재 오케스트레이션 흐름 |
| `src/alert/repositories/alert-history.repository.ts` | 현재 PG 저장 로직 (교체 대상) |
| `src/alert/services/email-alert.service.ts` | email 채널 — html_body 생성 위치 |
| `src/alert/services/snmp-trap.service.ts` | snmp 채널 — varbinds 생성 위치 |
| `src/alert/services/custom-endpoint-alert.service.ts` | custom endpoint 채널 |
| `src/threshold/opensearch/index-setup.ts` | 기존 ES index setup 패턴 참조 |
| `scripts/local-test/README.md` | 로컬 테스트 환경 |
| `scripts/local-test/.env.local` | ES 접속 정보 (localhost:9200) |

---

## 로컬 테스트 방법

```bash
# 1. 인프라 (ES 포함)
npm run local:infra

# 2. alert 모듈 실행
PORT=9094 BC_MODULE_NAME=alert npx tsx --env-file=scripts/local-test/.env.local src/alert/index.local.ts

# 3. incident 발행 → alert 처리 → ES 확인
npm run local:publish -- report dev-001 TEM-01
./scripts/local-test/infra/es-query.sh alert-history
```

---

## E2E 테스트 마이그레이션 가이드

E2E 테스트에서 `alert_history` PG 조회를 ES 조회로 변경해야 합니다.

### 변경 패턴

**Before (PG):**
```typescript
const { rows } = await db.query("SELECT * FROM alert_history WHERE workspace_id='prop-001'");
```

**After (ES):**
```typescript
await esRefresh('alert-history');
const result = await esQuery('alert-history', {
  query: { term: { workspace_id: 'prop-001' } },
  size: 50,
  sort: [{ '@timestamp': 'desc' }],
});
const docs = result?.hits?.hits?.map((h: any) => h._source) || [];
```

### 영향 파일

| 파일 | 변경 내용 |
|------|-----------|
| `scripts/local-test/e2e-test.ts` | PG alert_history 조회 → ES alert-history 조회 |
| `scripts/local-test/e2e-alert-test.ts` | `queryAlertHistory()` 함수 ES로 교체 |
| `scripts/local-test/e2e-batch-test.ts` | Step 4 alert_history 검증 ES로 교체 |
| `scripts/local-test/publish-issues.ts` | status 명령의 alert_history 출력 ES로 교체 |
| `scripts/local-test/setup-db.ts` | alert_history 테이블 검증 제거 |
| `scripts/local-test/001-create-tables.sql` | alert_history CREATE TABLE 제거 (또는 주석 처리) |

### 검증 필드 매핑

| PG 컬럼 | ES 필드 |
|---------|---------|
| `incident_id` | `incident_id` |
| `workspace_id` | `workspace_id` |
| `channel` | `channels.email.status` / `channels.snmp.status` / `channels.custom_endpoint.status` |
| `status` | 각 채널의 status 필드 |
| `sent_at` | `@timestamp` |
| `error_message` | 각 채널의 error_message 필드 |

### 주의사항

- ES datastream은 `_delete_by_query`로 정리 가능 (테스트 초기화 시)
- `esRefresh('alert-history')` 호출 후 조회해야 최신 데이터 반영
- 1 alert = 1 document이므로, 기존 "3 records (email + snmp + custom)" 검증은 "1 document with 3 channel statuses" 검증으로 변경
