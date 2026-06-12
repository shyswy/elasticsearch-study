# Elasticsearch / OpenSearch 학습 계획서

> **목적**: 이 문서는 AWS OpenSearch Service를 활용한 로그 수집 및 Issue History 데이터 저장·집계 시스템을
> 담당 개발자로서 자신 있게 운영하고, Elasticsearch 전반 + OpenSearch 특화 지식을 쌓기 위한 단계별 학습 로드맵이다.
>
> **사용법**: 이 파일을 Claude/Kiro와의 학습 세션에서 로드하면, 프로젝트 맥락과 학습 진도를 바탕으로
> 이어서 학습할 수 있다. 완료한 항목은 `[x]`로 표시하고 저장하면 된다.
>
> **구조**: 각 단계의 상세 내용은 `chapters/` 디렉토리의 개별 파일에 있다.
> 이 파일은 목차 + 진도 관리 + 학습 세션 가이드 역할을 한다.

---

## 챕터 파일 매핑

| 단계 | 파일 경로 |
|------|----------|
| 컨텍스트 & 개요 | [chapters/00-context.md](chapters/00-context.md) |
| **Phase 1** | |
| 1단계 | [chapters/phase1/01-core-concepts.md](chapters/phase1/01-core-concepts.md) |
| 2단계 | [chapters/phase1/02-index-design-mapping.md](chapters/phase1/02-index-design-mapping.md) |
| 3단계 | [chapters/phase1/03-crud-search-queries.md](chapters/phase1/03-crud-search-queries.md) |
| 3.5단계 | [chapters/phase1/03.5-search-result-control.md](chapters/phase1/03.5-search-result-control.md) |
| 4단계 | [chapters/phase1/04-aggregation.md](chapters/phase1/04-aggregation.md) |
| 4.5단계 | [chapters/phase1/04.5-advanced-aggregation-patterns.md](chapters/phase1/04.5-advanced-aggregation-patterns.md) |
| 5단계 | [chapters/phase1/05-opensearch-service-ops.md](chapters/phase1/05-opensearch-service-ops.md) |
| 6단계 | [chapters/phase1/06-index-management-troubleshooting.md](chapters/phase1/06-index-management-troubleshooting.md) |
| 6.5단계 ★NEW | [chapters/phase1/06.5-lucene-storage-internals.md](chapters/phase1/06.5-lucene-storage-internals.md) |
| **Phase 2** | |
| 7단계 | [chapters/phase2/07-search-engine-internals.md](chapters/phase2/07-search-engine-internals.md) |
| 8단계 | [chapters/phase2/08-advanced-queries.md](chapters/phase2/08-advanced-queries.md) |
| 9단계 | [chapters/phase2/09-performance-optimization.md](chapters/phase2/09-performance-optimization.md) |
| 10단계 | [chapters/phase2/10-security-access-control.md](chapters/phase2/10-security-access-control.md) |
| 11단계 | [chapters/phase2/11-data-pipeline-ingestion.md](chapters/phase2/11-data-pipeline-ingestion.md) |
| 12단계 | [chapters/phase2/12-monitoring-automation.md](chapters/phase2/12-monitoring-automation.md) |
| 13단계 | [chapters/phase2/13-architecture-patterns.md](chapters/phase2/13-architecture-patterns.md) |
| **Design Phase** | |
| D0단계 ★NEW | [chapters/design/D0-data-modeling-decisions.md](chapters/design/D0-data-modeling-decisions.md) |
| D1단계 | [chapters/design/D1-shard-sizing.md](chapters/design/D1-shard-sizing.md) |
| D1.5단계 ★NEW | [chapters/design/D1.5-index-settings-storage-strategy.md](chapters/design/D1.5-index-settings-storage-strategy.md) |
| D2단계 | [chapters/design/D2-data-stream-rollover.md](chapters/design/D2-data-stream-rollover.md) |
| D3단계 | [chapters/design/D3-ism-policy.md](chapters/design/D3-ism-policy.md) |
| D4단계 | [chapters/design/D4-template-strategy.md](chapters/design/D4-template-strategy.md) |
| D5단계 | [chapters/design/D5-alias-migration.md](chapters/design/D5-alias-migration.md) |
| D6단계 | [chapters/design/D6-mapping-practice.md](chapters/design/D6-mapping-practice.md) |
| D7단계 | [chapters/design/D7-data-stream-practice.md](chapters/design/D7-data-stream-practice.md) |
| D8단계 | [chapters/design/D8-ism-practice.md](chapters/design/D8-ism-practice.md) |
| D9단계 | [chapters/design/D9-e2e-practice.md](chapters/design/D9-e2e-practice.md) |

