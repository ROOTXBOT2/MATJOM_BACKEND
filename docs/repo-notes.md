# MATJOM Repo Notes

> 이 문서는 MATJOM 프로젝트의 공식 저장소, 개인 구현 저장소, 작업 브랜치 관계를 포트폴리오 관점에서 설명하기 위한 메모입니다.

## 1. 저장소 기준

MATJOM은 공식 팀 저장소와 개인 작업 저장소가 나뉘어 있습니다.

```text
공식 팀 저장소: MATJOM/BACKEND
공식 기준 브랜치: develop

개인 구현 근거 저장소: ROOTXBOT2/MATJOM_BACKEND
개인 구현 기준 브랜치: feature/7-30-search-session-lifecycle
포트폴리오 문서 브랜치: docs/matjom-portfolio
```

## 2. 왜 개인 구현 브랜치를 기준으로 문서를 작성했는가

`MATJOM/BACKEND`는 공식 기준 저장소입니다. 다만 현재 확인 가능한 `develop` 브랜치는 프로젝트 스켈레톤과 기본 실행 환경 중심입니다.

실제 위치 기반 검색, 룰렛 추천, 방문 세션 lifecycle, 지오펜스 도착 판정, 멱등성, 테스트 코드는 `ROOTXBOT2/MATJOM_BACKEND`의 `feature/7-30-search-session-lifecycle` 브랜치에 가장 많이 남아 있습니다.

따라서 포트폴리오 문서는 공식 저장소 구조를 언급하되, 구현 근거는 개인 작업 브랜치를 기준으로 작성했습니다.

## 3. 주요 브랜치 역할

### MATJOM/BACKEND develop

공식 팀 저장소의 기준 브랜치입니다.

확인된 성격:

- 프로젝트 기본 구조
- Spring Boot 실행 환경
- 공통 응답/예외 규약
- 인프라 연결 테스트 기록

### ROOTXBOT2/MATJOM_BACKEND main

개인 fork 또는 개인 작업 저장소의 기본 브랜치입니다.

확인된 성격:

- 공식 저장소 초기 구조와 유사한 기반
- 실제 기능 구현은 feature 브랜치에 집중됨

### ROOTXBOT2/MATJOM_BACKEND feature/7-search-cursor-paging

장소 검색과 룰렛 추천 중심의 기능 브랜치입니다.

확인된 성격:

- PostGIS 기반 장소 검색
- distance:id 커서 페이징
- Redis 60초 검색 캐싱
- 룰렛 추천 API
- RateLimit / Idempotency 기반 일부

### ROOTXBOT2/MATJOM_BACKEND feature/30-session-lifecycle

방문 세션 lifecycle 중심의 기능 브랜치입니다.

확인된 성격:

- 방문 세션 생성
- 위치 이벤트 기록
- 지오펜스 도착 판정
- 수동 도착 확정
- 세션 만료 스케줄러

### ROOTXBOT2/MATJOM_BACKEND feature/7-30-search-session-lifecycle

포트폴리오 기준 브랜치입니다.

확인된 성격:

- 검색/추천 기능과 방문 세션 기능이 함께 반영된 통합 브랜치
- 테스트 코드와 통합 테스트가 가장 많이 포함됨
- 포트폴리오 문서 작성 기준으로 사용

## 4. 포트폴리오에서 설명할 때의 기준 문장

면접이나 이력서에서 저장소 관계를 설명해야 한다면 아래처럼 말하는 것이 안전합니다.

```text
공식 팀 저장소는 MATJOM/BACKEND이고, 제가 담당한 백엔드 기능 구현과 검증 코드는 ROOTXBOT2/MATJOM_BACKEND의 feature 브랜치에 남아 있습니다. 포트폴리오 문서는 실제 구현 근거가 가장 많은 feature/7-30-search-session-lifecycle 브랜치를 기준으로 정리했습니다.
```

## 5. 구현 근거가 있는 핵심 기능

```text
- PostGIS 기반 반경 검색
- 거리 ASC + place_id ASC 안정 정렬
- distance:id 커서 페이징
- Redis 60초 검색 캐싱
- 결과 수 기반 meta.reason 응답
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
```

## 6. 표현 주의 사항

포트폴리오와 이력서에서는 아래 표현을 조심합니다.

| 피할 표현 | 이유 | 안전한 표현 |
|---|---|---|
| 공식 develop에 최종 반영 완료 | 실제 핵심 구현은 개인 feature 브랜치에 있음 | 개인 작업 브랜치에서 핵심 백엔드 기능 구현 |
| RS256 JWT 구현 | 현재 구현은 HMAC 기반 JWT | JWT 기반 Access/Refresh Token 구조 구현 |
| 완전한 운영 관측성 구축 | 관측성은 의존성과 기반 중심 | 운영 지표 수집 기반 고려 |
| AI 추천 알고리즘 구현 | 현재는 룰렛/랜덤 추천 중심 | seed 기반 재현 가능한 룰렛 추천 |

## 7. 레포 관리 메모

이 프로젝트는 공식 저장소와 개인 구현 저장소가 분리되어 있어, 채용자에게는 `PORTFOLIO.md`를 먼저 보여주는 것이 좋습니다.

우선순위:

```text
1. PORTFOLIO.md 링크 제공
2. 기준 브랜치 설명
3. 실제 코드 파일과 테스트 파일 확인 가능하게 경로 제공
4. 공식 저장소와 개인 구현 저장소 관계 설명
```

## 8. 다음에 정리하면 좋은 항목

필수는 아니지만, 시간이 있다면 아래를 추가하면 좋습니다.

```text
- README.md에 PORTFOLIO.md 링크 추가
- docs/code-walkthrough.md 작성
- docs/test-strategy.md 작성
- feature/7-30-search-session-lifecycle 브랜치의 핵심 커밋 목록 요약
```
