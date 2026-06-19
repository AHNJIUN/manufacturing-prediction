# PredictionAgent 구현 노트

작성일: 2026-06-19

---

## 배경

`manufacturing_agent.ipynb`는 팀원이 구성한 LangGraph 스켈레톤 노트북이다.
기존 스켈레톤은 `PredictionAgent`가 단일 함수(`prediction_agent()`)로 구현되어 있었고,
모드 분기·재질문 처리·민감도 탐색이 없는 단순 실행 구조였다.

이 노트북에 다음 작업을 추가했다:
- **PredictionAgent 심화 구현**: 5가지 실행 모드 + 2-레벨 라우팅 + 전체 SmartNode 파이프라인
- **PredictionResult 계약 구조화**: `ml_prediction` 제거 및 downstream 힌트 필드 추가
- **1차 QA 수정**: 버그 2건 수정, EXPLORATORY LLM ToolNode 루프 → SmartNode 전환 (A-1)

> **A-1 아키텍처 변경**: EXPLORATORY 경로의 LLM ToolNode 루프(`prediction_agent_explorer` + `prediction_tool_node` + `prediction_result_builder`)를 제거하고 `prediction_exploratory_node`(SmartNode)로 교체. 세 Tool 모두 결정론적 Rule 계산이므로 LLM 판단이 불필요하다고 결론.

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
                    ┌──────────────────────────────┼──────────────────────────┐
                    │                    │                    │                │
               WHAT_IF             RUN_RULE           EXPLORATORY        NEED_MORE/CANNOT
                    │                    │                    │                │
                    ▼                    ▼                    ▼                ▼
        [prediction_what_    [prediction_run_    [prediction_       [prediction_
         if_node]             rule_node]          exploratory_       clarification_
         이전값 병합 +         Rule 계산 +         node]              node]
         Rule 재실행 +         LLM 요약            전체 피처 순회 +   누락 피처 목록
         LLM 요약                                  민감도 정렬 +      반환
                    │                    │         LLM 요약           │
                    └────────────────────┴──────────────┴────────────┘
                                                  │
                                                  ▼
                                         [prediction_gate]
                                                  │
                                   ┌──────────────┼──────────────┐
                              OK/PARTIAL     INSUFFICIENT       FAIL
                                   ▼               ▼              ▼
                             evidence_agent  final_answer   retry(최대2회)
```

**설계 선택 포인트**: 모든 실행 노드를 SmartNode로 구현. EXPLORATORY 경로도 LLM ToolNode 루프 대신 SmartNode(`prediction_exploratory_node`)로 전환.
세 Tool(`run_prediction`, `get_sensitivity`, `request_clarification`) 모두 결정론적 Rule 계산이므로 LLM 판단이 추가 가치를 제공하지 않음.
전체 피처 고정 순회로 동일한 결과를 더 단순하게 달성 가능.

---

## 2. 실행 모드 5종

| 모드 | 진입 조건 | 담당 노드 | LLM 사용 |
|---|---|---|---|
| `WHAT_IF` | 이전 예측 있음 + 변경 키워드 | `prediction_what_if_node` | 요약만 |
| `EXPLORATORY` | 탐색형 키워드 감지 | `prediction_exploratory_node` | 요약만 |
| `RUN_RULE` | Rule 계산 가능한 피처 하나라도 있음 | `prediction_run_rule_node` | 요약만 |
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
| `prediction_exploratory_node` | 전체 수치 피처 순회 + delta별 위험점수 변화 계산 + 영향도 정렬 + LLM 요약 |
| `prediction_clarification_node` | 누락 피처 목록 조립. 계산 없음 |
| `prediction_gate` | PredictionResult.status 기준으로 다음 경로 결정. 모드를 모름 |

**이 노드들이 하지 않는 것**: 피처 파싱(ContextManager), 도메인 간 라우팅(Supervisor), 최종 답변 생성(FinalAnswerNode), 안전 판단(SafetyNode)

---

## 4. `prediction_exploratory_node` SmartNode 처리 흐름

> A-1 결정으로 Tool 3종 + LLM ToolNode 루프 전체를 SmartNode로 교체.

```
1. compute_rule_risks(feats) → baseline 위험 계산
2. type 제외 수치 피처 순회 → delta [-10, -5, +5, +10] 각각 적용
3. delta_score = after_score - baseline_score, 절댓값 내림차순 정렬
4. build_interpretable_reason + 민감도 분석 결과 합산
5. call_llm → 1~2문장 요약
6. state["previous_features"] = feats 저장 (다음 턴 WHAT_IF용)
```

| 질문 유형 | 처리 방식 |
|---|---|
| "어떤 값 바꾸면 위험이 줄어?" | 전체 피처 순회 → delta_score 정렬 → 상위 피처 안내 |
| "민감도 분석해줘" | 동일 — 전체 피처 순회 결과 반환 |
| "토크 vs rpm 중 뭐가 더 영향 커?" | 두 피처의 delta_score 비교 포함 |
| 피처 없음 | status="INSUFFICIENT" (clarification_node와 동일 처리) |

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

### A-1 구조 변경으로 해소된 항목 (3건)

| ID | 항목 | 상태 |
|---|---|---|
| B-2 | Explorer에서 `get_sensitivity`만 호출 시 결과 버려지고 CANNOT_PREDICT 폴백 | ✅ N/A — SmartNode 전환으로 ToolNode 루프 자체 제거 |
| D-2 | EXPLORATORY 이후 `previous_features` 미저장 → 다음 턴 WHAT_IF 분류 불가 | ✅ 해소 — `prediction_exploratory_node`가 직접 저장 |
| E-1 | `prediction_result_builder`에서 ToolMessage.content 파싱 방어 미비 | ✅ N/A — `prediction_result_builder` 노드 제거 |

### 미수정 (1건)

| ID | 항목 | 분류 |
|---|---|---|
| D-3 | CANNOT_PREDICT와 NEED_MORE_FEATURES 안내문 미구분 (final_answer_node에서 동일 처리) | 설계 개선 |

**D-3 수정 방향**: `final_answer_node`에서 `execution_mode`를 읽어 안내문 분기.
- `CANNOT_PREDICT` → "설비 정보가 전혀 없습니다. 아래 값들을 알려주세요."
- `NEED_MORE_FEATURES` → "일부 값은 있지만 고장 유형 계산을 위해 추가로 필요합니다: ..."

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