---

## 현재 학습 진도

> 아래에 완료 날짜를 기록하세요

### Phase 1: 실용 레벨

> **목표**: 현재 OpenSearch 클러스터를 혼자 운영·수정할 수 있고,
> 인덱스 설계부터 쿼리 작성, 장애 대응까지 수행할 수 있는 수준

- [x] 1단계: Elasticsearch 핵심 개념 & 아키텍처 — 완료일: 2026-05-06
- [x] 2단계: 인덱스 설계 & 매핑 (+ 매핑 심화) — 완료일: 2026-05-08
- [x] 3단계: CRUD & 검색 쿼리 기초 — 완료일: 2026-05-08
- [x] 3.5단계: 검색 결과 제어 & 페이지네이션 — 완료일: 2026-06-09
- [x] 4단계: Aggregation (집계) — 완료일: 2026-06-09
- [x] 4.5단계: 고급 집계 패턴 — 완료일: 2026-06-09
- [ ] 5단계: AWS OpenSearch Service 운영 — 완료일: ______
- [ ] 6단계: 인덱스 관리 & 트러블슈팅 (+ 대량 작업) — 완료일: ______
- [ ] 6.5단계: Lucene 저장 구조 심화 ★NEW — 완료일: ______

**Phase 1 최종 체크리스트**:
- [ ] Elasticsearch의 핵심 개념(역색인, 샤드, 매핑)을 설명할 수 있다
- [ ] 로그와 Issue History에 적합한 인덱스를 설계할 수 있다
- [ ] bool 쿼리로 복합 조건 검색을 작성할 수 있다
- [ ] collapse + inner_hits로 그룹별 결과를 조회할 수 있다
- [ ] search_after로 딥 페이지네이션을 구현할 수 있다
- [ ] Aggregation으로 통계/분석 데이터를 추출할 수 있다
- [ ] top_hits, composite, bucket_selector 등 고급 집계를 활용할 수 있다
- [ ] AWS OpenSearch Service의 ISM 정책으로 인덱스 수명주기를 관리할 수 있다
- [ ] 클러스터 장애 시 원인을 진단하고 해결할 수 있다
- [ ] _update_by_query / _delete_by_query로 대량 작업을 수행할 수 있다
- [ ] 저렴한 티어에서의 제약을 이해하고 최적화 전략을 적용할 수 있다
- [ ] Lucene 내부 저장 구조(역색인, Doc Values, BKD, _source)를 원리적으로 설명할 수 있다
- [ ] 매핑 옵션이 물리 파일 생성에 미치는 영향을 설명하고 최적화 결정을 내릴 수 있다

### Phase 2: 전문가 레벨

> **목표**: "왜 이렇게 설계됐는가"를 이해하고, 새로운 상황에서 올바른 선택을 할 수 있으며,
> 다른 팀원에게 설명해줄 수 있는 수준

- [ ] 7단계: 검색 엔진 내부 동작 원리 — 완료일: ______
- [ ] 8단계: 고급 쿼리 & 분석 — 완료일: ______
- [ ] 9단계: 성능 최적화 — 완료일: ______
- [ ] 10단계: 보안 & 접근 제어 — 완료일: ______
- [ ] 11단계: 데이터 파이프라인 & 수집 — 완료일: ______
- [ ] 12단계: 모니터링 & 운영 자동화 — 완료일: ______
- [ ] 13단계: 대규모 운영 패턴 & 아키텍처 — 완료일: ______

