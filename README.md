# Report Agent Basic

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115+-009688?logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_18-pgvector-4169E1?logo=postgresql&logoColor=white)
![Embedding](https://img.shields.io/badge/Embedding-multilingual--e5--small-FF6F00)
![OpenAI](https://img.shields.io/badge/LLM-OpenAI_Responses_API-412991?logo=openai&logoColor=white)

> **6종 업무 보고서 CRUD · E5 기반 pgvector Hybrid 검색 · OpenAI 원문 요약 · 명시적 Action Router Agent**를 제공하는 FastAPI 백엔드 프로젝트입니다.

> 📄 **상세 설계·검증 결과·트러블슈팅 정리**: 👉 [**최종 프로젝트 수행 보고서 (`FINAL_REPORT.md`)**](./FINAL_REPORT.md)

---

## 1. 프로젝트 개요

조직 내에서 생성되는 6가지 유형의 업무 보고서를 정규화된 10개 테이블 스키마로 관리하고, 보고서 개요(`overview`)를 로컬 임베딩 모델(`intfloat/multilingual-e5-small`, 384차원)로 벡터화하여 **Keyword + Semantic Hybrid 검색**과 **LLM 기반 원문 요약**을 단일 Agent 인터페이스로 제공합니다.

### 지원 보고서 카테고리 (6종)
| 코드 | 카테고리명 | 상세 테이블 |
| :--- | :--- | :--- |
| `EXPERIMENT` | 실험 보고서 | `experiment_reports` |
| `DAILY_WORK` | 일일 업무 보고서 | `daily_work_reports` |
| `RESEARCH_PLAN` | 연구 계획서 | `research_plans` |
| `DATA_PROCESSING` | 데이터 처리 보고서 | `data_processing_reports` |
| `PROMPT` | 프롬프트 보고서 | `prompt_reports` (+ `prompt_versions`) |
| `MEETING` | 회의록 | `meeting_minutes` |

---

## 2. 핵심 설계 포인트

* **트랜잭션 경계 분리 (Transaction Boundary Separation)**
  * 보고서 등록(`POST`)·수정(`PUT`) 시 CPU 연산 비용이 큰 E5 임베딩 계산을 DB 트랜잭션(`with db.begin():`) 바깥에서 선행 처리하여 DB 커넥션 점유 시간을 최소화하고 원자적 저장을 보장합니다.
* **데이터·벡터 무결성 보장**
  * 보고서 수정 시 `overview`가 변경된 경우에만 행 잠금(`with_for_update`)을 통해 벡터를 재계산하고 `revision` 및 SHA-256 `overview_hash`를 동기화합니다.
  * 공통 `reports` 행 삭제 시 `ON DELETE CASCADE`로 상세 테이블·프롬프트 버전·임베딩 벡터를 일괄 정리합니다.
* **Hybrid 검색 (Keyword 우선 + Semantic Cosine 유사도)**
  * `index_status = 'ready'`, 최신 `revision`, 동일 임베딩 모델 조건을 만족하는 보고서만 대상으로 검색합니다.
  * 제목·개요 기반 Keyword(`ILIKE`) 검색을 우선 수행하고, 미매칭 시 `pgvector` 코사인 거리(`<=>`) 기반 Semantic 1위 결과에 유사도 임계값(0.75)을 적용합니다.
* **신뢰성 있는 LLM 요약 & Basic Agent**
  * OpenAI Responses API로 보고서 원문(`content`)을 요약하며 외부 API 장애·타임아웃 시 일관된 에러 계약(`503`/`504`)으로 응답합니다.
  * 명시적 Action Router(`get_report`, `find_report`, `summarize_report`)를 통해 조회·검색·요약 워크플로우를 제어합니다.

---

## 3. 시스템 아키텍처 & 디렉터리 구조

```text
[Client / Postman / Swagger]
             │
             ▼
    ┌─────────────────┐
    │   app/api/      │  FastAPI Routers (API-00 ~ API-09)
    └────────┬────────┘
             ▼
    ┌─────────────────┐      ┌──────────────────────────────┐
    │  app/services/  │─────▶│      app/integrations/       │
    │  (CRUD/Search/  │      │ • embedding_client (E5 CPU)  │
    │   Agent Logic)  │      │ • llm_client (OpenAI API)    │
    └────────┬────────┘      └──────────────────────────────┘
             ▼
    ┌─────────────────┐
    │app/repositories/│  SQLAlchemy ORM (10 Tables + Read View)
    └────────┬────────┘
             ▼
    ┌─────────────────┐
    │  PostgreSQL 18  │  pgvector (Vector 384) & SHA-256 Integrity
    └─────────────────┘
```

```text
report-agent-project/
├── app/
│   ├── api/                 # FastAPI 라우터 (health, reports, search, agent)
│   ├── core/                # 환경 설정, 에러 코드 및 예외 핸들러
│   ├── db/                  # DB 세션 및 SQLAlchemy ORM 모델 (10개 테이블)
│   ├── integrations/        # E5 임베딩 클라이언트 & OpenAI LLM 클라이언트
│   ├── repositories/        # 보고서 조회·직렬화 Repository
│   ├── schemas/             # Pydantic 요청/응답 스키마 계약
│   ├── services/            # 보고서 CRUD, Hybrid 검색, Basic Agent 비즈니스 로직
│   └── main.py              # FastAPI 앱 진입점
├── common/
│   └── sql/                 # schema.sql, seed.sql (180건), verify.sql
├── scripts/
│   └── index_overviews.py   # 전체 보고서 개요 E5 일괄 벡터화 스크립트
├── tests/
│   ├── test_contracts.py        # 오프라인 모델·API 계약 단위 테스트 (12건)
│   └── test_live_integration.py # 실제 DB·pgvector 통합 테스트 (1건)
├── pytest.ini               # pytest 경로 및 live 마커 설정
├── README.md                # 프로젝트 실행 및 구조 안내 (현재 문서)
└── FINAL_REPORT.md          # 최종 프로젝트 수행 보고서
```

---

## 4. API 명세 요약 (Basic 10종)

| ID | Method | Endpoint | 설명 | 상태 코드 |
| :--- | :---: | :--- | :--- | :---: |
| **API-00** | `GET` | `/health` | 서버 및 DB 연결 상태 확인 | `200` |
| **API-01** | `GET` | `/api/v1/categories` | 보고서 카테고리 6종 목록 조회 | `200` |
| **API-02** | `POST` | `/api/v1/reports` | 6종 보고서 등록 + 실시간 E5 임베딩 저장 | `201` |
| **API-03** | `GET` | `/api/v1/reports/{report_id}` | 보고서 단건 상세 조회 | `200` / `404` |
| **API-04** | `GET` | `/api/v1/reports` | 보고서 목록 조회 (카테고리·제목 필터, 페이징) | `200` |
| **API-05** | `PUT` | `/api/v1/reports/{report_id}` | 보고서 전체 수정 (개요 변경 시 E5 재계산·`revision` 증가) | `200` / `404` |
| **API-06** | `DELETE` | `/api/v1/reports/{report_id}` | 보고서 삭제 (`ON DELETE CASCADE`) | `204` / `404` |
| **API-07** | `POST` | `/api/v1/search/similar` | Hybrid 유사 보고서 검색 (Keyword 우선 + pgvector) | `200` |
| **API-08** | `POST` | `/api/v1/reports/{report_id}/summarize` | OpenAI 기반 보고서 원문 요약 | `200` / `404` |
| **API-09** | `POST` | `/api/v1/agent/run` | Basic 명시적 Action Router Agent 실행 | `200` |

---

## 5. 실행 방법 (Quick Start)

### 1) 가상환경 생성 및 패키지 설치
```bash
python -m venv .venv
source .venv/Scripts/activate   # Windows Git Bash 기준
pip install -r requirements.txt
```

### 2) 환경 변수 설정 (`.env`)
프로젝트 루트에 `.env` 파일을 생성하고 DB 및 OpenAI 설정을 입력합니다. *(주의: `.env`는 `.gitignore`로 관리되며 저장소에 커밋하지 않습니다.)*
```env
DATABASE_URL=postgresql+psycopg://postgres:<PASSWORD>@localhost:5432/report_agent
EMBEDDING_MODEL=intfloat/multilingual-e5-small
OPENAI_API_KEY=sk-...
LLM_MODEL=gpt-4o-mini
```

### 3) 데이터베이스 초기화 및 180건 일괄 벡터 색인
1. pgAdmin 또는 `psql`에서 `common/sql/schema.sql` → `common/sql/seed.sql` 순서로 실행합니다.
2. 시드 보고서 180건의 개요를 E5 모델로 일괄 색인합니다.
```bash
python scripts/index_overviews.py
```

### 4) 서버 실행 및 API 문서 확인
```bash
python -m app.main
```
* Swagger UI: `http://127.0.0.1:8000/docs`

---

## 6. 테스트 및 무결성 검증

```bash
# 1. 오프라인 계약 단위 테스트 (DB/E5/LLM 호출 없음 → 12 passed)
python -m pytest -q -m "not live"

# 2. 실제 PostgreSQL · pgvector 통합 테스트 (유료 LLM 호출 없음 → 1 passed)
RUN_LIVE_TESTS=1 python -m pytest -q -m live
```

* **DB 정합성 검증**: `common/sql/verify.sql` 실행 시 시드 기준 180건(`ready=180`, `report_embeddings=180`) 및 전체 무결성 검사 `error_count = 0` 확인.
* **E2E 시나리오 검증**: Postman Basic Collection 30개 요청(무료 CRUD·검색·Agent, 오류 422/404, 추가 5종 등록, 유료 LLM 요약 2건, 테스트 데이터 정리) 전 항목 통과.