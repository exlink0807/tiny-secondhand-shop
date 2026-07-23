# 3. 보안 체크리스트 및 취약점 리뷰 (OWASP Top 10:2021 기준)

"먼저 취약점을 심고 나중에 고치는" 방식 대신, 설계 단계부터 안전하게 짜는 쪽으로 진행했다.
그래서 아래 표는 "발견 → 수정" 기록이라기보다, OWASP Top 10 항목별로 어떤 위험을 검토했고
어떤 방어 로직을 넣었으며 어떤 테스트로 확인했는지를 정리한 리뷰에 가깝다. 실제로 코드를
짜다가 방향을 바꾼 부분은 3.2에 따로 적어두었다.

## 3.1 OWASP Top 10:2021 매핑

| # | 항목 | 이 프로젝트에서의 위험 | 구현한 대응 | 검증 테스트 |
|---|------|----------------------|-------------|--------------|
| A01 | Broken Access Control | 타인의 상품 수정/삭제(IDOR), 타인 채팅방 열람, 일반 유저의 관리자 기능 접근 | 모든 라우트에서 `require_login`/`require_admin` 의존성 강제, 리소스 조회 후 `seller_id`/참여자 id와 현재 로그인 유저 비교 | `test_authorization.py` (403 확인 6건) |
| A02 | Cryptographic Failures | 비밀번호 평문 저장/노출 | bcrypt 해시 저장(passlib), 평문 비밀번호 로깅 금지, 세션은 서버 서명 쿠키 사용(JWT를 localStorage에 두지 않음) | 코드 리뷰 (security.py) |
| A03 | Injection (SQL/XSS) | 검색어를 통한 SQL Injection, 채팅/상품명을 통한 Stored XSS | SQLAlchemy ORM만 사용(raw SQL 미사용), Jinja2 autoescape 유지(`|safe` 미사용) | `test_product_search_is_safe_against_sql_injection_like_input`, `test_stored_xss_payload_is_escaped_in_product_title` |
| A04 | Insecure Design | 송금 시 잔액 음수/이중지출, 회원가입 시 과도한 포인트 지급 등 비즈니스 로직 결함 | DB `CheckConstraint(balance >= 0)`, 트랜잭션 단위 잔액 재검증, 관리자 지급 한도(1회 최대 100만원) | `test_wallet.py` (잔액부족/음수금액/한도), `test_admin_grant_balance_rejects_excessive_amount` |
| A05 | Security Misconfiguration | 운영 환경에서 상세 에러/스택트레이스 노출, 보안 헤더 미설정 | 전역 예외 핸들러가 `DEBUG=false`일 때 내부 오류 은닉, CSP/X-Frame-Options/X-Content-Type-Options 등 보안 헤더 미들웨어 적용 | 수동 curl 헤더 확인 (§4 테스트 로그) |
| A06 | Vulnerable and Outdated Components | 알려진 취약점이 있는 패키지 버전 사용 | requirements.txt에 버전 고정, 정기적 `pip list --outdated` 점검 권장 (README 참고) | 수동 점검 |
| A07 | Identification and Authentication Failures | 브루트포스 로그인, 세션 고정(Session Fixation), 계정 존재 여부 노출 | 로그인 5회 실패 시 5분 잠금, 로그인 성공 시 세션 재발급(clear 후 재구성), 로그인 실패 메시지를 계정 존재 여부와 무관하게 동일하게 처리 | `test_login_rate_limited_after_repeated_failures`, `test_login_wrong_password_generic_message`, `test_login_nonexistent_user_same_generic_message` |
| A08 | Software and Data Integrity Failures | 업로드 파일을 통한 악성 코드 실행(확장자 위장) | 확장자 화이트리스트 + 매직바이트(실제 파일 시그니처) 검증 + 저장 파일명 UUID 재생성 | `test_upload_rejects_non_image_extension`, `test_upload_rejects_fake_extension_with_wrong_content`, `test_upload_accepts_valid_png` |
| A09 | Security Logging and Monitoring Failures | 관리자 행위 부인/추적 불가 | `AuditLog` 테이블에 관리자 행위(정지, 숨김, 신고처리, 지급) 기록, 실패 로그인 로그(향후 확장 가능) | `test_admin.py` (조치 후 대시보드 로그 노출 확인) |
| A10 | Server-Side Request Forgery (SSRF) | 본 앱은 사용자 입력을 기반으로 서버가 외부 URL을 요청하는 기능이 없음(해당 없음) | 해당 기능 미구현으로 공격면 자체가 없음 | - |

