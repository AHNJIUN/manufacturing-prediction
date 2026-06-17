# `manufacturing_agent.ipynb` 구성 가이드

> 제조업 AI Agent를 **LangGraph 멀티에이전트 + Context Engineering + Gate Control** 구조로 구현한 실행 가능한 노트북.
> 설계 원본: [`README.md`](./README.md) · 보완 과제: [`IMPROVEMENTS.md`](./IMPROVEMENTS.md)

---

## 1. 한눈에 보기

```
User
 │
 ▼
InputGate ──(빈 입력/차단)──────────────► FinalAnswer
 │ PASS
 ▼
ContextManager   (장기 메모리 조회 → 선택 → 정규화 → Agent별 포장)
 │
 ▼
Supervisor ──(일반 질문)──► EvidenceAgent
 │ (제조 질문)
 ▼
PredictionAgent → PredictionGate ─(retry/누락)─┐
 │ PASS                                         │
 ▼                                              ▼
EvidenceAgent  → EvidenceGate  ─(profile retry)─┐
 │ PASS                                          │
 ▼                                               ▼
SafetyAgent    → SafetyGate    ─(BLOCK)──► FinalAnswer
 │ PASS                                     │
 ▼                                          ▼
FinalAnswerNode → OutputGate → MemoryWriterNode → Response
```

- **Agent (독립 판단)** 3개: `PredictionAgent` · `EvidenceAgent` · `SafetyAgent`
- **Node (실행 단계)** 2개: `FinalAnswerNode` · `MemoryWriterNode`
- **Gate (검증)** 5개: `Input` · `Prediction` · `Evidence` · `Safety` · `Output`
- 최종 답변은 별도 ResponseAgent 없이 **`FinalAnswerNode`** 가 조립한다.

---

## 2. 메모리 3계층 (Context Engineering 핵심)

| 계층 | 구현 | 키 | 역할 |
|------|------|----|------|
| **단기 메모리** | LangGraph `MemorySaver` 체크포인터 | `thread_id`(세션) | 한 실행/세션의 working state 자동 보존 |
| **장기 메모리** | `SqliteSaver` 체크포인터 + SQLite `ConversationStore`/`RunStore` | `session_id` | 세션 간 대화·설비값·요약·실행 이력 영속 |
| **지식 베이스** | ChromaDB 벡터 스토어 (OpenAI 임베딩) | 문서 컬렉션 | EvidenceAgent의 Adaptive RAG 검색 |

**핵심 원칙**: 이전 대화 전체를 그대로 Agent에 주입하지 않는다.
`ConversationStore 조회 → ContextSelector(선택) → ContextNormalizer(정규화) → ContextPacker(Agent별 포장)` 단계를 거쳐 현재 질문과 관련된 정보만 전달한다. (현재값 우선·오래된 값 stale 표시·prompt injection 무력화)

---

## 3. 셀 구성 (섹션 맵)

노트북은 README의 폴더 구조를 단일 노트북 섹션으로 대응시켰다.

| 섹션 | 내용 | README 대응 |
|------|------|-------------|
| **헤더** | 아키텍처 개요, 메모리 3계층 표 | — |
| **§0 설치 & 환경** | 의존성 설치, LangGraph/SqliteSaver import | — |
| **§1 설정 & LLM 어댑터** | `OPENAI_API_KEY` 입력란, `DEFAULT_MODEL`/`EMBED_MODEL`, `call_llm()` (키 없으면 StubLLM 폴백) | — |
| **§2 contracts** | Pydantic 스키마: `PredictionResult`·`EvidenceBundle`·`SafetyDecision`·`FinalAnswer`·`ContextPacket`·`GateReport` 등 + `ManufacturingState`(TypedDict) | `contracts/` |
| **§3 memory** | `ConversationStore`·`RunStore` (SQLite 장기 메모리) | `memory/` |
| **§4 벡터 스토어** | 1번 노트북에서 만든 ChromaDB 컬렉션 연결 + `vector_search()` | (RAG 기반) |
| **§5 context** | `context_policy`·`selector`·`normalizer`·`packer` | `context/` |
| **§6 services** | `prediction_service`·`rag_service`·`safety_policy_service`·`citation_service` | `services/` |
| **§7 agents** | `prediction_agent`·`evidence_agent`·`safety_agent` | `agents/` |
| **§8 gates** | `input`·`prediction`·`evidence`·`safety`·`output` gate | `gates/` |
| **§9 nodes** | `final_answer_node`·`memory_writer_node` | `nodes/` |
| **§10 context_manager** | Context Engineering 진입 노드 | `context/context_manager.py` |
| **§11 graph** | `supervisor`·`route_policy`·`build_graph()` (노드/엣지 조립) | `graph/` |
| **§12 체크포인터** | 단기(`MemorySaver`)/장기(`SqliteSaver`) 구성 + `app` 컴파일 | — |
| **§13 시각화** | Mermaid 그래프 출력 (선택) | — |
| **§14 실행** | 멀티턴 시나리오 3턴 + 장기 메모리/체크포인트 복원 확인 | — |
| **§15 정리** | 폴더↔섹션 대응표 | — |

