# MATJOM - 위치 기반 점심 추천 백엔드 프로젝트

> 팀 협업 프로젝트에서 담당한 백엔드 기여 내용을 정리한 포트폴리오 문서입니다.  
> 공식 기준 저장소: `MATJOM/BACKEND`  
> 개인 구현 근거 저장소: `ROOTXBOT2/MATJOM_BACKEND`  
> 문서 기준 브랜치: `docs/matjom-portfolio`  
> 구현 기준 브랜치: `feature/7-30-search-session-lifecycle`

## 1. 프로젝트 개요

MATJOM은 직장인의 반복적인 점심 의사결정 문제를 줄이기 위한 위치 기반 점심 추천 서비스입니다.

사용자는 현재 위치를 기준으로 주변 식당을 검색하고, 룰렛 추천으로 빠르게 후보를 고른 뒤, 방문 세션을 통해 실제 이동과 도착 상태를 기록할 수 있습니다.

- 프로젝트명: 오늘은 또 뭐먹지, MATJOM
- 팀명: 맛점을 위해서
- 역할: 백엔드 개발자 / 팀장
- 주요 담당: 위치 기반 검색, 룰렛 추천, 방문 세션 lifecycle, 지오펜스 도착 판정, 멱등성, 캐싱, 테스트
- 공식 저장소: https://github.com/MATJOM/BACKEND
- 개인 구현 저장소: https://github.com/ROOTXBOT2/MATJOM_BACKEND

## 2. 담당 역할 요약

| 영역 | 담당 내용 |
|---|---|
| 프로젝트 리딩 | 요구사항 정의, 기능 분해, 백엔드 구현 방향 정리 |
| 장소 검색 | PostGIS 기반 반경 검색, 거리 ASC 정렬, distance:id 커서 페이징 |
| 캐싱 | Redis 60초 검색 결과 캐싱, 캐시 hit/miss 흐름 처리 |
| 룰렛 추천 | Idempotency-Key 기반 중복 요청 방지, seed 기반 재현 가능한 추천 |
| 방문 세션 | 사용자당 ACTIVE 세션 1개 제한, 위치 이벤트 기록, 수동 도착 확정, 세션 만료 스케줄러 |
| 지오펜스 | 30m 반경, GPS 정확도 30m 이하, 180초 dwell, 10초 grace window 기반 도착 판정 |
| 공통 인프라 | JWT 인증, RateLimit, Redis 기반 IdempotencyStore |
| 테스트 | 단위 테스트 및 Testcontainers 기반 점심 여정 통합 테스트 |

## 3. 주요 구현 내용

### 3.1 PostGIS 기반 위치 검색

현재 위치 기준으로 주변 식당을 검색하는 API를 구현했습니다.

주요 구현 내용:

- 기본 반경 300m
- 기본 페이지 크기 20
- 최대 검색 결과 500건 제한
- PostGIS `ST_DWithin` 기반 반경 필터
- PostGIS `ST_Distance` 기반 meter 단위 거리 계산
- `distance ASC, place_id ASC` 안정 정렬
- `distance:lastPlaceId` 기반 커서 페이징
- `size + 1` 조회로 다음 페이지 여부 판단
- 결과가 너무 많거나 적을 때 `meta.reason` 응답 제공

관련 코드:

- `src/main/java/com/matjom/matjom/place/api/PlaceController.java`
- `src/main/java/com/matjom/matjom/place/service/PlaceSearchService.java`
- `src/main/java/com/matjom/matjom/place/repository/PlaceRepository.java`
- `src/main/java/com/matjom/matjom/place/dto/PlaceSearchCursor.java`
- `src/main/java/com/matjom/matjom/place/dto/PlaceSearchRequest.java`
- `src/main/java/com/matjom/matjom/place/dto/PlaceSearchResponse.java`

### 3.2 Redis 60초 검색 캐싱

