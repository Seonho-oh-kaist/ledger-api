GitHub: https://github.com/Seonho-oh-kaist/ledger-api
Render: https://ledger-api-52ax.onrender.com

# 가계부 API (ledger-api)

FastAPI + SQLAlchemy로 만든 가계부 REST API입니다. 계좌·카테고리·거래 데이터를 클라우드 PostgreSQL(Supabase)에 저장하고, Render에 배포해 인터넷 주소에서 같은 데이터를 조회할 수 있습니다.

```
요청 → FastAPI → SQLAlchemy(ORM) → PostgreSQL (Supabase)
```

## 기술 스택

| 구분 | 사용 기술 |
|---|---|
| 웹 프레임워크 | FastAPI, Uvicorn |
| ORM | SQLAlchemy 2.0 (`Mapped` / `mapped_column`) |
| DB | Supabase (PostgreSQL, Session pooler) |
| DB 드라이버 | psycopg 3 |
| 설정 | python-dotenv (`.env`) |
| 배포 | GitHub → Render |

## 프로젝트 구조

```
ledger-api/
├── main.py            # 앱 생성, 테이블 생성, API 경로
├── database.py        # Engine · Session · Base · get_db
├── models.py          # 테이블 정의 (Account, Category, Transaction)
├── schemas.py         # API 입출력 형식 (Pydantic)
├── requirements.txt   # 패키지 목록 (배포 시 재현용)
├── .gitignore         # .env, .venv 등 제외
└── README.md
```

## 데이터베이스 스키마

```
accounts (1) ──< transactions (N) >── (1) categories
```

| 테이블 | 주요 컬럼 |
|---|---|
| `accounts` | `id` (PK), `name`, `balance` |
| `categories` | `id` (PK), `name` (UNIQUE), `kind` (income/expense) |
| `transactions` | `id` (PK), `account_id` (FK → accounts), `category_id` (FK → categories, NULL 허용), `amount`, `memo`, `occurred_at` |

- 계좌를 삭제하면 해당 계좌의 거래도 함께 삭제됩니다 (`cascade="all, delete-orphan"`).
- `occurred_at`은 DB 서버의 현재 시각으로 자동 입력됩니다 (`server_default=func.now()`).

## API 엔드포인트

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/accounts` | 계좌 생성 |
| GET | `/accounts` | 계좌 목록 조회 |
| GET | `/accounts/{account_id}` | 계좌 단건 조회 |
| GET | `/accounts/{account_id}/detail` | 계좌 + 거래 목록 (중첩 응답) |
| POST | `/transactions` | 거래 생성 (계좌가 없으면 404) |
| GET | `/stats/by-category` | 카테고리별 지출 합계·건수 (GROUP BY) |

실행 후 `/docs`(Swagger UI)에서 모든 경로를 직접 호출해 볼 수 있습니다.

## 로컬 실행 방법

**1. 가상환경 만들기 & 패키지 설치**

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
```

**2. `.env` 파일 만들기** (프로젝트 루트, Git에는 올리지 않음)

```
DATABASE_URL=postgresql+psycopg://postgres.<프로젝트ref>:<비밀번호>@aws-<클러스터>-<region>.pooler.supabase.com:5432/postgres
```

- Supabase 대시보드의 **Connect → Session pooler** 문자열을 사용합니다.
- 맨 앞을 `postgresql://` 대신 `postgresql+psycopg://`로 바꿉니다.
- 값을 지정하지 않으면 기본값(`sqlite:///./ledger.db`)으로 동작합니다.

**3. 서버 실행**

```bash
uvicorn main:app --reload
```

브라우저에서 http://127.0.0.1:8000/docs 를 엽니다.

## 배포 (Render)

| 항목 | 값 |
|---|---|
| Build Command | `pip install -r requirements.txt` |
| Start Command | `uvicorn main:app --host 0.0.0.0 --port $PORT` |
| Environment Variable | `DATABASE_URL` = Supabase Session pooler 연결 문자열 (`postgresql+psycopg://…`) |

- 코드는 로컬과 동일하며, `.env` 대신 Render 환경변수로 DB 연결 정보를 넣습니다.
- 무료 플랜은 잠들었다 깨어날 때 첫 요청이 30~60초 느릴 수 있습니다 (콜드 스타트).
- `main` 브랜치에 push하면 자동으로 재배포됩니다.

---

## 실습 기록

### ① 결과 확인

- Supabase Table Editor의 `accounts` / `transactions` 테이블:

  ![Supabase Table Editor](images/supabase-table-editor.png)

- Render 배포 주소 `/docs`의 `GET /accounts` 실행 결과:

  ![Render GET /accounts](images/render-get-accounts.png)

### ② 핵심 개념 되새김

- **계좌·거래를 두 테이블로 나눈 이유 (1:N):** 한 계좌에는 거래가 여러 건 생기므로, 계좌 정보는 `accounts`에 한 번만 저장하고 거래는 `account_id`(외래키)로 가리키게 했다. 이렇게 하면 계좌 이름이 바뀌어도 한 행만 고치면 되고, 존재하지 않는 계좌의 거래는 DB가 막아 준다.
- **모델 클래스와 테이블의 대응:** `class Account(Base)` 하나가 `accounts` 테이블 하나이고, 클래스의 속성이 컬럼, 인스턴스 하나가 행 하나에 대응한다. `Mapped[str | None]`처럼 타입 표기가 `NULL` 허용 같은 제약 조건이 된다.
- **접속 문자열을 `.env`로 분리하는 이유:** 연결 문자열에 DB 비밀번호가 들어 있어 코드에 쓰면 GitHub에 공개된다. `.env`를 `.gitignore`로 제외하고, 로컬·배포 환경마다 값만 바꾸면 코드는 그대로 둘 수 있다.

### ③ 자유 로그

**오늘 배운 것**

- (직접 작성)

**막힌 곳과 푼 과정**

- 없음(실습 완료)

**아직 안 풀린 것**

- 없음(실습 완료)

**AI 활용**

- README 작성 외 실습워크북 내용으로 실습 완료