---

## 4. 데이터 흐름 (한 번의 요청)

1. **`run_turn(user_message, session_id, request_id)`** 가 초기 state를 만들어 `app.invoke()` 호출 (`thread_id = session_id`).
2. **InputGate**: 빈 입력/injection/센서값 포함 등 `InputFlags` 생성.
3. **ContextManager**: `ConversationStore`에서 이전 설비값·요약 조회 → 현재값 우선 병합 → Agent별 `AgentContextPacket` 생성.
4. **Supervisor**: intent 판정 후 라우팅.
5. **PredictionAgent**: `prediction_service`가 AI4I 규칙으로 HDF/PWF/OSF/TWF 부분 위험 계산 → `PredictionResult`. LLM은 요약 문장만 생성.
6. **EvidenceAgent**: retrieval profile 선택 → `vector_search`(ChromaDB) → `EvidenceBundle` + citations.
7. **SafetyAgent**: 위험 요청·정책 검사 → `SafetyDecision`(필요 시 `blocked=True`).
8. **각 Gate**: 결과를 검사해 `GateReport` 추가, conditional edge가 retry/redirect/block/final 분기.
9. **FinalAnswerNode**: 결과 조립(라우팅 판단 없음) → `FinalAnswer`.
10. **MemoryWriterNode**: 대화·현재 설비값·요약을 `ConversationStore`에, 실행 이력을 `RunStore`에 저장.

> `gate_reports`는 reducer 없이 각 gate가 직전 state를 읽어 append하고, 턴마다 입력 `[]`로 초기화한다.
> (영속 체크포인터 + `operator.add` 조합 시 턴 간 무한 누적되는 문제를 피하기 위함.)

---

## 5. 실행 방법

### 5.1 설치
```bash
uv pip install langgraph langgraph-checkpoint-sqlite langchain-core chromadb
uv pip install langchain-openai openai   # 실제 OpenAI 사용 시
uv pip install grandalf                  # 그래프 PNG 시각화(선택)
```

### 5.2 API 키
- **§1 셀**의 `OPENAI_API_KEY = ""` 에 키 입력, 또는
- 셸 환경변수로 설정:
  ```bash
  export OPENAI_API_KEY=sk-...          # PowerShell: $env:OPENAI_API_KEY="sk-..."
  ```
- 키가 없으면 **StubLLM + 키워드 검색**으로 폴백해 오프라인에서도 끝까지 실행된다.

### 5.3 실행
위에서 아래로 셀을 순서대로 실행. §14의 `run_turn(...)`로 멀티턴 대화를 확인한다.

### 5.4 모델 변경
| 변수 | 위치 | 기본값 | 대안 |
|------|------|--------|------|
| `DEFAULT_MODEL` | §1 | `gpt-4o` | `gpt-4o-mini`(저비용) |
| `EMBED_MODEL` | §1 | `text-embedding-3-small` | `text-embedding-3-large`(고품질) |

---

## 6. 산출물/경로

| 경로 | 설명 |
|------|------|
| `agent_data/longterm_memory.sqlite` | 장기 메모리(대화·설비값·요약·실행 이력) |
| `agent_data/checkpoints.sqlite` | `SqliteSaver` 체크포인트 |
| `agent_data/chroma/` | ChromaDB 벡터 스토어 |

> `agent_data/` 는 `.gitignore` 에 포함(런타임 데이터). 임베딩 모델이 바뀌면 벡터 차원이 달라지므로 OpenAI/기본 임베딩을 **다른 컬렉션**(`manufacturing_docs_openai` / `_default`)에 분리 저장한다.

---

## 7. 현재 한계 (요약)

실행 흐름·Context Engineering·메모리·Gate는 동작하지만, 핵심 판단 로직 일부는 **데모용 stub/규칙 기반**이다.
- PredictionAgent: 규칙 기반(실제 AI4I ML 모델 아님)
- RAG: ChromaDB 기반 문서 검색 (rerank/grading 미구현)
- SafetyAgent: 정규식 기반 탐지

