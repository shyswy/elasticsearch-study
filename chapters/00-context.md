# 프로젝트 컨텍스트 & 학습 단계 개요

> 이 섹션은 학습 세션 시작 시 Claude가 프로젝트를 즉시 파악할 수 있도록 작성된 요약이다.

## 프로젝트 개요

**사용 목적**:
- 로그 수집 및 검색
- Issue History 데이터 저장 및 집계

**핵심 스택**:

| 구성요소 | 기술 | 역할 |
|---------|------|------|
| 검색/저장 엔진 | AWS OpenSearch Service | 관리형 Elasticsearch 호환 서비스 |
| 티어 | 가장 저렴한 티어 | 비용 최적화 운영 |
| 데이터 유형 | 로그 + Issue History | 시계열 + 구조화 문서 |
| 집계 | Aggregation | 통계/분석 쿼리 |

## AWS OpenSearch Service 특성

```
Elasticsearch (오픈소스)
      ↓ fork
OpenSearch (AWS 주도, Apache 2.0 라이선스)
      ↓ 관리형 서비스
AWS OpenSearch Service (구 Amazon Elasticsearch Service)
```

**Elasticsearch vs OpenSearch 관계**:
- OpenSearch는 Elasticsearch 7.10에서 fork됨
- 핵심 개념(인덱스, 매핑, 쿼리 DSL, 집계)은 동일
- 차이점: 보안(Security Plugin 기본 내장), 일부 API 경로, 플러그인 생태계

**현재 환경 제약 (저렴한 티어)**:
- 노드 수 제한 (단일 노드 또는 소수 노드)
- 스토리지 제한
- 인스턴스 타입 제한 (t3.small 등)
- → 인덱스 설계, 샤드 전략, 데이터 라이프사이클 관리가 더욱 중요

## 데이터 흐름 (예상)

```
[애플리케이션 로그]
      ↓
[로그 수집기] (Fluent Bit / Logstash / 직접 API 호출)
      ↓
[AWS OpenSearch Service]
      ↓
[OpenSearch Dashboards] ← 시각화/검색 UI
```

```
[Issue 발생]
      ↓
[애플리케이션] → Issue History 문서 저장 (Index API)
      ↓
[AWS OpenSearch Service]
      ↓
[집계 쿼리] → 통계/리포트 생성
```

---

## 학습 단계 개요

```
[ Phase 1 ] 실용 레벨 ── 현장에서 바로 쓸 수 있는 수준
  1단계: Elasticsearch 핵심 개념 & 아키텍처
  2단계: 인덱스 설계 & 매핑 (+ 매핑 심화)
  3단계: CRUD & 검색 쿼리 기초
  3.5단계: 검색 결과 제어 & 페이지네이션 ★ NEW
  4단계: Aggregation (집계)
  4.5단계: 고급 집계 패턴 ★ NEW
  5단계: AWS OpenSearch Service 운영
  6단계: 인덱스 관리 & 트러블슈팅 (+ 대량 작업)
  6.5단계: Lucene 저장 구조 심화 ★NEW (설계 전제지식)

[ Phase 2 ] 전문가 레벨 ── 큰 그림이 보이는 수준
  7단계: 검색 엔진 내부 동작 원리
  8단계: 고급 쿼리 & 분석 (+ 실전 쿼리 최적화 패턴)
  9단계: 성능 최적화
  10단계: 보안 & 접근 제어
  11단계: 데이터 파이프라인 & 수집
  12단계: 모니터링 & 운영 자동화
  13단계: 대규모 운영 패턴 & 아키텍처

[ Design Phase ] 설계 실전 ── 직접 만들 수 있는 수준
  D0단계: 데이터 모델링 의사결정 프레임워크 ★NEW
  D1단계: 샤드 사이징 & 용량 계획 (이론)
  D1.5단계: 인덱스 설정 & 스토리지 전략 ★NEW
  D2단계: Data Stream & Rollover 설계 (이론)
  D3단계: ISM 정책 설계 (이론)
  D4단계: 템플릿 구성 전략 (이론)
  D5단계: Alias 전략 & 마이그레이션 (이론)
  D6단계: 인덱스 매핑 설계 실습
  D7단계: Data Stream 설계 실습
  D8단계: ISM 정책 설계 실습
  D9단계: 통합 설계 실습 (end-to-end)

[ Phase 3 ] 아키텍트 레벨 ── 어떤 요구사항이든 최적 설계를 낼 수 있는 수준
  A1단계: 데이터 모델링 의사결정 프레임워크
  A2단계: 멀티테넌시 & Routing 고급 설계
  A3단계: 쿼리 성능 프로파일링 & 튜닝 실전
  A4단계: 환경별 인프라 사이징 & 비용 최적화
  A5단계: 무중단 스키마 진화 & 마이그레이션 실전
  A6단계: OpenSearch 고유 기능 심화 활용
  A7단계: 애플리케이션 통합 패턴 & SDK 설계
  A8단계: 종합 설계 의사결정 워크숍
```
