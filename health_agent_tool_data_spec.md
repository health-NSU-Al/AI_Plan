# 복약·측정 에이전트 — 도구·데이터 명세서 (Tool & Data Spec)

> C. 도구·MCP, D. 데이터(ERD). 아키텍처·파일 구조는 `health_agent_architecture_spec.md`.
> 이름·인자·컬럼은 초안이며 구현하면서 확정한다.

---

## C. 도구·MCP
> 웹 API 대신 LLM이 호출하는 **도구**와 **MCP 서버 도구**를 정의한다.
> 이름은 "동사 + 대상", 반환은 모델이 그대로 읽을 **문장**, 오류도 문장으로 돌려준다 (`@tool(parse_docstring=True)`, 수업 3장 14단계).
> **환자 ID는 어떤 도구의 인자에도 없다.** 서버 변수로 고정한다.
> 인자 이름·형식은 초안이며 구현하면서 확정한다.

### 1. 도구 목록

| 도구 | 종류 | 위험도 등급 (3장 17단계) | 처리 |
|---|---|---|---|
| `get_today_medications` | 로컬 | 읽기 | 자유 |
| `check_medication` | 로컬 | 되돌릴 수 있는 쓰기 | 실행 + 기록 |
| `search_health_docs` | RAG | 읽기 | 자유 |
| `get_blood_pressure` | MCP | 읽기 | 자유 |
| `get_glucose` | MCP | 읽기 | 자유 |
| `get_heart_rate` | MCP | 읽기 | 자유 |
| `make_report` | 로컬 | 되돌릴 수 있는 쓰기 | 실행 + 기록 |
| 보호자 알림 | 그래프 노드 | 외부 전송 | **LLM 도구로 노출하지 않음** |

### 2. 로컬 도구

#### 2.1 `get_today_medications`
- 인자: 없음
- 반환 예: "오늘 아침 약 3종 중 3종 복용, 저녁 약 2종 아직 체크 안 됨."

#### 2.2 `check_medication`
- 인자: `slot` — `"morning"` | `"evening"`
- 규칙: 시간대별 1회만 기록. 이미 체크된 시간대면 저장하지 않고 경고 문장 반환.
- 반환 예: "아침 약 복용을 기록했어요." / "아침 약은 이미 08:02에 체크되어 있어요. 두 번 드시지 않도록 주의하세요."

#### 2.3 `make_report`
- 인자: `days` — 조회 기간(일)
- 처리: 통계는 코드로 계산, 그래프·질문 목록 포함 PDF 생성 (한글 TTF, "가상 데이터" 표기)
- 반환: 생성된 파일 경로를 담은 문장

### 3. RAG 도구

#### 3.1 `search_health_docs`
- 인자: `query`
- 처리: FAISS k=3 → 점수가 임계값 미만이면 "확인할 수 없어요" (`query_rag_safe` 방식)
- 반환: 근거 내용 + 출처 문서명·쪽수

| 설정 | 값 |
|---|---|
| 로더 | `PyPDFLoader` |
| 청크 | 500자 / 겹침 50 |
| 임베딩 | `HuggingFaceEmbeddings('BAAI/bge-m3', normalize_embeddings=True)` |
| 인덱스 | FAISS `MAX_INNER_PRODUCT` |

문서 출처 (공공 자료만):
- 의약품안전나라 허가정보 — 가상 처방 5종 (전문의약품이라 e약은요 API에는 없음)
- 질병관리청 국가건강정보포털 — 고혈압·당뇨·부정맥 안내, 가정혈압 측정법

### 4. MCP 가상 측정기 서버 (`mcp_server/device_server.py`)

- FastMCP로 구현, 뒤에 SQLite 가상 데이터
- 에이전트 쪽: `langchain-mcp-adapters`의 `MultiServerMCPClient` → `get_tools()` → 로컬 도구와 한 리스트
- 비동기 전용 → `ainvoke` 사용

| 도구 | 인자 | 반환 |
|---|---|---|
| `get_blood_pressure` | `days` | 기간별 수축기·이완기·맥박 |
| `get_glucose` | `days` | 기간별 혈당 + 측정 상황(공복/식후) |
| `get_heart_rate` | `days` | 심박수 + 불규칙 리듬 알림 이벤트 |

