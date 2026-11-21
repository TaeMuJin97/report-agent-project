# Report Agent Basic 최종 프로젝트 수행 보고서

## 1. 프로젝트 개요

### 1.1 추진 배경 및 목표
본 프로젝트는 다양한 유형의 사내 업무 보고서를 체계적으로 저장·관리하고, 사용자가 자연어 질의만으로 유사 보고서를 탐색·요약할 수 있는 **AI 백엔드 서비스(Report Agent Basic)**를 설계·구현·검증하는 것을 목표로 수행되었습니다.

단순 CRUD 기능 구현에 그치지 않고, **정규화된 10개 테이블 관계형 스키마**, **로컬 E5 임베딩과 pgvector를 결합한 Hybrid 검색**, **트랜잭션 경계 분리를 통한 성능·정합성 확보**, 그리고 **자동화 테스트(pytest) 및 E2E API 시나리오 검증(Postman 30건)**까지 프로덕션 지향 백엔드 품질 기준을 충족하는 데 집중했습니다.

### 1.2 기술 스택
| 영역 | 사용 기술 | 선정 이유 및 역할 |
| :--- | :--- | :--- |
| **Backend Framework** | Python, FastAPI, Pydantic v2 | 타입 기반 요청/응답 계약 검증 및 자동 OpenAPI 문서화 |
| **Database & ORM** | PostgreSQL 18, `pgvector`, SQLAlchemy 2.0 | 관계형 무결성과 384차원 벡터 유사도 검색을 단일 DB에서 일관 처리 |
| **Embedding Model** | `intfloat/multilingual-e5-small` (CPU) | 한국어/다국어 문맥 이해도가 높고 로컬 CPU 환경에서 비용 없이 실행 가능 |
| **LLM Integration** | OpenAI Responses API | 보고서 원문(`content`) 기반 한국어 요약 및 구조화된 출력 검증 |
| **Testing & QA** | `pytest`, Postman Collection, `verify.sql` | 오프라인 계약 테스트, 실제 DB 통합 테스트, SQL 기반 데이터 무결성 교차 검증 |

---

## 2. 데이터 모델링 및 시스템 아키텍처

### 2.1 10개 테이블 스키마 및 뷰 구조
공통 메타데이터와 6종 보고서별 고유 필드를 분리하여 확장성과 무결성을 확보했습니다.

* **공통 관리 테이블 (4개)**
  * `report_categories`: 6종 카테고리 마스터 (`EXPERIMENT`, `DAILY_WORK`, `RESEARCH_PLAN`, `DATA_PROCESSING`, `PROMPT`, `MEETING`)
  * `reports`: 공통 보고서 식별자(`report_id`), `overview`, `revision`, `index_status`(`pending`/`ready`) 관리
  * `prompt_versions`: `PROMPT` 유형 보고서의 버전 이력 및 활성 버전 관리
  * `report_embeddings`: `Vector(384)` 임베딩, `model_name`, `indexed_revision`, SHA-256 `overview_hash` 저장
* **카테고리별 상세 테이블 (6개)**
  * `experiment_reports`, `daily_work_reports`, `research_plans`, `data_processing_reports`, `prompt_reports`, `meeting_minutes`
  * 모든 상세 테이블 및 임베딩 테이블은 `reports.report_id`를 외래키로 참조하며 `ON DELETE CASCADE`로 연결
* **조회 전용 뷰 (`report_read_view`)**
  * 6종 상세 테이블의 공통 표시 필드(`title`, `content`, `author`, `create_at`, `update_at`)와 카테고리 정보를 통합 조회하도록 구성

### 2.2 계층형 아키텍처
관심사 분리(SoC) 원칙에 따라 **API Router → Service → Repository / Integration → DB** 4계층으로 모듈화했습니다.

* `app/api/`: HTTP 요청 파싱, 상태 코드 반환, 전역 에러 계약 처리
* `app/services/`: 6종 상세 유효성 검증, 트랜잭션 제어, Hybrid 검색 정책, Agent Action Routing
* `app/repositories/`: SQLAlchemy 쿼리 및 응답 직렬화
* `app/integrations/`: E5 임베딩 어댑터(`embedding_client.py`) 및 OpenAI 호출 어댑터(`llm_client.py`)

---

## 3. 핵심 기술 구현 및 설계 포인트

