# PredictionAgent 구현 노트

작성일: 2026-06-19

---

## 배경

`manufacturing_agent.ipynb`는 팀원이 구성한 LangGraph 스켈레톤 노트북이다.
기존 스켈레톤은 `PredictionAgent`가 단일 함수(`prediction_agent()`)로 구현되어 있었고,
모드 분기·재질문 처리·민감도 탐색이 없는 단순 실행 구조였다.

이 노트북에 다음 작업을 추가했다:
- **PredictionAgent 심화 구현**: 5가지 실행 모드 + 2-레벨 라우팅 + ToolNode 루프
- **PredictionResult 계약 구조화**: `ml_prediction` 제거 및 downstream 힌트 필드 추가
- **1차 QA 수정**: 버그 2건 수정, 개선 항목 발굴 및 기록

세부 기획 근거: `기획안_v1.4_draft.md` / 설계 원본: `../99.personal_space/Prediction_Agent/prediction_Agent_code.md`

---

## 1. 내가 추가한 PredictionAgent 구조

### 전체 흐름 (2-레벨 라우팅)

```
[사용자 입력]
     │
     ▼
[Supervisor]                           ← 1차 라우팅: 예측 / 문서검색 / 안전
     │
     └──────────────────────────────────▶ [prediction_router]
                                                   │       ← 2차 라우팅: 예측 내부 모드 결정
                              ┌────────────────────┼─────────────────────────┐
                              │                    │                         │
                         RUN_RULE             WHAT_IF                  EXPLORATORY
                              │                    │                         │
                              ▼                    ▼                         ▼
                  [prediction_run_       [prediction_what_       [prediction_agent_
                   rule_node]             if_node]                explorer]
                   Rule 계산 +            이전값 병합 +                  │
                   LLM 요약               Rule 재실행 +           tool_calls 있음?
                              │           LLM 요약          ┌──────────┘
                              │                    │        ▼           ▼
                              │                    │  [prediction_   [prediction_
                              │                    │   tool_node]   result_builder]
                              │                    │        │
                              │                    │        └── loop back ──┘
                              │                    │                    │
              NEED_MORE / CANNOT                   │                    │
                              ▼                    │                    │
              [prediction_clarification_           │                    │
               node]                              │                    │
               누락 피처 목록 반환                 │                    │
                              │                   │                    │
                              └───────────────────┴────────────────────┘
                                                  │
                                                  ▼
                                         [prediction_gate]
                                                  │
                                   ┌──────────────┼──────────────┐
                              OK/PARTIAL     INSUFFICIENT       FAIL
                                   ▼               ▼              ▼
                             evidence_agent  final_answer   retry(최대2회)
```

**설계 선택 포인트**: ReAct 대신 ToolNode + conditional edge 채택.
그래프 구조를 시각화할 수 있고, get_sensitivity 반복 호출을 conditional edge로 제어할 수 있으며, Tool 결과가 state에 누적되어 추적이 용이하다.
ReAct는 루프가 블랙박스가 되어 디버깅과 설명이 불리하다.

---

## 2. 실행 모드 5종

| 모드 | 진입 조건 | 담당 노드 | LLM 사용 |
|---|---|---|---|
| `RUN_RULE` | Rule 계산 가능한 피처 하나라도 있음 | `prediction_run_rule_node` | 요약만 |
| `WHAT_IF` | 이전 예측 있음 + 변경 키워드 | `prediction_what_if_node` | 요약만 |
| `EXPLORATORY` | 탐색형 키워드 감지 | `prediction_agent_explorer` | 판단 주체 |
| `NEED_MORE_FEATURES` | Rule 불가, 피처 일부 있음 | `prediction_clarification_node` | X |
| `CANNOT_PREDICT` | 피처 0개 | `prediction_clarification_node` | X |

**우선순위 (D-1 수정 반영):** `WHAT_IF → EXPLORATORY → RUN_RULE → CANNOT → NEED_MORE`
(원래 EXPLORATORY가 WHAT_IF보다 우선이었으나 키워드 충돌 버그로 역전 수정)

---

## 3. 주요 노드별 역할