**Phase 2 최종 체크리스트**:
- [ ] 문서 색인 → 검색 가능까지의 내부 흐름을 각 컴포넌트 역할과 함께 설명할 수 있다
- [ ] 복잡한 검색 요구사항을 적절한 쿼리 조합으로 해결할 수 있다
- [ ] 저렴한 티어에서 성능 병목을 찾고 최적화할 수 있다
- [ ] FGAC로 팀/역할별 접근 제어를 설계할 수 있다
- [ ] 데이터 수집 파이프라인을 설계하고 Ingest Pipeline을 활용할 수 있다
- [ ] 모니터링 + 알림으로 장애를 사전에 감지하는 체계를 구축할 수 있다
- [ ] 데이터 증가에 대응하는 인덱스 전략과 스케일링 계획을 수립할 수 있다

### Design Phase: 설계 실전

> **목표**: 요구사항을 받으면 인덱스 구조, Data Stream, ISM 정책, 템플릿을
> 처음부터 설계하고 생성할 수 있는 수준.
>
> **전제 조건**: Phase 1의 1~6단계 완료 후 진행 권장

- [ ] D0단계: 데이터 모델링 의사결정 프레임워크 ★NEW — 완료일: ______
- [ ] D1단계: 샤드 사이징 & 용량 계획 — 완료일: ______
- [ ] D1.5단계: 인덱스 설정 & 스토리지 전략 ★NEW — 완료일: ______
- [ ] D2단계: Data Stream & Rollover 설계 — 완료일: ______
- [ ] D3단계: ISM 정책 설계 — 완료일: ______
- [ ] D4단계: 템플릿 구성 전략 — 완료일: ______
- [ ] D5단계: Alias 전략 & 마이그레이션 — 완료일: ______
- [ ] D6단계: 인덱스 매핑 설계 실습 — 완료일: ______
- [ ] D7단계: Data Stream 설계 실습 — 완료일: ______
- [ ] D8단계: ISM 정책 설계 실습 — 완료일: ______
- [ ] D9단계: 통합 설계 실습 (end-to-end) — 완료일: ______

**Design Phase 최종 체크리스트**:
- [ ] 주어진 데이터/쿼리 패턴에서 최적 문서 구조(Object/Nested/Flattened/비정규화)를 결정할 수 있다
- [ ] Access Pattern에서 필드별 최적 타입과 옵션(index/doc_values)을 역산할 수 있다
- [ ] 데이터 규모에 맞는 샤드 수와 인덱스 분할 주기를 산출할 수 있다
- [ ] 인덱스 설정(refresh/translog/merge/codec)을 데이터 특성에 맞게 튜닝할 수 있다
- [ ] Hot-Warm-Cold 계층을 비용 시뮬레이션 기반으로 설계할 수 있다
- [ ] Data Stream의 rollover 조건을 적절히 설계할 수 있다
- [ ] 다단계 ISM 정책을 상태 머신으로 설계하고 JSON으로 작성할 수 있다
- [ ] Component Template + Index Template 계층으로 재사용 가능한 구조를 설계할 수 있다
- [ ] Alias 기반 무중단 마이그레이션을 계획할 수 있다
- [ ] 새로운 데이터 저장 요구사항에 대해 end-to-end 인덱스 전략을 설계할 수 있다
- [ ] 설계 결정의 근거를 명확히 설명할 수 있다

### Phase 3: 아키텍트 레벨 (TODO)

> 어떤 요구사항이든 최적 설계를 낼 수 있는 수준
> (상세 내용은 Phase 2 완료 후 추가 예정)

- [ ] A1단계: 데이터 모델링 의사결정 프레임워크
- [ ] A2단계: 멀티테넌시 & Routing 고급 설계
- [ ] A3단계: 쿼리 성능 프로파일링 & 튜닝 실전
- [ ] A4단계: 환경별 인프라 사이징 & 비용 최적화
- [ ] A5단계: 무중단 스키마 진화 & 마이그레이션 실전
- [ ] A6단계: OpenSearch 고유 기능 심화 활용
- [ ] A7단계: 애플리케이션 통합 패턴 & SDK 설계
- [ ] A8단계: 종합 설계 의사결정 워크숍

---

## 다음 학습 단계 자동 감지