### 3.1 외부 연산과 DB 트랜잭션 경계 분리
* **문제 인식**: 보고서 등록(`API-02`) 및 수정(`API-05`) 시 DB 트랜잭션 내부에서 E5 임베딩 모델(CPU 연산)을 호출하면 DB 커넥션과 행 잠금(Lock)을 불필요하게 오래 점유하게 됩니다.
* **해결 설계**:
  1. `create_report()`는 E5 벡터 연산(`encode_overview`)을 `with db.begin():` **바깥에서 먼저 완료**한 뒤, 트랜잭션 안에서 `reports` 저장 → `db.flush()`로 `report_id` 획득 → 상세 테이블(`PROMPT`는 v1 포함) 저장 → `report_embeddings` 저장을 단일 트랜잭션으로 확정합니다.
  2. `update_report()`는 기존 `overview`와 달라진 경우에만 읽기 트랜잭션을 닫고(`rollback()`) E5 벡터를 재계산한 뒤, `with_for_update()` 행 잠금을 획득하여 본문·`revision`·벡터·`overview_hash`를 원자적으로 갱신합니다.

### 3.2 E5 임베딩 정합성 및 일괄 색인 파이프라인
* E5 모델 규격에 맞춰 저장 시 `passage:` 접두어, 질의 시 `query:` 접두어를 강제하고 L2 정규화된 384차원 벡터를 생성했습니다.
* `scripts/index_overviews.py` 배치 스크립트를 구현하여 초기 시드 보고서 180건을 16건 단위 배치로 일괄 벡터화하고 `index_status = 'ready'`로 전환했습니다.
* `hashlib.sha256(overview.encode('utf-8')).hexdigest()`와 PostgreSQL 내장 `sha256()` 함수를 동일한 기준으로 맞춰, 개요 텍스트와 저장된 벡터 간의 불일치(`overview_hash_mismatch`)를 SQL 단에서 즉시 탐지할 수 있게 설계했습니다.

### 3.3 Keyword 우선 + pgvector Semantic Hybrid 검색 (`API-07`)
* 검색 대상은 `index_status = 'ready'`, `indexed_revision = revision`, 동일 모델명을 만족하는 유효 벡터로 엄격히 필터링했습니다.
* **1단계 (Keyword 우선)**: 정확한 키워드 매칭이 필요한 업무 보고서 특성을 고려해 제목·개요 기반 `ILIKE` 매칭을 우선 탐색합니다.
* **2단계 (Semantic Fallback)**: 키워드 매칭이 없을 경우 `pgvector` 코사인 거리(`<=>`) 기준 최상위 1건을 추출하고, 유사도 임계값(0.75) 미만인 무관한 질의(예: *"화성의 낙타 전용 미용실"*)는 오류 대신 정상 응답(`200 OK`, `status="not_found"`, `best_match=null`)으로 처리했습니다.

### 3.4 OpenAI 원문 요약 (`API-08`) 및 Basic Action Router Agent (`API-09`)
* 요약 기능은 단순 개요가 아닌 보고서 원문(`content`)을 OpenAI Responses API로 전달하여 한국어 요약과 출처 메타데이터를 반환하며, 외부 서비스 오류·시간 초과 시 표준 에러 코드(`503`/`504`)로 격리했습니다.
* Basic Agent는 명시적 Action Router 패턴으로 `get_report`, `find_report`, `summarize_report` 3가지 액션을 분기 실행하여 예측 가능하고 테스트 가능한 에이전트 동작을 구현했습니다.

---

## 4. 구현 API 및 단계별 검증 결과

### 4.1 자동화 테스트 (`pytest`) 결과
| 테스트 구분 | 실행 명령어 | 검증 내용 | 결과 |
| :--- | :--- | :--- | :---: |
| **오프라인 계약 테스트** | `python -m pytest -q -m "not live"` | 10개 ORM 테이블 구조, API 요청/응답 스키마, 6종 상세 검증 규칙 | **12 passed** |
| **실제 DB 통합 테스트** | `RUN_LIVE_TESTS=1 python -m pytest -q -m live` | 실제 PostgreSQL 연결, 180건 이상 보고서·벡터 수량 및 384차원 확인 | **1 passed** |

### 4.2 Postman Basic E2E 시나리오 (30개 요청 전수 통과)
| 검증 구간 | 주요 테스트 항목 | 검증 결과 |
| :--- | :--- | :---: |
| **01 Basic 무료 경로** | Health(`API-00`), 카테고리 6종(`API-01`), 시드 조회·목록 검색(`API-03·04`), 실험 보고서 등록(`201`)·수정(`revision=2`)·재조회, Hybrid 검색(`found`/`not_found`), Agent 무료 액션 2건 | **PASS** |
| **04 오류 사례** | 공백 제목 `422`, 존재하지 않는 ID `404 (REPORT_NOT_FOUND)`, `limit=101` `422`, 공백 검색어 `422` | **PASS** |
| **05 추가 5종 등록** | `DAILY_WORK`, `RESEARCH_PLAN`, `DATA_PROCESSING`, `PROMPT`(v1 자동 생성), `MEETING` 각 `201 Created` 확인 | **PASS** |
| **06 실제 LLM 유료 요청** | `allow_paid_llm=true` 전환 후 `API-08`(생성 보고서 요약), `API-09`(시드 1번 Agent 요약) 각 1회 실호출 검증 후 `false` 원복 | **PASS** |
| **07~08 생성 데이터 정리** | 테스트 생성 보고서 6건 전체 `DELETE (204)` → 재조회 `404` → 검색 결과 제외 확인 | **PASS** |