| 노드 | 역할 |
|---|---|
| `prediction_router` | 피처 완비 여부 + 키워드 감지로 모드 결정. LLM 호출 없음 |
| `prediction_run_rule_node` | HDF/PWF/OSF/TWF Rule 계산 + LLM 요약 |
| `prediction_what_if_node` | 이전값 병합(prev + curr) + Rule 재실행 + delta 계산 + LLM 요약 |
| `prediction_clarification_node` | 누락 피처 목록 조립. 계산 없음 |
| `prediction_agent_explorer` | 탐색형 질문 전용. Tool 선택·반복이 LLM 판단으로 이루어짐 |
| `prediction_tool_node` | explorer가 요청한 Tool 실행 (ToolNode) |
| `prediction_result_builder` | Explorer 루프 종료 후 ToolMessage 결과 수집 → PredictionResult 조립 |
| `prediction_gate` | PredictionResult.status 기준으로 다음 경로 결정. 모드를 모름 |

**이 노드들이 하지 않는 것**: 피처 파싱(ContextManager), 도메인 간 라우팅(Supervisor), 최종 답변 생성(FinalAnswerNode), 안전 판단(SafetyNode)

---

## 4. Explorer 경로 Tool 3종

| Tool | 역할 | 호출 시점 |
|---|---|---|
| `run_prediction` | 피처값 → 고장 위험 Rule 계산 | baseline 확보, 변경값 재계산 |
| `get_sensitivity` | 특정 피처 delta 변경 시 위험 점수 변화 | "뭘 바꾸면 좋아지나" 탐색 시, 반복 가능 |
| `request_clarification` | 수치 불명확 피처 목록 + 안내 가이드 반환 | 정량값 없을 때. 호출 후 루프 종료 |

무한 루프 방어: `explorer_loop_count >= 10` 조건으로 애플리케이션 레벨 강제 종료.

---

## 5. PredictionResult 계약 변경 (구조화 작업)

### 변경 전 (팀원 스켈레톤)

```python
class PredictionResult(BaseModel):
    status: str
    available_features: list[str] = []
    missing_features: list[str] = []
    full_prediction_available: bool = False
    partial_risks: list[dict] = []
    ml_prediction: Optional[dict] = None
    summary: str = ""
```

### 변경 후 (내가 수정)

```python
class PredictionResult(BaseModel):
    status: str
    available_features: list[str] = []
    missing_features: list[str] = []
    full_prediction_available: bool = False
    partial_risks: list[FailureRisk] = []    # dict → 구조화 모델
    evidence_hints: list[EvidenceHint] = []  # EvidenceAgent용 검색 힌트
    safety_hints: list[SafetyHint] = []      # SafetyAgent용 안전 힌트
    used_stale_features: list[str] = []      # 멀티턴에서 이전 턴 값 사용 기록
    confidence: str = "low"
    limitations: list[str] = []
    summary: str = ""
```

**`ml_prediction` 제거 이유**: 실제 ML 학습 모델이 아니라 규칙 기반 top risk 요약이었음. 이름이 오해를 유발하므로 완전히 삭제.

**구조화 모델 추가 이유**:
- `EvidenceAgent`가 `partial_risks[*]["failure_type"]`만 보고 검색 쿼리를 만들던 구조에서 → `evidence_hints`를 직접 소비하는 방식으로 개선
- `SafetyAgent`가 `level == "high"` 문자열 비교에서 → `safety_hints`를 직접 소비하는 방식으로 개선
- downstream agent가 키 이름에 취약한 dict 구조 제거

**FailureRisk 추가 필드**:
- `contributing_features`: 해당 risk 계산에 기여한 feature 목록
- `evidence_query_terms`: RAG 검색용 핵심 키워드
- `recommended_checks`: 작업자/안전 판단에 필요한 점검 항목

---

## 6. Rule 계산 논리 (4종 고장 유형)

AI4I 2020 논문에 정의된 물리 규칙 기반. LLM 판단 없이 수식으로만 계산.

| 고장 유형 | 계산 조건 | 필요 피처 |
|---|---|---|
| HDF (Heat Dissipation Failure) | 온도차 < 8.6K AND rpm < 1380 → 각 조건 0.5점 | air_temperature, process_temperature, rotational_speed |
| PWF (Power Failure) | torque × rpm × 2π/60 이 3500W 미만 또는 9000W 초과 | rotational_speed, torque |
| OSF (Overstrain Failure) | tool_wear × torque ÷ 설비등급별 임계값(L:11000, M:12000, H:13000) | tool_wear, torque, type |
| TWF (Tool Wear Failure) | tool_wear ≥ 200min → 0.8, ≥ 180min → 0.4, 나머지 → 0.1 | tool_wear |