상세 보완 과제와 우선순위는 [`IMPROVEMENTS.md`](./IMPROVEMENTS.md) 참고.

---

## 8. 관련 파일

| 파일 | 역할 |
|------|------|
| `01_embed_documents_chroma.ipynb` | 1번: `document/` 문서를 ChromaDB에 임베딩 |
| `manufacturing_agent.ipynb` | 2번: 임베딩된 ChromaDB를 읽어 Agent/RAG 실행 |
| `README.md` | 아키텍처 설계서(기획안) |
| `IMPROVEMENTS.md` | 보완 과제 |
| `pyproject.toml` | 의존성 정의 |

---

## 9. 노트북 → 실제 프로젝트 구조로 확장하기

이 노트북은 **검증용 뼈대(skeleton)** 다. 한 파일에서 전체 흐름을 빠르게 확인하는 데는 좋지만,
협업·테스트·배포가 필요한 실제 프로젝트에서는 노트북의 각 섹션을 **모듈 파일로 분해**해야 한다.
노트북 섹션은 이미 README의 폴더 구조와 1:1로 대응하도록 만들었으므로, 분해가 곧 패키지화다.

### 9.1 목표 패키지 구조

```
manufacturing_agent/
├── app.py                      # 진입점: 그래프 컴파일 + invoke 래퍼 (노트북 §12·§14)
├── config.py                   # 설정/모델명/경로/키 (노트북 §1)  ← 상수는 전부 여기로
│
├── contracts/                  # 노트북 §2
│   ├── state.py                #   ManufacturingState (TypedDict)
│   ├── routing.py              #   InputFlags, RouteDecision, GateReport
│   ├── results.py              #   PredictionResult, EvidenceBundle, SafetyDecision, FinalAnswer
│   └── context.py              #   ContextPacket, AgentContextPacket, MachineValue
│
├── memory/                     # 노트북 §3
│   ├── conversation_store.py   #   ConversationStore (SQLite 장기 메모리)
│   ├── run_store.py            #   RunStore
│   └── checkpointer.py         #   MemorySaver(단기)/SqliteSaver(장기) 팩토리 (노트북 §12)
│
├── vectorstore/                # 노트북 §4
│   ├── client.py               #   Chroma 컬렉션 + OpenAI 임베딩
│   └── seed.py                 #   초기 문서 적재 (운영에선 ETL/인덱싱 파이프라인으로 대체)
│
├── context/                    # 노트북 §5·§10
│   ├── context_policy.py       │   context_manager.py
│   ├── context_selector.py     │   context_normalizer.py
│   └── context_packer.py
│
├── services/                   # 노트북 §6
│   ├── prediction_service.py   #   ← 여기서 규칙 → 실제 ML 모델로 교체 (IMPROVEMENTS §1.1)
│   ├── rag_service.py          │   safety_policy_service.py
│   └── citation_service.py
│
├── agents/                     # 노트북 §7
│   ├── prediction_agent.py     │   evidence_agent.py
│   └── safety_agent.py
│
├── gates/                      # 노트북 §8 (input/prediction/evidence/safety/output)
├── nodes/                      # 노트북 §9 (final_answer/memory_writer)
├── graph/                      # 노트북 §11
│   ├── supervisor.py           │   route_policy.py
│   └── graph.py                #   build_graph()
│
├── prompts/                    # 시스템 프롬프트 .md (LLM 호출부에서 로드)
└── llm.py                      # call_llm() / call_llm_structured() 어댑터 (노트북 §1)

api/                            # (선택) 서비스 노출 계층
└── main.py                     # FastAPI: POST /chat → run_turn 호출, 스트리밍/세션 관리

tests/                          # 단위/통합/평가 테스트 (IMPROVEMENTS §6)
└── ...

data/                           # 런타임 (gitignore): SQLite·Chroma  ← 노트북의 agent_data/
```

### 9.2 셀 → 파일 매핑