> AI가 이 파일을 읽으면, 위 진도에서 첫 번째 미완료(`[ ]`) 단계를 찾아서
> 해당 챕터 파일을 로드하고 학습 세션을 시작한다.
>
> **현재 다음 학습 대상**: 5단계 (chapters/phase1/05-opensearch-service-ops.md)
>
> **권장 학습 경로** (설계 역할 최적화):
> ```
> Phase1: 5 → 6 → 6.5(저장 구조 심화)
>    ↓
> Design: D0(모델링 의사결정) → D1(샤드 사이징) → D1.5(설정 & 스토리지)
>    → D2~D5(이론) → D6(매핑 실습, 4개 시나리오) → D7~D9(실습)
>    ↓
> Phase2: 7~13 (설계와 병행 가능, 필요 시 참조)
> ```

---

## 학습 문서 시스템

> 매 학습 세션이 끝날 때마다 해당 단계의 내용을 `docs/` 하위 MD 파일로 저장한다.
> 생성된 문서들은 복습 자료로 활용한다.

### 디렉토리 구조

```
docs/
├── phase1-practical/                  ← Phase 1 학습 노트
│   ├── 01-es-core-concepts.md
│   ├── 02-index-design-mapping.md
│   ├── 03-crud-search-queries.md
│   ├── 03.5-search-result-control.md
│   ├── 04-aggregation.md
│   ├── 04.5-advanced-aggregation-patterns.md
│   ├── 05-opensearch-service-ops.md
│   └── 06-index-management-troubleshooting.md
└── phase2-expert/                     ← Phase 2 학습 노트
    ├── 07-search-engine-internals.md
    ├── 08-advanced-queries.md
    ├── 09-performance-optimization.md
    ├── 10-security-access-control.md
    ├── 11-data-pipeline-ingestion.md
    ├── 12-monitoring-automation.md
    └── 13-architecture-patterns.md
```

### 학습 문서 작성 규칙

각 챕터 파일의 구성:

```
# [챕터 제목]

## 학습 일자
YYYY-MM-DD

## 핵심 개념 정리
배운 개념들을 내 언어로 재서술.
학습 세션에서 AI가 설명한 상세 내용(정의, 왜 존재하는가, 내부 동작, 프로젝트 적용)을 전부 포함.
추가 질문에서 다룬 심화 내용도 별도 섹션으로 정리.
STUDY_PLAN.md의 표/다이어그램을 복붙하는 것이 아니라, 세션에서 깊이 있게 풀어낸 설명을 기록.
과도하게 축약하지 않고, 이 문서만 읽으면 해당 단계를 다시 학습하지 않아도 될 정도의 상세함 유지.

## 현재 프로젝트 연결
이 개념이 프로젝트(로그 수집, Issue History)의 어느 부분과 연결되는지

## 실전 쿼리/설정 예시
학습에서 다룬 쿼리, 매핑, 설정을 실제 프로젝트 데이터 기준으로 작성.

## 핵심 명령어 / API
실습에서 쓴 REST API, curl 명령어 모음

## 헷갈렸던 점 & 해결
학습 중 막혔던 부분과 이해한 방식.
형식: "착각: ... → 실제: ... → 이유: ..."

## 복습 포인트
"이것만 보면 전체 내용이 떠오르는" 수준으로 상세하게 작성.
```

### 학습 문서 생성 시점

- 해당 단계의 **학습 체크** 항목을 70% 이상 통과했을 때 AI가 생성할지 유저에게 질문, 혹은 유저가 직접 요청.
- 요청 방법: `"이 단계 학습 내용을 docs/ 에 정리해줘"`

---

## 학습 세션 가이드

> **Claude와 함께 학습하는 방법**

### 세션 시작 방법

학습 세션 시작 시 Claude에게:
```
STUDY_PLAN.md를 기반으로 학습 세션을 시작할게.
현재 [단계명]까지 완료했고, [다음 단계]부터 학습하고 싶어.
```

또는 간단히:
```
STUDY_PLAN.md 기반으로 다음 단계 학습 시작해줘.
```
→ AI가 진도를 확인하고 다음 미완료 단계의 챕터 파일을 로드하여 학습 시작.

### 추가 질문 대응 규칙 (AI 필수 준수)

학습 중 유저가 현재 단계 범위를 넘어서는 추가 질문을 할 경우:

1. **해당 내용이 뒤 단계에 있는지 확인**한다.
2. 뒤 단계에 있다면 → **"이 내용은 N단계에서 다룹니다"**라고 안내하고, 현재 단계에서 알아야 할 최소한의 요약만 제공한다.
3. **해당 단계의 챕터 파일에 메모를 추가**한다.
   - 형식: `> **N단계에서 이어지는 질문**: [질문 요약] — [다룰 내용 설명]`
   - 이렇게 하면 해당 단계 학습 시 반드시 이 질문을 다루게 된다.