**설계 원칙**: 누락값을 평균으로 채우지 않는다. 계산 가능한 피처만으로 해당 고장 유형을 계산한다.

---

## 7. Gate 분기 규칙

| PredictionResult.status | Gate 결정 | 다음 노드 |
|---|---|---|
| `OK` | `PASS` | evidence_agent |
| `PARTIAL` | `PASS` | evidence_agent |
| `INSUFFICIENT` | `ASK_MISSING_INPUT` | final_answer |
| result 없음 / 예외 | `FAIL` → retry < 2 | prediction_router 재실행 |
| result 없음 / 예외 | `FAIL` → retry ≥ 2 | final_answer |

Gate는 실행 모드를 모른다. status만 읽고 분기한다.

---

## 8. 1차 QA 결과

> 검토 기준: 구현 코드 + `manufacturing_agent.ipynb` cell-24/29/31

### 수정 완료 (2건)

**B-1 — WHAT_IF missing_features 계산 오류**
- 위치: `prediction_what_if_node`
- 문제: WHAT_IF는 `merged_feats`로 계산하는데 `missing_features`는 `feats`(현재 입력만) 기준으로 계산 → 이전 턴에 6개 입력 후 이번 턴에 토크 하나만 바꾸면 나머지 5개가 missing으로 잘못 보고됨
- 수정: `missing = [f for f in STANDARD_FEATURES if f not in raw["merged_feats"]]`

**D-1 — EXPLORATORY ↔ WHAT_IF 키워드 충돌**
- 위치: `_determine_mode`
- 문제: WHAT_IF 요청 문장에 탐색형 단어가 포함되면 EXPLORATORY로 오분류 (예: "토크 60으로 낮추면 어떤 고장이 줄어?" → previous_features 있음에도 EXPLORATORY로 분류)
- 수정: 우선순위를 `EXPLORATORY → WHAT_IF` 에서 `WHAT_IF → EXPLORATORY` 로 역전. WHAT_IF는 "이전값 존재 + 구체적 변경"을 동시에 만족해야 하므로 더 구체적인 의도로 판단.

### 미수정 (4건)

| ID | 항목 | 분류 |
|---|---|---|
| B-2 | Explorer에서 `get_sensitivity`만 호출 시 결과 버려지고 CANNOT_PREDICT 폴백 | 버그 |
| D-2 | EXPLORATORY 이후 `previous_features` 미저장 → 다음 턴 WHAT_IF 분류 불가 | 설계 개선 |
| D-3 | CANNOT_PREDICT와 NEED_MORE_FEATURES 안내문 미구분 (final_answer_node에서 동일 처리) | 설계 개선 |
| E-1 | `prediction_result_builder`에서 ToolMessage.content 파싱 방어 미비 | 확장 고려 |

**B-2 수정 방향**: `sensitivity_results`만 있는 경우도 PARTIAL 상태로 처리하는 분기 추가 필요.

**D-2 수정 방향**: `prediction_result_builder`가 Explorer의 `run_prediction` tool 호출 결과에서 features를 추출해 `previous_features`도 함께 반환. 단, `run_prediction` tool 반환값에 `features_used: dict` 포함 필요.

**D-3 수정 방향**: `final_answer_node`에서 `execution_mode`를 읽어 안내문 분기.

---

## 9. 파일 관계 정리

| 파일 | 역할 |
|---|---|
| `manufacturing_agent.ipynb` | 실행 가능한 전체 스켈레톤. 팀원 구조에 PredictionAgent 심화 코드 추가됨 |
| `PREDICTION_RESULT_REFACTOR.md` | PredictionResult 계약 변경 작업 상세 기록 (2026-06-18) |
| `NOTEBOOK_GUIDE.md` | 노트북 섹션 구성 및 실행 방법 가이드 |
| `01_embed_documents_chroma.ipynb` | RAG용 문서 ChromaDB 임베딩 (별도 선실행 필요) |
| `../99.personal_space/Prediction_Agent/prediction_Agent_code.md` | PredictionAgent 전체 설계 원본 (노드 코드 + 시나리오 포함) |
| `../99.personal_space/Prediction_Agent/1st_qa.md` | 1차 QA 목록 원본 (버그/설계개선/확장 분류) |