| 노트북 셀 | 이동 위치 | 분해 시 주의 |
|-----------|-----------|--------------|
| §1 설정/키 | `config.py`, `llm.py` | 상수는 `config.py`로, 키는 코드에 두지 말고 `.env`+`python-dotenv` |
| §2 contracts | `contracts/*.py` | 가장 먼저 분리(다른 모듈이 전부 의존) |
| §3 memory | `memory/conversation_store.py`, `run_store.py` | 전역 인스턴스 대신 의존성 주입 |
| §4 벡터 | `vectorstore/client.py`, `seed.py` | 시드 적재는 1회성 스크립트로 분리 |
| §5·§10 context | `context/*.py` | `ConversationStore`를 인자로 주입 |
| §6 services | `services/*.py` | 외부 I/O(모델·DB) 경계 |
| §7 agents | `agents/*.py` | state → 결과 객체만 반환(순수하게) |
| §8 gates | `gates/*.py` | 판단 금지, 검증만 |
| §9 nodes | `nodes/*.py` | FinalAnswer는 조립만, 라우팅 금지 |
| §11 graph | `graph/graph.py`, `supervisor.py`, `route_policy.py` | 노드/엣지 등록만 |
| §12·§14 | `app.py` (+ `api/main.py`) | 체크포인터 선택, `run_turn` |

### 9.3 계층(의존성 방향)

```
        ┌─────────────────────────────────────────────┐
 진입   │  app.py / api/main.py                        │
        └───────────────┬─────────────────────────────┘
                        ▼
 오케스트레이션 │  graph/ (supervisor, route_policy, graph)   │
        ┌───────────────┼─────────────────────────────┐
        ▼               ▼               ▼
 도메인  │  gates/   │  agents/   │  nodes/   │  context/ │
        └───────────────┬─────────────────────────────┘
                        ▼
 기능   │  services/ (prediction · rag · safety · citation)   │
        └───────────────┬─────────────────────────────┘
                        ▼
 인프라  │  memory/ (SQLite)  │  vectorstore/ (Chroma)  │  llm.py  │
        └───────────────┬─────────────────────────────┘
                        ▼
 계약   │  contracts/ (모든 계층이 의존하는 순수 스키마)        │
```

**규칙**: 의존성은 위 → 아래 단방향. `contracts/` 는 아무것도 의존하지 않는다.
`agents/`·`gates/`·`nodes/` 는 `services/`·`memory/`·`llm.py` 를 호출하되 서로는 모른다.
이것이 README의 "Agent는 판단만, Gate는 검증만, Node는 조립만, Service는 실행만" 원칙을 코드 구조로 강제하는 형태다.

### 9.4 마이그레이션 순서 (권장)

README §15 구현 순서와 동일하게, **의존성이 적은 것부터** 옮긴다.

1. `contracts/` 분리 → `pip install -e .` 로 패키지화
2. `config.py` + `llm.py` (전역 상수/키 정리, `.env` 도입)
3. `memory/` + `vectorstore/` (인프라, 의존성 주입으로 전환)
4. `services/` → `agents/` → `gates/` → `nodes/` → `graph/`
5. `app.py` 로 조립, **노트북과 동일한 §14 시나리오를 회귀 테스트**로 고정
6. `api/main.py` (FastAPI) 로 서비스 노출, `tests/` 작성

> 각 단계마다 노트북의 멀티턴 결과(`run_turn` 출력)와 모듈 버전 결과가 **동일한지** 비교하면
> 분해 과정에서 동작이 깨지지 않았음을 보장할 수 있다. 노트북이 곧 살아있는 명세 역할을 한다.

### 9.5 운영(런타임) 아키텍처 예시

```
[Client / 현장 UI]
       │ HTTP
       ▼
[FastAPI  api/main.py]  ── 세션/인증/요청검증
       │ run_turn(session_id, msg)
       ▼
[LangGraph app]  ──(체크포인터)──► [Postgres/SQLite]   # 단기·장기 state
       │
       ├─► [services/prediction] ──► [ML 모델 서버 / 레지스트리]   # 실제 추론 (IMPROVEMENTS §1.1)
       ├─► [services/rag]        ──► [Chroma/PGVector] ◄─ [임베딩 API]
       ├─► [services/safety]     ──► [정책 룰셋 DB]
       └─► [llm.py]              ──► [OpenAI API]
       │
       ▼
[관측: LangSmith/OTel] ◄─ RunStore 트레이스   # IMPROVEMENTS §5
```

- 단일 프로세스(노트북) → **API 서버 + 외부 모델/벡터/DB** 로 분리.
- 동시성·다중 사용자를 위해 SQLite → **Postgres 체크포인터/PGVector** 전환 권장(IMPROVEMENTS §3·§4).
- 고위험(critical/BLOCK) 요청은 LangGraph `interrupt` 로 **사람 승인(HITL)** 단계 삽입.

상세 보완 항목과 우선순위는 [`IMPROVEMENTS.md`](./IMPROVEMENTS.md) 참고.
