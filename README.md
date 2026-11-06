# Ecobot - 생활 속 환경 실천 안내 챗봇

> 이 저장소는 SK네트웍스 Family AI 캠프 3차 팀 프로젝트([원본](https://github.com/SKNETWORKS-FAMILY-AICAMP/SKN32-3rd-3Team))의 포크입니다. 4차에서 이어서 개선한 버전은 [SKN32-4th-3Team](https://github.com/SKNETWORKS-FAMILY-AICAMP/SKN32-4th-3Team)에 있습니다.

## 제가 맡은 일 (정세환)

- **3차 — 검색 파이프라인·성능평가**: 법령은 조항 단위, 안내문은 목록 항목 단위로 청킹하고, 지역 조건과 3단계 검색 쿼터로 다른 지역 문서가 섞이지 않게 했습니다. 근거 문서가 없으면 답하지 않는 기준을 평가셋으로 정했습니다. ([evals/run_eval.py](evals/run_eval.py))
- **4차 — 데이터·검색**: 데이터 수집·정제, 지자체 민원 연결, 검색 파이프라인 실험을 맡았습니다.

## 평가로 정한 것

| 판단 | 근거 |
|---|---|
| 답변 거부 기준 | 답해야 할 질문과 거부해야 할 질문을 함께 넣은 30문항: 통과율 93.3%, 거부 정확도 100% |
| 평가셋 확장과 정답 교정 | 3차와 같은 조건으로 비교하려고 기존 30문항을 유지한 채 100문항으로 확장, 모범답안을 원문과 대조해 지역 지정 오류 13문항 교정 |
| 검색 방식 | 통과율은 BM25가 94%로 1위였지만 환각률 8%. 환각 방지가 핵심 요구라 환각률이 가장 낮은 하이브리드(통과율 91%, 환각률 6%)를 채택. R@5 0.607 → 0.730 |

자세한 수치: [4차 테스트 계획 및 결과 보고서](https://github.com/SKNETWORKS-FAMILY-AICAMP/SKN32-4th-3Team/blob/main/%EC%82%B0%EC%B6%9C%EB%AC%BC/%ED%85%8C%EC%8A%A4%ED%8A%B8%20%EA%B3%84%ED%9A%8D%20%EB%B0%8F%20%EA%B2%B0%EA%B3%BC%20%EB%B3%B4%EA%B3%A0%EC%84%9C.md)

---

RAG 기반 LLM 챗봇으로, 지역별(서울·천안·부산 남구) 분리배출 가이드와 환경 법령을 근거로 답변합니다.

## 기술 스택

- **Backend** : FastAPI + SQLAlchemy + MySQL
- **LLM** : Google Gemini (gemini-2.5-flash)
- **벡터DB** : FAISS (faiss-cpu)
- **임베딩** : Gemini Embedding / Local(개발용)
- **Frontend** : 순수 HTML + CSS + JS (static/)
- **인증** : JWT (httpOnly Cookie)

## 프로젝트 구조

```
SKN32-3rd-3Team/
├── app/
│   ├── main.py              # FastAPI 진입점
│   ├── models.py            # SQLAlchemy ORM 모델
│   ├── schemas.py           # Pydantic 스키마
│   ├── database.py          # DB 엔진/세션
│   ├── core/
│   │   ├── config.py        # 환경변수 설정 (pydantic-settings)
│   │   └── security.py      # JWT + bcrypt
│   ├── routers/
│   │   ├── api.py           # 인증 + 문서 CRUD
│   │   └── rag.py           # RAG 채팅 엔드포인트
│   └── services/
│       ├── rag_service.py           # RAG 오케스트레이터
│       ├── chunk_service.py         # 문서 청킹
│       ├── embedding_service.py     # 임베딩 생성
│       ├── vector_store_service.py  # FAISS 벡터 저장소
│       └── gemini_service.py        # Gemini 답변 생성
├── static/                  # 프론트엔드 (Ecobot UI)
│   ├── index.html
│   ├── style.css
│   └── app.js
├── data/
│   ├── guide/               # 내부 문서 (분리배출 가이드)
│   └── laws/                # 외부 문서 (법령 원문)
├── requirements.txt
├── .env                     # 환경변수 (직접 생성)
└── run.sh                   # 실행 스크립트
```

## 실행 방법

### 1. 저장소 클론

```bash
git clone <repo-url>
cd SKN32-3rd-3Team
```

### 2. 가상환경 생성 및 패키지 설치

```bash
python -m venv venv

# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate

pip install -r requirements.txt
```

### 3. 환경변수 설정

프로젝트 루트에 `.env` 파일을 생성합니다.

```env
# DB
DATABASE_URL=mysql+pymysql://<user>:<password>@<host>:3306/<dbname>

# 인증
SECRET_KEY=your-secret-key-here

# Gemini
GEMINI_API_KEY=your-gemini-api-key

# RAG (선택 — 기본값이 있으므로 필요한 것만 오버라이드)
RAG_SOURCE=db              # db 또는 files
EMBEDDING_BACKEND=gemini   # gemini 또는 local(개발용)
```

### 4. MySQL 데이터베이스 생성

```sql
CREATE DATABASE ecora CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

서버 최초 실행 시 테이블이 자동 생성됩니다 (`Base.metadata.create_all`).

### 5. 서버 실행

```bash
# 방법 1: run.sh 사용 (Linux/macOS)
chmod +x run.sh
./run.sh

# 방법 2: 직접 실행
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 6. 접속

브라우저에서 `http://localhost:8000` 으로 접속합니다.

## RAG 문서 인덱싱

DB 없이 로컬 파일로 테스트하려면:

```env
RAG_SOURCE=files
EMBEDDING_BACKEND=local
```

설정 후 `data/docs/` 폴더에 txt/md/pdf 파일을 넣고, 서버 실행 후 인덱스를 빌드합니다:

```bash
curl -X POST http://localhost:8000/api/rag/rebuild
```

## API 엔드포인트

| Method | Path | 설명 |
|--------|------|------|
| POST | `/api/auth/register` | 회원가입 |
| POST | `/api/auth/login` | 로그인 (JWT 쿠키 발급) |
| POST | `/api/auth/logout` | 로그아웃 |
| GET | `/api/auth/me` | 현재 사용자 정보 |
| POST | `/api/chat` | RAG 챗봇 질문 |
| POST | `/api/rag/rebuild` | 벡터 인덱스 재구축 |
| GET | `/api/rag/status` | 인덱스 상태 확인 |
| GET | `/api/health` | 헬스체크 |

## 팀원 및 역할

| 역할 | 담당 |
|------|------|
| A | 기반 인프라 / 인증 (models, schemas, security) |
| B | 프론트엔드 + 문서 CRUD |
| C | 데이터 수집 / 벡터DB |
| D | LLM·RAG 파이프라인 + 관리자 대시보드 |