동일한 검색 조건에 대해서 Redis 캐시를 먼저 조회하고, 캐시가 없을 때만 PostGIS 쿼리를 수행하도록 구성했습니다.

주요 구현 내용:

- 검색 조건 기반 캐시 키 생성
- 응답 DTO 전체를 JSON으로 직렬화해 Redis 저장
- TTL 60초 적용
- 캐시 데이터 역직렬화 실패 시 삭제 후 DB 재조회
- 캐시 hit 시 Repository 호출 생략

관련 코드:

- `PlaceSearchService.java`
- `PlaceSearchServiceTest.java`

### 3.3 Idempotency-Key 기반 룰렛 추천

점심 추천 후보 중 하나를 룰렛 방식으로 선택하는 API를 구현했습니다.

주요 구현 내용:

- `Idempotency-Key` 기반 결과 재사용
- 요청 body를 SHA-256 hash로 비교
- 동일 key + 동일 요청이면 기존 추천 결과 replay
- 동일 key + 다른 요청이면 conflict 처리
- seed가 있으면 재현 가능한 랜덤 추천
- seed가 없으면 일반 랜덤 추천
- 응답 meta에 후보 수와 replay 여부 포함

관련 코드:

- `src/main/java/com/matjom/matjom/recommendation/api/RouletteController.java`
- `src/main/java/com/matjom/matjom/recommendation/service/RouletteService.java`
- `src/main/java/com/matjom/matjom/recommendation/dto/RouletteRequest.java`
- `src/main/java/com/matjom/matjom/recommendation/dto/RouletteResponse.java`
- `src/main/java/com/matjom/matjom/common/idempotency/RedisIdempotencyStore.java`

### 3.4 방문 세션 lifecycle

장소 선택 이후 사용자의 방문 흐름을 세션으로 관리했습니다.

주요 구현 내용:

- 방문 세션 시작 API
- 세션 시작 요청 멱등 처리
- 사용자당 ACTIVE 세션 1개 제한
- 위치 이벤트 기록
- 수동 도착 확정 API
- 수동 도착 요청 멱등 처리
- 수동 도착 최소 10분 / 최대 60분 조건
- 수동 도착 30m 이내 조건
- 상태 전이 이벤트 기록
- 30분 초과 ACTIVE 세션 만료 처리

관련 코드:

- `src/main/java/com/matjom/matjom/visit/api/VisitSessionController.java`
- `src/main/java/com/matjom/matjom/visit/service/VisitSessionService.java`
- `src/main/java/com/matjom/matjom/visit/service/VisitPositionService.java`
- `src/main/java/com/matjom/matjom/visit/service/VisitTimeoutScheduler.java`
- `src/main/java/com/matjom/matjom/visit/service/VisitStateTransitionRecorder.java`
- `src/main/java/com/matjom/matjom/visit/entity/Visit.java`
- `src/main/java/com/matjom/matjom/visit/entity/VisitState.java`
- `src/main/java/com/matjom/matjom/visit/entity/VisitEvent.java`

### 3.5 지오펜스 도착 판정

자동 도착 판정을 위해 지오펜스 평가 로직을 구현했습니다.

주요 정책:

- 장소 기준 30m 반경 이내
- GPS 정확도 30m 이하일 때만 판정
- 180초 dwell 조건
- 10초 grace window
- ACTIVE 세션일 때만 ARRIVED 전이
- GPS 정확도가 낮으면 dwell 계산 일시 정지
- grace window 초과 이탈 시 dwell reset

관련 코드:

- `src/main/java/com/matjom/matjom/visit/geofence/DefaultGeoFenceEvaluator.java`
- `src/main/java/com/matjom/matjom/visit/geofence/GeoFenceEvaluator.java`
- `src/main/java/com/matjom/matjom/visit/geofence/GeoFenceEvaluationResult.java`
- `src/main/java/com/matjom/matjom/visit/util/GeoDistanceCalculator.java`

### 3.6 RateLimit / JWT / Idempotency 공통 기반

