# MATJOM BACKEND

MATJOM은 직장인의 반복적인 점심 의사결정 문제를 줄이기 위한 위치 기반 점심 추천 백엔드 프로젝트입니다.

공식 팀 저장소는 `MATJOM/BACKEND`이며, 이 저장소는 제가 담당한 백엔드 기능 구현과 검증 내용을 정리한 개인 작업 저장소입니다. 문서는 위치 기반 검색, 룰렛 추천, 방문 세션, 지오펜스 도착 판정 구현이 포함된 `feature/7-30-search-session-lifecycle` 브랜치를 기준으로 작성했습니다.

## 문서

- [PORTFOLIO.md](./PORTFOLIO.md): 주요 기여와 구현 내용 요약
- [docs/repo-notes.md](./docs/repo-notes.md): 저장소/브랜치 기준과 프로젝트 범위 설명

## 저장소 / 브랜치 기준

| 구분 | 내용 |
|---|---|
| 공식 팀 저장소 | `MATJOM/BACKEND` |
| 공식 기준 브랜치 | `develop` |
| 개인 작업 저장소 | `ROOTXBOT2/MATJOM_BACKEND` |
| 구현 기준 브랜치 | `feature/7-30-search-session-lifecycle` |
| 문서 브랜치 | `docs/matjom-portfolio` |

## 주요 기능

- PostGIS 기반 위치 검색
- 거리 ASC + `place_id` ASC 안정 정렬
- `distance:lastPlaceId` 기반 커서 페이징
- Redis 60초 검색 결과 캐싱
- 검색 결과 수 기반 `meta.reason` 응답
- Idempotency-Key 기반 룰렛 추천
- seed 기반 재현 가능한 추천
- 방문 세션 lifecycle
- 30m / 180초 dwell / 10초 grace window 기반 지오펜스 도착 판정
- 세션 만료 스케줄러
- 검색 API RateLimit
- JWT 기반 Access/Refresh Token
- Testcontainers 기반 점심 여정 통합 테스트

## 기술 스택

- Java 21
- Spring Boot 3.5.5
- Spring Security
- Spring Data JPA
- PostgreSQL / PostGIS
- Redis
- JDBC Template
- JJWT
- Swagger / Springdoc OpenAPI
- Micrometer / Prometheus
- Testcontainers
- JUnit5 / Mockito / AssertJ
- Docker Compose

## 필수 요구 사항

- Java 21
- Gradle Wrapper
- 로컬 기본 실행은 H2 인메모리 DB 프로필을 사용합니다.
- Postgres/PostGIS, Redis 기반 기능을 확인하려면 dev/test 환경 구성이 필요합니다.

## 실행 방법

```bash
# 프로젝트 루트에서 실행
GRADLE_USER_HOME=.gradle ./gradlew bootRun
```

별도의 `SPRING_PROFILES_ACTIVE` 값을 지정하지 않으면 `local` 프로필이 적용되어 H2 DB와 함께 서버가 기동됩니다.

Postgres/Redis 등을 사용해 실제 개발 환경을 구성하려면 `SPRING_PROFILES_ACTIVE=dev`로 실행하고, `infra/` 디렉터리의 Docker Compose 파일을 통해 의존 서비스를 띄우면 됩니다.

## 테스트

```bash
./gradlew test
```

주요 테스트 범위:

- `PlaceSearchServiceTest`
  - Redis 캐시 hit/miss
  - TTL 60초 저장
  - distance cursor 생성
  - 중복 없는 커서 페이징
  - `too_many_results`, `low_results` meta
- `RouletteServiceTest`
  - seed 기반 재현성
  - 후보 없음 예외
  - replay meta 처리
- `VisitSessionServiceTest`
  - 세션 생성
  - ACTIVE 세션 중복 방지
  - 수동 도착 조건 검증
  - 상태 전이 이벤트 기록
- `DefaultGeoFenceEvaluatorTest`
  - 30m / 180초 도착 판정
  - 10초 grace window
  - GPS 정확도 pause
- `LunchJourneyIntegrationTest`
  - Testcontainers 기반 PostGIS/Redis 통합 테스트
  - 회원가입 → 장소 검색 → 방문 세션 생성 → 위치 전송 → 도착 확정 흐름 검증

## 주요 디렉터리 구조

- `src/main/java/com/matjom/matjom/place` : 장소 검색, 상세 조회, PostGIS 기반 Repository
- `src/main/java/com/matjom/matjom/recommendation` : 룰렛 추천 API
- `src/main/java/com/matjom/matjom/visit` : 방문 세션, 위치 이벤트, 지오펜스, 세션 만료
- `src/main/java/com/matjom/matjom/common/idempotency` : 멱등성 공통 모듈
- `src/main/java/com/matjom/matjom/common/ratelimit` : 검색 API RateLimit
- `src/main/java/com/matjom/matjom/common/security` : Spring Security / JWT
- `src/test/java/com/matjom/matjom` : 단위 테스트 및 통합 테스트
- `docs/` : 프로젝트 문서

## API 응답 규약

`com.matjom.matjom.common.response.ApiResponse` 클래스를 통해 REST 응답을 `{ success, data, error, timestamp }` 형태로 통일했습니다.

컨트롤러에서 객체/DTO를 그대로 반환하면 `ApiResponseBodyAdvice`가 자동으로 감싸며, 직접 제어가 필요하면 `ApiResponse.ok(...)`, `ApiResponse.error(...)`를 사용합니다.

예외는 `GlobalExceptionHandler`가 수신해 공통 에러 포맷으로 반환합니다.

## 인프라 연결 테스트 기록

도커로 띄운 Postgres(`db-postgis`)와 Redis(`cache-redis`) 컨테이너가 애플리케이션 dev 프로필이 기대하는 호스트/포트에서 정상적으로 동작하는지 아래와 같이 확인했습니다.

```bash
# 컨테이너 상태 확인
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'

# Postgres 헬스체크 및 접속 검증
docker exec db-postgis pg_isready -U devuser
docker exec db-postgis psql -U devuser -d matjom_dev -c 'SELECT current_database(), current_user;'

# Redis 핑/쓰기/조회 검증
docker exec cache-redis redis-cli ping
docker exec cache-redis redis-cli set healthcheck ok
docker exec cache-redis redis-cli get healthcheck
```

위 명령 결과 `db-postgis`는 `matjom_dev` 데이터베이스에 `devuser` 계정으로 접속 가능했고, Redis는 키/값 쓰기와 `PING` 응답이 정상적으로 반환되었습니다.
