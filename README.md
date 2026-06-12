# Elasticsearch / OpenSearch Study

AWS OpenSearch Service를 활용한 로그 수집 및 Issue History 데이터 저장·집계 시스템을 위한 학습 레포지토리.

## 목적

- Elasticsearch 전반 + OpenSearch 특화 지식 체계적 학습
- 실무 프로젝트(AWS OpenSearch, 로그 수집, Issue History) 운영 능력 확보
- 인덱스 설계부터 쿼리 작성, 장애 대응, 아키텍처 설계까지 커버

## 학습 구조

```
Phase 1 (실용 레벨)
 └── 핵심 개념 → 인덱스 설계 → 검색 쿼리 → Aggregation → 운영 → 트러블슈팅

Phase 2 (전문가 레벨)
 └── 내부 동작 → 고급 쿼리 → 성능 최적화 → 보안 → 파이프라인 → 모니터링 → 아키텍처

Design Phase (설계 실전)
 └── 데이터 모델링 → 샤드 사이징 → Data Stream → ISM → 템플릿 → E2E 설계
```

## 디렉토리 구조

```
├── STUDY_PLAN.md              # 학습 로드맵 & 진도 관리
├── chapters/                  # 챕터별 학습 커리큘럼
│   ├── phase1/                # Phase 1: 실용 레벨 (9 챕터)
│   ├── phase2/                # Phase 2: 전문가 레벨 (7 챕터)
│   └── design/                # Design Phase: 설계 실전 (10 챕터)
├── docs/                      # 학습 완료 후 정리 노트
│   ├── phase1-practical/
│   └── phase2-expert/
├── design/                    # 실무 설계 문서
│   ├── issue/                 # Issue History 아키텍처 설계
│   ├── tasks/                 # 구현 태스크 설계
│   └── cross-pizza-specs/     # 크로스 도메인 API 스펙
└── docker-compose.yml         # 로컬 ES 클러스터 (실습용)
```

## 현재 진도

- **Phase 1**: 4.5단계 완료 (고급 집계 패턴까지)
- **Phase 2**: 미시작
- **Design Phase**: 미시작

## 기술 스택

- AWS OpenSearch Service (ap-northeast-2)
- Elasticsearch 7.10 호환
- OpenSearch Dashboards

## 학습 방법

`STUDY_PLAN.md`를 AI 학습 세션에서 로드하면, 진도를 기반으로 이어서 학습할 수 있도록 설계됨.