검색 API와 추천/세션 API의 안정성을 위해 공통 인프라를 구성했습니다.

주요 구현 내용:

- 장소 검색 API RateLimit 적용
- 사용자 ID 또는 IP 기반 RateLimit key 구성
- 초과 시 `429 Too Many Requests` 응답
- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After` 헤더 반환
- JWT Access/Refresh Token 발급 및 claims parsing
- 로그아웃 시 Access Token 잔여 TTL 기반 블랙리스트 처리
- Redis 기반 IdempotencyStore 구현

관련 코드:

- `src/main/java/com/matjom/matjom/common/ratelimit/RateLimitFilter.java`
- `src/main/java/com/matjom/matjom/common/ratelimit/RedisSearchRateLimiter.java`
- `src/main/java/com/matjom/matjom/common/security/jwt/JwtTokenProvider.java`
- `src/main/java/com/matjom/matjom/auth/service/LoginService.java`
- `src/main/java/com/matjom/matjom/auth/service/LogoutService.java`
- `src/main/java/com/matjom/matjom/common/idempotency/RedisIdempotencyStore.java`

## 4. 테스트 / 검증

### 4.1 단위 테스트

작성한 주요 테스트:

- `PlaceSearchServiceTest`
  - 캐시 hit/miss
  - TTL 60초 저장
  - nextCursor 생성
  - 여러 페이지 조회 시 중복 없는 복원
  - invalid cursor 예외
  - `too_many_results`, `low_results` meta
- `RouletteServiceTest`
  - seed 기반 재현성
  - 후보 없음 예외
  - 추천 분포 검증
  - replay meta 처리
- `VisitSessionServiceTest`
  - 세션 생성
  - ACTIVE 세션 중복 방지
  - 수동 도착 조건 검증
  - 30m 초과 도착 차단
  - 10분 미만 / 60분 초과 도착 차단
  - 상태 전이 이벤트 기록
- `DefaultGeoFenceEvaluatorTest`
  - 30m / 180초 도착 판정
  - 10초 grace 유지
  - grace 초과 시 dwell reset
  - GPS 정확도 낮을 때 pause
  - 경계값 테스트

### 4.2 통합 테스트

`LunchJourneyIntegrationTest`에서는 Testcontainers로 PostGIS와 Redis를 구동해 실제 점심 추천 여정을 검증했습니다.

검증 흐름:

1. 회원가입
2. Access / Refresh Token 확보
3. PostGIS geography 필드 포함 장소 데이터 삽입
4. 위치 기반 장소 검색
5. 방문 세션 생성
6. 위치 이벤트 전송
7. 수동 도착 확정
8. 세션 상태 ARRIVED 확인

## 5. 기술 스택

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

## 6. 이 프로젝트에서 보여줄 수 있는 역량

- 위치 기반 서비스의 검색/추천/방문 상태 흐름을 백엔드 도메인으로 설계한 경험
- PostGIS 기반 거리 계산과 cursor pagination 구현 경험
- Redis 캐싱과 Idempotency-Key를 이용해 재시도 안정성을 고려한 경험
- 지오펜스 도착 판정처럼 시간/위치/상태 조건이 결합된 도메인 로직 구현 경험
- Testcontainers 기반 통합 테스트로 실제 사용자 여정을 검증한 경험

## 7. 이력서용 요약

MATJOM은 직장인의 점심 선택 시간을 줄이기 위한 위치 기반 점심 추천 백엔드 프로젝트입니다. 팀장 역할로 요구사항 정의와 기능 분해를 주도했고, 백엔드에서는 PostGIS 기반 장소 검색, 거리 정렬 커서 페이징, Redis 캐싱, 멱등성 기반 룰렛 추천, 방문 세션 lifecycle, 지오펜스 도착 판정을 구현했습니다. 특히 Testcontainers로 회원가입부터 장소 검색, 방문 세션 생성, 도착 확정까지 전체 여정을 검증했습니다.