또한 OWASP Top 10에 명시적으로 없지만 검토한 항목:

| 위험 | 대응 |
|---|---|
| CSRF | 모든 상태 변경 POST 요청에 세션 저장 토큰과 폼 토큰을 `secrets.compare_digest`로 비교 검증 |
| 클릭재킹 | `X-Frame-Options: DENY`, CSP `frame-ancestors 'none'` |
| MIME 스니핑 | `X-Content-Type-Options: nosniff` |
| 계정 열거(Enumeration) | 회원가입 시 아이디/이메일 중복만 안내(부득이 노출), 로그인 실패 시에는 동일 메시지로 통일 |
| 대용량 파일 업로드(DoS) | 업로드 크기 5MB 제한 |
| 자기 자신에게 송금 등 논리적 이상 흐름 | 서버측에서 별도 검증 후 차단 |

## 3.2 코드 리뷰 과정에서 조정한 사항

설계와 구현을 진행하며 실제로 방향을 수정한 부분들입니다.

1. **세션 저장 방식**: 처음에는 클라이언트 저장이 간편한 JWT를 고려했으나, 브라우저(localStorage)에 저장 시 XSS 한 번으로 토큰이 탈취될 수 있어 서버 서명 쿠키(HttpOnly) 기반 세션으로 설계를 변경함.
2. **로그인 실패 메시지**: 최초에는 "존재하지 않는 아이디입니다"/"비밀번호가 틀렸습니다"로 구분하려 했으나, 이는 계정 존재 여부를 공격자에게 알려주는 사용자 열거(User Enumeration) 취약점이 되므로 두 경우 모두 동일한 문구로 통일함.
3. **파일 업로드 검증**: 확장자 검사만으로는 `.jpg`로 이름만 바꾼 스크립트 파일 업로드를 막을 수 없어, 매직바이트(파일 시그니처) 검증을 추가함. `python-magic`(libmagic) 사용이 불가능한 배포 환경을 고려해 시그니처 하드코딩 fallback도 함께 구현함.
4. **송금 동시성**: 단순히 "잔액 조회 → 차감"으로 구현할 경우 동시 요청 시 이중 지출 가능성이 있어, 같은 DB 트랜잭션 내에서 최신 잔액을 재조회 후 커밋하는 방식으로 조정하고, `CheckConstraint(balance >= 0)`를 DB 레벨에서도 강제함. (다중 워커/분산 배포 시에는 PostgreSQL 등 row-lock 지원 DB로 전환 필요 — README/설계문서에 명시)
5. **비밀번호 해시 라이브러리 버전 충돌**: `passlib[bcrypt]` 최신 `bcrypt`(5.x)와 호환성 문제(72바이트 초과 등 내부 버그 감지 로직 오류)가 있어 `bcrypt==4.0.1`로 버전을 고정함. (기능 버그였으나 인증 핵심 로직이라 보안 체크리스트에 기록)
6. **관리자 초기 계정**: 하드코딩된 고정 비밀번호로 관리자 계정을 생성하면 소스가 공개(GitHub public)되었을 때 그대로 노출되므로, 환경변수 미설정 시 매 최초 실행마다 랜덤 비밀번호를 생성해 최초 1회만 콘솔에 출력하도록 변경함.
7. **SECRET_KEY 고정 위험**: 세션 서명 키를 코드에 하드코딩하면 GitHub 공개 저장소에 노출되어 세션 위조가 가능해지므로, 환경변수 미설정 시 매 실행마다 랜덤 키를 생성하도록 처리(단, 이 경우 서버 재시작 시 기존 세션은 무효화됨 — 운영 배포 시에는 반드시 `SECRET_KEY` 환경변수 고정 필요, README에 명시).

## 3.3 테스트 실행 결과 요약

```
tests/ 31 passed (pytest, FastAPI TestClient 기반)
tests/smoke_test.py 18/18 통과 (실제 실행 서버 대상 end-to-end 시나리오)
```

실행 방법은 README.md의 "테스트 실행" 항목 참고.