> 실제 기기 API 서버로 바꿔 끼워도 에이전트 코드는 그대로다 — 발표의 "기술 선택 이유" 핵심 문장.

### 5. 알림 (`notify/discord_notify.py`)

- 디스코드 웹훅: `requests.post(url, json={"content": ...})`, 봇 토큰 불필요
- 채널: `#복약알림`(본인), `#보호자알림`(가상 보호자)
- 메시지는 최소 정보: "아침 약 드실 시간이에요", "아침 약 확인이 안 됐어요" — 수치·약 이름·질환명 금지
- 전송 실패는 기록하고 Gradio 토스트로 대체

---

## D. 데이터(ERD)
> 대상 DB: SQLite (`data/health.db`), 생성은 `data/generate_fake_data.py`.
> 가상 사용자 1명이라 환자 테이블 없이 단일 사용자를 전제한다 (환자 ID는 서버 변수).
> 컬럼은 기능 명세가 요구하는 최소 초안이며 구현하면서 확정한다.

### 1. 테이블 목록

| 테이블명 | 설명 | 쓰는 쪽 |
|---|---|---|
| `medication` | 가상 처방 약 (아침 3종·저녁 2종) | 로컬 도구, RAG 문서 매칭 |
| `medication_log` | 시간대별 복용 기록 | `check_medication`, 플로우 A |
| `blood_pressure` | 혈압·맥박 측정값 | MCP 서버, 플로우 B |
| `glucose` | 혈당 측정값 | MCP 서버, 플로우 B |
| `heart_rate` | 심박수·불규칙 리듬 이벤트 | MCP 서버 |
| `notification_log` | 알림 발송 기록 | 플로우 A·B |

### 2. 테이블 상세

#### 2.1 `medication`
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | INTEGER PK | |
| name | TEXT | 약 이름 (가상 처방 5종, 미확정) |
| slot | TEXT | `morning` / `evening` |
| description | TEXT | 화면 표시용 짧은 설명 (예: "하얀 원형 정제") |

#### 2.2 `medication_log`
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | INTEGER PK | |
| date | TEXT | `YYYY-MM-DD` |
| slot | TEXT | `morning` / `evening` |
| status | TEXT | `taken` / `missed` |
| checked_at | TEXT | 체크 시각 |

> `UNIQUE(date, slot)` — 시간대별 1회만 기록해 중복 복용 체크를 막는다.

#### 2.3 `blood_pressure`
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | INTEGER PK | |
| measured_at | TEXT | 측정 시각 |
| systolic | INTEGER | 수축기 (mmHg) |
| diastolic | INTEGER | 이완기 (mmHg) |
| pulse | INTEGER | 맥박 (bpm) |
| judgement | TEXT | 규칙 엔진 결과: `normal` / `out_of_range` / `suspect` / `danger` |

#### 2.4 `glucose`
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | INTEGER PK | |
| measured_at | TEXT | |
| value | INTEGER | mg/dL |
| context | TEXT | `fasting` / `post_meal` |
| judgement | TEXT | 2.3과 동일 |

#### 2.5 `heart_rate`
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | INTEGER PK | |
| measured_at | TEXT | |
| bpm | INTEGER | |
| irregular_alert | INTEGER | 워치 불규칙 리듬 알림 여부 (0/1). 판정하지 않고 기록만 |

#### 2.6 `notification_log`
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | INTEGER PK | |
| sent_at | TEXT | |
| channel | TEXT | `patient` / `guardian` / `toast` |
| message | TEXT | 최소 정보 문구 |
| success | INTEGER | 디스코드 전송 성공 여부 (0/1) |

### 3. 관계 요약

- `medication_log.slot` ↔ `medication.slot` (시간대 단위로 묶어 체크)
- 측정 테이블 3개는 서로 독립, 시각으로만 정렬·집계
- 단일 사용자라 사용자 FK 없음

### 4. 추후 고도화 시 고려

- 다중 사용자 → `patient` 테이블 + 모든 테이블에 FK (도구 인자에는 여전히 노출하지 않음)
- 약별 개별 체크 (현재는 시간대 단위)
