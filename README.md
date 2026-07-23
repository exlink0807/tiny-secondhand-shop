# Tiny Second-hand Shopping Platform

화이트햇스쿨(WHS) 시큐어 코딩 과제로 제작한 중고거래 플랫폼입니다.
FastAPI + Jinja2(서버사이드 렌더링) + SQLite 기반이며, 설계 단계부터 보안을 고려해 구현했습니다.

- 요구사항 분석: [`docs/01_requirements.md`](docs/01_requirements.md)
- 시스템 설계(ERD/API/위협모델링): [`docs/02_design.md`](docs/02_design.md)
- 보안 체크리스트 및 리뷰: [`docs/03_security_checklist.md`](docs/03_security_checklist.md)
- 유지보수 계획: [`docs/04_maintenance.md`](docs/04_maintenance.md)

## 주요 기능
- 회원가입/로그인/로그아웃 (bcrypt 해시, 세션 기반 인증)
- 상품 등록/조회/검색/수정/삭제 (이미지 업로드 포함)
- 상품 기준 구매자-판매자 1:1 채팅
- 유저/상품 신고 → 관리자 검토 처리
- 유저 간 포인트 송금 및 거래 내역 조회
- 관리자 대시보드 (회원 정지/해제, 상품 숨김, 신고 처리, 잔액 지급, 감사 로그)

## 기술 스택
Python 3.12 · FastAPI · Jinja2 · SQLAlchemy · SQLite · passlib(bcrypt) · Starlette SessionMiddleware

## 1. 환경 설정

### 1-1. 저장소 클론 및 가상환경
```bash
git clone <이 저장소 URL>
cd tiny-secondhand-shop

python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
```

### 1-2. 패키지 설치
```bash
pip install -r requirements.txt
```

### 1-3. 환경변수 설정 (선택, 데모 실행에는 필수 아님)
```bash
cp .env.example .env
# .env 파일을 열어 SECRET_KEY, ADMIN_PASSWORD 등을 지정
```
> `.env`를 설정하지 않고 실행해도 동작합니다. 이 경우 `SECRET_KEY`와 관리자(admin) 비밀번호가
> 매 최초 실행마다 랜덤 생성되며, 관리자 비밀번호는 **최초 실행 시 콘솔 로그에 한 번만 출력**됩니다.
> (운영 배포 시에는 반드시 `.env`에 고정 값을 지정하세요.)

## 2. 실행 방법

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

브라우저에서 `http://localhost:8000` 접속.

- 최초 실행 시 콘솔에 다음과 같은 로그가 출력됩니다. 관리자 계정으로 로그인하려면 이 비밀번호를 사용하세요.
  ```
  초기 관리자 계정이 생성되었습니다. username=admin password=xxxxxxxxxx
  ```
- 일반 회원가입은 `/register`에서 진행하며, 가입 시 데모용으로 10,000원이 기본 지급됩니다(실제 결제 연동 없음).

## 3. 테스트 실행

### 3-1. pytest (단위/통합 테스트, 격리된 임시 DB 사용)
```bash
pip install pytest httpx
pytest tests/ -v
```
인증, 인가(IDOR), CSRF, 파일 업로드 검증, XSS 이스케이프, 송금 동시성/한도, 관리자 기능 등 31개 테스트를 포함합니다.

### 3-2. 엔드투엔드 스모크 테스트 (실제 실행 중인 서버 대상)
```bash
# 터미널 1
uvicorn app.main:app --port 8000

# 터미널 2
python3 tests/smoke_test.py
```

## 4. 프로젝트 구조
```
app/
  main.py            # FastAPI 앱, 미들웨어(보안 헤더/세션), 에러 핸들러
  config.py          # 환경설정
  database.py        # SQLAlchemy 세션/엔진
  models.py          # ORM 모델 (User, Product, Conversation, Message, Report, Transaction, AuditLog)
  security.py        # 비밀번호 해시, CSRF, 로그인 시도 제한
  dependencies.py     # 인증/인가 Depends
  utils.py           # CSRF 토큰 헬퍼, 업로드 파일 검증
  templating.py      # Jinja2Templates 인스턴스
  routers/
    auth.py products.py chat.py reports.py wallet.py admin.py
templates/           # Jinja2 템플릿 (서버사이드 렌더링)
static/              # CSS, 업로드 이미지
docs/                # 요구사항/설계/보안체크리스트/유지보수 문서
tests/               # pytest + 스모크 테스트
```

## 5. 보안 설계 요약
- SQL Injection 방지: SQLAlchemy ORM만 사용 (raw SQL 문자열 결합 없음)
- XSS 방지: Jinja2 autoescape 유지, `|safe` 미사용
- CSRF 방지: 모든 상태 변경 요청에 세션-폼 토큰 대조 검증
- IDOR 방지: 모든 리소스 접근에서 소유자/참여자/역할 검증
- 인증: bcrypt 비밀번호 해시, 서버 서명 세션 쿠키(HttpOnly/SameSite=Strict), 로그인 시도 제한, 세션 고정 공격 방지
- 파일 업로드: 확장자 화이트리스트 + 매직바이트 검증 + 파일명 UUID 재생성 + 크기 제한
- 송금: DB 트랜잭션 단위 잔액 검증, 음수 잔액 DB 제약조건으로 차단
- 보안 헤더: CSP, X-Frame-Options, X-Content-Type-Options 등
- 운영 모드에서 내부 오류/스택트레이스 미노출

자세한 내용은 [`docs/03_security_checklist.md`](docs/03_security_checklist.md)를 참고하세요.

## 6. 알려진 제한사항
- 데모 목적의 프로젝트로, 실제 결제(PG) 연동 없이 가상 포인트로 송금 기능을 시연합니다.
- SQLite는 단일 프로세스 배포를 전제로 하며, 다중 워커/분산 배포 시 PostgreSQL 등으로 전환 후 동시성 로직 재검증이 필요합니다 (`docs/04_maintenance.md` 참고).
