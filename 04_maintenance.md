# 4. 유지보수 계획 (Maintenance)

## 4.1 의존성 관리
- `requirements.txt`에 모든 패키지 버전을 고정(pin)하여 배포 환경 재현성을 확보.
- 월 1회 `pip list --outdated`로 알려진 취약점(CVE)이 있는 패키지 확인 후 마이너 버전 업데이트.
- 특히 `passlib`, `bcrypt`, `starlette`, `fastapi`는 인증/세션 관련 핵심 의존성이므로 CVE 공지를 우선 확인.

## 4.2 배포 환경 전환 시 체크리스트
- `DATABASE_URL`을 PostgreSQL 등으로 전환 시 `wallet.py`의 동시성 처리 로직(§2.5, §3.2-4)을 `SELECT ... FOR UPDATE` 기반으로 재검증.
- `SECRET_KEY`, `ADMIN_PASSWORD`를 환경변수로 반드시 고정 지정(미지정 시 재시작마다 랜덤 값으로 세션 무효화/관리자 비밀번호 변경됨).
- 리버스 프록시(Nginx 등) 뒤에서 HTTPS로 서비스할 경우 `HTTPS_ONLY=true`로 세션 쿠키 Secure 플래그 활성화.

## 4.3 로깅/모니터링
- 현재 애플리케이션 로그는 표준 `logging` 모듈로 콘솔 출력. 운영 배포 시 파일 로테이션(`TimedRotatingFileHandler`) 또는 중앙 로그 수집(ELK 등) 연동 권장.
- `AuditLog` 테이블은 삭제되지 않고 계속 누적되므로, 일정 주기로 아카이빙 정책 수립 필요.
- 5xx 에러 발생 시 알림(Slack/이메일 등) 연동은 향후 개선 과제로 분류.

## 4.4 백업
- SQLite 파일(`app.db`) 기준 매일 1회 스냅샷 백업 권장. PostgreSQL 전환 시 `pg_dump` 기반 정기 백업으로 전환.

## 4.5 기능 확장 시 보안 회귀 방지
- 새 라우트 추가 시 아래 3가지를 반드시 리뷰:
  1. 로그인이 필요한가? (`require_login`/`require_admin` 의존성 부착 여부)
  2. 리소스 소유자 검증이 필요한가? (IDOR 방지)
  3. 상태를 변경하는 POST/PUT/DELETE라면 CSRF 토큰 검증이 포함되었는가?
- `tests/` 하위에 최소 1개 이상의 인가/입력검증 테스트를 추가하고 CI(또는 PR 전 로컬 `pytest`)에서 통과를 확인 후 병합.

## 4.6 버전 관리 및 브랜치 전략
- `main` 브랜치는 항상 배포 가능한 상태 유지.
- 기능 단위로 브랜치를 분리(`feature/xxx`)하고, 병합 전 `pytest` 전체 통과를 필수 조건으로 함.
