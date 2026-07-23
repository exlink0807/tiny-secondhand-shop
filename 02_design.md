# 2. 시스템 설계 (System Design)

## 2.1 아키텍처
```
[Browser] --HTTPS--> [FastAPI (Uvicorn)]
                          |-- Jinja2 Templates (SSR, autoescape)
                          |-- Session Middleware (서명된 쿠키, HttpOnly/SameSite=Strict)
                          |-- SQLAlchemy ORM ---> [SQLite DB]
                          |-- Static/Upload 파일 서빙 (검증된 파일만)
```
- 서버사이드 렌더링(Jinja2) 기반의 단순 구조. 클라이언트에 토큰(JWT)을 저장하지 않고, 서버 서명 세션 쿠키 사용 → XSS로 인한 토큰 탈취 표면 축소.
- 모든 상태 변경 요청(POST/PUT/DELETE)에 CSRF 토큰 필수.

## 2.2 ERD (개체-관계 모델)

```
User
 - id (PK)
 - username (unique)
 - email (unique)
 - password_hash
 - role (enum: user, admin)
 - status (enum: active, suspended)
 - balance (integer, 원 단위, >= 0)
 - created_at

Product
 - id (PK)
 - seller_id (FK -> User.id)
 - title
 - description
 - price (integer)
 - category
 - status (enum: available, sold, hidden_by_admin, deleted)
 - image_path (nullable)
 - created_at

Conversation
 - id (PK)
 - product_id (FK -> Product.id)
 - buyer_id (FK -> User.id)
 - seller_id (FK -> User.id)
 - created_at
 - unique(product_id, buyer_id)

Message
 - id (PK)
 - conversation_id (FK -> Conversation.id)
 - sender_id (FK -> User.id)
 - content
 - created_at

Report
 - id (PK)
 - reporter_id (FK -> User.id)
 - target_type (enum: user, product)
 - target_id (int)
 - reason
 - status (enum: pending, reviewed, actioned, dismissed)
 - created_at
 - reviewed_by (FK -> User.id, nullable)
 - reviewed_at (nullable)

Transaction (송금)
 - id (PK)
 - sender_id (FK -> User.id)
 - receiver_id (FK -> User.id)
 - amount (integer, > 0)
 - memo
 - created_at

AuditLog (관리자 행위 기록)
 - id (PK)
 - admin_id (FK -> User.id)
 - action (예: SUSPEND_USER, HIDE_PRODUCT, RESOLVE_REPORT, GRANT_BALANCE)
 - target_type
 - target_id
 - detail
 - created_at
```

관계:
- User 1 --- N Product (seller)
- Product 1 --- N Conversation
- Conversation 1 --- N Message
- User 1 --- N Report (reporter)
- User 1 --- N Transaction (sender), 1 --- N Transaction (receiver)

## 2.3 API 설계 (요약)

| 메서드 | 경로 | 설명 | 권한 |
|---|---|---|---|
| GET/POST | /register | 회원가입 | 비로그인 |
| GET/POST | /login | 로그인 | 비로그인 |
| POST | /logout | 로그아웃 | 로그인 |
| GET | /products | 상품 목록/검색 | 전체 |
| GET | /products/{id} | 상품 상세 | 전체 |
| GET/POST | /products/new | 상품 등록 | 로그인 |
| GET/POST | /products/{id}/edit | 상품 수정 | 소유자 |
| POST | /products/{id}/delete | 상품 삭제 | 소유자/관리자 |
| GET | /chat | 대화 목록 | 로그인 |
| GET/POST | /chat/{conversation_id} | 채팅방 | 대화 참여자만 |
| POST | /products/{id}/chat/start | 채팅 시작(구매자) | 로그인 |
| POST | /reports/new | 신고 등록 | 로그인 |
| GET/POST | /wallet | 잔액 확인 및 송금 | 로그인 |
| GET | /admin | 관리자 대시보드 | 관리자 |
| POST | /admin/users/{id}/suspend | 유저 정지/해제 | 관리자 |
| POST | /admin/products/{id}/hide | 상품 숨김/복구 | 관리자 |
| POST | /admin/reports/{id}/resolve | 신고 처리 | 관리자 |
| POST | /admin/users/{id}/grant | 데모용 잔액 지급 | 관리자 |

모든 "권한: 소유자/참여자/관리자" 엔드포인트는 서버에서 `현재 로그인 사용자 == 리소스 소유자 or role==admin` 을 반드시 검증 (IDOR 방지).

## 2.4 위협 모델링 (STRIDE 요약)

| 위협 유형 | 시나리오 | 대응 |
|---|---|---|
| Spoofing | 세션 쿠키 위조/탈취 | 서명된 세션(itsdangerous), HttpOnly+Secure+SameSite, 세션 만료 |
| Tampering | 상품 가격/타인 채팅 조작 | 서버측 소유자 검증, ORM 파라미터 바인딩 |
| Repudiation | 관리자 조치 부인 | AuditLog 기록 |
| Information Disclosure | 타 유저 정보/채팅 열람(IDOR), 에러 스택 노출 | 인가 검사, 프로덕션 에러 핸들러 |
| Denial of Service | 로그인 브루트포스, 대량 요청 | 로그인 시도 제한(rate limit), 잠금 |
| Elevation of Privilege | 일반 유저가 관리자 API 접근 | role 기반 의존성 검사(require_admin) |

## 2.5 보안 설계 원칙 (구현 전 확정)
1. 비밀번호: bcrypt(passlib)로 해시, 원문 저장/로그 금지
2. DB 접근: SQLAlchemy ORM만 사용, raw SQL 문자열 결합 금지
3. 출력: Jinja2 autoescape 유지 (수동 |safe 사용 금지)
4. 인가: 모든 수정/조회 라우트에서 소유권·역할 검사를 의존성(Depends)으로 강제
5. 파일 업로드: 확장자 화이트리스트 + 매직바이트 검증 + UUID 파일명 + 정적 서빙 경로 분리
6. 송금: DB 트랜잭션 + row lock(또는 SQLite 특성상 단일 트랜잭션 커밋)으로 동시성 이슈 방지, 잔액 검증
7. CSRF: 폼마다 hidden token, 서버에서 세션 저장 토큰과 대조
8. 보안 헤더 미들웨어: CSP, X-Content-Type-Options, X-Frame-Options, Referrer-Policy
9. 로깅: 실패 로그인 시도, 관리자 액션만 기록. 비밀번호/세션토큰 로깅 금지
