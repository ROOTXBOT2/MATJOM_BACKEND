# MATJOM Repository Notes

> 이 문서는 MATJOM 프로젝트의 저장소 구조와 문서 기준을 설명합니다.  
> 공식 팀 저장소와 개인 작업 저장소가 나뉘어 있어, 코드 검토 시 참고할 기준을 명확히 남기기 위한 문서입니다.

## 1. 저장소 기준

| 구분 | 내용 |
|---|---|
| 공식 팀 저장소 | `MATJOM/BACKEND` |
| 공식 기준 브랜치 | `develop` |
| 개인 작업 저장소 | `ROOTXBOT2/MATJOM_BACKEND` |
| 구현 기준 브랜치 | `feature/7-30-search-session-lifecycle` |
| 문서 브랜치 | `docs/matjom-portfolio` |

공식 팀 저장소는 프로젝트의 공용 기준 저장소입니다.  
개인 작업 저장소에는 제가 담당한 백엔드 기능 구현, 테스트 코드, 문서화 작업이 남아 있습니다.

## 2. 문서 작성 기준

`PORTFOLIO.md`와 README는 실제 구현 범위가 가장 잘 남아 있는 `feature/7-30-search-session-lifecycle` 브랜치를 기준으로 작성했습니다.

이 브랜치에는 다음 기능이 함께 포함되어 있습니다.

- PostGIS 기반 장소 검색
- 거리 정렬 커서 페이징
- Redis 검색 캐싱
- Idempotency-Key 기반 룰렛 추천
- 방문 세션 lifecycle
- 지오펜스 도착 판정
- 세션 만료 스케줄러
- RateLimit / JWT / Idempotency 공통 기반
- 단위 테스트 및 Testcontainers 기반 통합 테스트

## 3. 주요 브랜치 역할

### `MATJOM/BACKEND develop`

공식 팀 저장소의 기준 브랜치입니다.

확인된 성격:

- 프로젝트 기본 구조
- Spring Boot 실행 환경
- 공통 응답/예외 규약
- 인프라 연결 테스트 기록

### `ROOTXBOT2/MATJOM_BACKEND main`

개인 작업 저장소의 기본 브랜치입니다.

확인된 성격:

- 공식 저장소 초기 구조와 유사한 기반
- 주요 기능 구현은 feature 브랜치에서 진행

### `ROOTXBOT2/MATJOM_BACKEND feature/7-search-cursor-paging`

장소 검색과 룰렛 추천 중심의 기능 브랜치입니다.

확인된 성격:

- PostGIS 기반 장소 검색
- `distance:id` 커서 페이징
- Redis 60초 검색 캐싱
- 룰렛 추천 API
- RateLimit / Idempotency 기반 일부

### `ROOTXBOT2/MATJOM_BACKEND feature/30-session-lifecycle`

방문 세션 lifecycle 중심의 기능 브랜치입니다.

확인된 성격:

- 방문 세션 생성
- 위치 이벤트 기록
- 지오펜스 도착 판정
- 수동 도착 확정
- 세션 만료 스케줄러

### `ROOTXBOT2/MATJOM_BACKEND feature/7-30-search-session-lifecycle`

검색/추천 기능과 방문 세션 기능을 함께 확인할 수 있는 통합 구현 브랜치입니다.

확인된 성격:

- 검색/추천 기능과 방문 세션 기능을 함께 포함
- 단위 테스트와 통합 테스트 포함
- 포트폴리오 문서의 구현 기준 브랜치

## 4. 구현 근거가 있는 핵심 기능

- PostGIS 기반 반경 검색
- 거리 ASC + `place_id` ASC 안정 정렬
- `distance:id` 커서 페이징
- Redis 60초 검색 캐싱
- 결과 수 기반 `meta.reason` 응답
- Idempotency-Key 기반 룰렛 추천
- seed 기반 재현 가능한 추천
- 사용자당 ACTIVE 방문 세션 1개 제한
- 방문 위치 이벤트 기록
- 수동 도착 확정
- 30m / 180초 dwell / 10초 grace 지오펜스 판정
- 세션 만료 스케줄러
- 검색 API RateLimit
- JWT 기반 Access/Refresh Token
- Testcontainers 기반 점심 여정 통합 테스트

## 5. 범위와 구현 기준

이 프로젝트의 포트폴리오 문서는 실제 구현된 기능을 중심으로 작성했습니다.

문서에서 사용하는 표현 기준은 다음과 같습니다.

| 범위 | 문서화 기준 |
|---|---|
| 위치 검색 | PostGIS `ST_DWithin`, `ST_Distance`, 커서 페이징 코드 기준 |
| 추천 | 룰렛 방식 추천, Idempotency-Key, seed 기반 재현성 기준 |
| 방문 세션 | ACTIVE 세션 제한, 위치 이벤트, 수동 도착, 만료 스케줄러 기준 |
| 지오펜스 | 30m, 180초 dwell, 10초 grace window 구현 기준 |
| 인증 | JWT 기반 Access/Refresh Token 구조 기준 |
| 테스트 | 단위 테스트와 Testcontainers 통합 테스트 기준 |

## 6. 향후 개선 가능 항목

프로젝트를 운영 서비스 수준으로 확장한다면 아래 항목을 추가로 개선할 수 있습니다.

- 공식 팀 저장소와 개인 구현 브랜치 병합 이력 정리
- JWT 키 관리 방식 고도화
- 검색 캐시 키 버킷팅으로 캐시 효율 개선
- RateLimit 기준을 SecurityContext 사용자 정보와 더 명확히 연결
- 운영 관측성 대시보드와 알림 정책 추가
- Flyway 기반 DB migration 정리

## 7. 검토 순서

코드 검토 시 아래 순서로 보면 구현 의도를 빠르게 확인할 수 있습니다.

1. `PORTFOLIO.md`
2. `src/main/java/com/matjom/matjom/place/service/PlaceSearchService.java`
3. `src/main/java/com/matjom/matjom/place/repository/PlaceRepository.java`
4. `src/main/java/com/matjom/matjom/recommendation/service/RouletteService.java`
5. `src/main/java/com/matjom/matjom/visit/service/VisitSessionService.java`
6. `src/main/java/com/matjom/matjom/visit/geofence/DefaultGeoFenceEvaluator.java`
7. `src/test/java/com/matjom/matjom/integration/LunchJourneyIntegrationTest.java`