4. 메모 추가 후 현재 단계 학습을 이어간다.

### 개념 학습 세션 깊이 기준 (AI 필수 준수)

> **핵심 원칙**: 챕터 파일에 적힌 내용을 그대로 읽어주는 것은 학습이 아니다.
> 각 개념을 "왜 존재하는가 → 정확한 정의 → 내부 동작 → 프로젝트 적용"의 흐름으로
> 깊이 있게 설명해야 한다. 유저가 체크 세션에서 자신 있게 답할 수 있을 정도의 깊이가 기준.

**개념 학습 시 반드시 포함해야 하는 4단계**:

```
1. 정확한 정의 (Definition)
   - 해당 개념이 정확히 무엇인지 한 문장으로 정의
   - 모호한 비유가 아닌, 기술적으로 정확한 설명

2. 왜 존재하는가 (Problem → Solution)
   - 이 개념이 없으면 어떤 문제가 생기는지
   - 이 개념이 그 문제를 어떻게 해결하는지

3. 내부 동작 원리 (How it works)
   - 표면적 설명이 아닌, 실제로 어떻게 동작하는지
   - 다른 기술(RDB 등)과의 구체적 비교

4. 현재 프로젝트 맥락 적용 (Context)
   - 이 개념이 현재 프로젝트(로그 수집, Issue History, 저렴한 티어)에서
     구체적으로 어떻게 적용되는지
```

**금지 사항**:
- 챕터 파일의 표/다이어그램을 그대로 복붙하고 "이렇습니다"로 끝내기
- 정의 없이 비유만으로 설명하기
- 한 개념당 2~3줄로 축약하기

**깊이 판단 기준**:
- 유저가 이 설명만 듣고 체크 세션의 질문에 자신 있게 답할 수 있는가?
- "왜?"라는 후속 질문이 나오지 않을 정도로 근거가 충분한가?

---

### 학습 세션 유형

**개념 학습 세션**: 특정 단계의 이론 학습
```
예: "1단계의 역색인에 대해 예시를 들어 설명해줘"
```

**실습 세션**: 직접 쿼리/설정 작성
```
예: "Issue History 인덱스 매핑을 직접 작성하는 걸 도와줘"
```

**질문 세션**: 특정 개념의 깊이 있는 이해
```
예: "왜 text 필드에 집계를 하면 안 되는가?"
```

**체크 세션**: 학습 내용 검증
```
예: "이 단계의 학습 체크 항목들을 나에게 질문해줘. 내가 답하면 맞는지 피드백해줘."
질문 개수는 해당 단계의 내용 범위와 난이도에 따라 유동적으로 출제.
체크 질문은 한 번에 모두 던진다.
```

**문서화 세션**: 학습 내용을 docs/에 저장
```
예: "이 단계 학습 내용을 docs/에 정리해줘"
```

---

## 참고 자료

### 공식 문서
- [OpenSearch 공식 문서](https://opensearch.org/docs/latest/)
- [AWS OpenSearch Service 공식 문서](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/)
- [Elasticsearch 공식 문서 (7.10 — OpenSearch 호환 버전)](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/index.html)
- [OpenSearch Dashboards](https://opensearch.org/docs/latest/dashboards/)

### 학습 자료
- [OpenSearch Playground (무료 체험)](https://playground.opensearch.org/)
- [AWS OpenSearch Workshop](https://catalog.workshops.aws/opensearch/en-US)
- [Elasticsearch: The Definitive Guide (무료 온라인)](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)

### Query DSL 레퍼런스
- [OpenSearch Query DSL](https://opensearch.org/docs/latest/query-dsl/)
- [OpenSearch Aggregations](https://opensearch.org/docs/latest/aggregations/)
- [OpenSearch REST API](https://opensearch.org/docs/latest/api-reference/)

### 현재 프로젝트 관련
- 서비스: AWS OpenSearch Service (저렴한 티어)
- 용도: 로그 수집 + Issue History 저장/집계
- 리전: ap-northeast-2 (서울)