### 4.3 PostgreSQL 최종 기준 상태 검증 (`common/sql/verify.sql`)
모든 E2E 테스트 및 정리 작업 완료 후, 시드 기준 초기 무결성 상태로 완벽 복귀했음을 확인했습니다.

| 검증 항목 | 기준값 | 최종 확인 결과 | 비고 |
| :--- | :---: | :---: | :--- |
| `report_categories` | 6 | **6** | 6종 카테고리 정상 |
| `reports` / 6종 상세 테이블 | 180 / 각 30 | **180 / 각 30** | 테스트 잔여 데이터 0건 |
| `prompt_versions` | 60 | **60** | 시드 30건 × 2버전 일치 |
| `report_embeddings` | 180 | **180** | 전수 384차원 E5 벡터 완료 |
| `index_status` (`ready` / `pending`) | 180 / 0 | **180 / 0** | 미색인 보고서 0건 |
| 전체 무결성 검사 13개 항목 (`error_count`) | 모두 0 | **ALL 0** | 차원·모델·해시·revision·뷰·활성버전 100% 일치 |

---

## 5. 트러블슈팅 및 품질 관리 기록

1. **pgAdmin 다중 `SELECT` 스크립트 확인 방식 파악**
   * **현상**: `verify.sql` 전체를 `F5`로 실행했을 때 마지막 `SELECT` 표 1개만 표시되어 수량표와 무결성 표 갱신 여부가 보이지 않음.
   * **해결**: pgAdmin Query Tool의 단일 결과 출력 특성을 파악하고, 검증 블록별 드래그 선택 실행 및 핵심 지표 통합 조회 쿼리를 병행하여 전수 무결성을 정확히 검증함.
2. **테스트 데이터 삭제 후 `404 REPORT_NOT_FOUND` 및 환경변수 동기화**
   * **현상**: 개발 확인용 보고서(ID 181) 삭제 후 재요청 시 `404 REPORT_NOT_FOUND` 발생.
   * **해결**: DB 실데이터와 Postman 환경변수(`created_report_id`) 간 상태 차이를 확인하고, 삭제 검증 요청(`API-03 · 삭제 후 조회 404`)을 통해 생성·스냅샷 변수를 자동 초기화한 뒤 180건 기준 상태를 확정함.
3. **테스트 설정 파일(`pytest.ini`) 경로 정정**
   * **현상**: `pytest.ini`를 `tests/` 하위 폴더에 둘지 프로젝트 루트에 둘지 혼선 발생.
   * **해결**: `testpaths = tests` 기준 경로 원리를 확인해 프로젝트 최상위 루트에 배치함으로써 오프라인·라이브 마커 테스트가 일관되게 실행되도록 정비함.
4. **비용·보안 안전장치 준수**
   * `.gitignore`를 통해 `.env`(OpenAI API 키, DB 비밀번호)와 `.venv`가 Git에 추적되지 않도록 격리(`git ls-files .env .venv` 결과 공백 확인).
   * Postman `allow_paid_llm` 가드 변수를 도입해 유료 LLM 호출을 최종 검증 시 정확히 2회로 통제 후 `false`로 원복.

---

## 6. 회고 및 향후 확장 방향

### 6.1 프로젝트 회고
이번 프로젝트를 통해 단순히 "동작하는 API"를 넘어, **관계형 DB 무결성과 AI 모델(임베딩·LLM)의 비결정적·고비용 특성을 어떻게 안전하게 결합할 것인가**를 체득했습니다. 특히 트랜잭션 경계 분리, SHA-256 해시 기반 벡터 정합성 검증, 그리고 오프라인 계약 테스트와 실 DB 통합 테스트의 분리 설계는 서비스 규모가 커질수록 필수적인 백엔드 기본기임을 실감했습니다.

### 6.2 향후 고도화 계획
* **Prompt 버전 관리 API 확장**: `PROMPT` 보고서의 새 버전 추가 및 활성 버전 롤백/전환 API 구현.
* **비동기 색인 및 검색 고도화**: 대용량 보고서 등록 시 메시지 큐 기반 비동기 임베딩 워커 도입 및 `pgvector` HNSW 인덱스 튜닝.
* **관측성(Observability) 강화**: LLM 호출 지연 시간·토큰 사용량 로깅 및 구조화된 감사 로그 추가.