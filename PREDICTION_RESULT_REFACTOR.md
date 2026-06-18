# PredictionResult 구조화 및 ML 필드 제거 작업사항

작성일: 2026-06-18

## 배경

기존 `PredictionResult`는 `partial_risks: list[dict]`와 `ml_prediction` 중심으로 구성되어 있었다.

이 구조에는 다음 한계가 있었다.

- `EvidenceAgent`가 `partial_risks[*]["failure_type"]`만 보고 단순 검색 쿼리를 생성했다.
- `SafetyAgent`가 `partial_risks[*]["level"] == "high"`만 확인했다.
- risk dict의 필드 계약이 명확하지 않아 downstream agent가 키 이름에 취약했다.
- `ml_prediction`이라는 이름이 있었지만 실제 구현은 학습 모델이 아니라 규칙 기반 top risk 요약이었다.
- 멀티턴에서 이전 턴 값을 사용했는지 결과 객체에 남지 않았다.

이번 작업에서는 ML 모델 사용을 제거한다는 결정에 맞춰 `ml_prediction`을 완전히 삭제하고, `PredictionAgent`가 이후 에이전트에 넘겨야 하는 정보를 명시적인 구조로 보강했다.

## 변경 파일

- `manufacturing_agent.ipynb`
- `PREDICTION_RESULT_REFACTOR.md`

## 주요 변경

### 1. `PredictionResult` 계약 변경

기존 형태:

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

변경 후:

```python
class PredictionResult(BaseModel):
    status: str
    available_features: list[str] = []
    missing_features: list[str] = []
    full_prediction_available: bool = False
    partial_risks: list[FailureRisk] = []
    evidence_hints: list[EvidenceHint] = []
    safety_hints: list[SafetyHint] = []
    used_stale_features: list[str] = []
    confidence: str = "low"
    limitations: list[str] = []
    summary: str = ""
```

`ml_prediction`은 제거했다.

### 2. 구조화 모델 추가

`FailureRisk`

- 고장 유형별 위험 계산 결과를 표현한다.
- `failure_type`, `level`, `score`, `detail` 외에 downstream agent가 쓸 정보를 같이 갖는다.
- `contributing_features`: 해당 risk 계산에 기여한 feature 목록.
- `evidence_query_terms`: RAG 검색용 핵심 키워드.
- `recommended_checks`: 작업자/안전 판단에 필요한 점검 항목.

`EvidenceHint`

- `EvidenceAgent`가 직접 사용할 검색 힌트다.
- `failure_type`, `priority`, `queries`, `features`를 갖는다.
- 기존처럼 EvidenceAgent가 risk dict를 해석해서 쿼리를 만드는 책임을 줄였다.

`SafetyHint`

- `SafetyAgent`가 직접 사용할 안전 힌트다.
- high risk에 대해 `reason`, `avoid_actions`, `required_checks`를 제공한다.
- SafetyAgent가 prediction detail 문자열을 직접 해석하지 않아도 된다.

### 3. prediction service 변경

`ML_FEATURES`를 `PREDICTION_FEATURES`로 변경했다.

이름상 ML 모델이 남아 있는 것처럼 보이는 부분을 제거했고, 현재 구현은 명확히 규칙 기반 예측 서비스로 정리했다.

`compute_partial_risks()`는 이제 `list[dict]`가 아니라 `list[FailureRisk]`를 반환한다.

또한 risk 목록을 score 내림차순으로 정렬한다. 기존에는 HDF, PWF, OSF, TWF 순서대로 append되었기 때문에 `partial_risks[:2]`가 실제 상위 위험 2개라는 보장이 없었다.

### 4. EvidenceAgent 활용 방식 개선

기존 `adaptive_retrieve()`:

```python
if prediction and prediction.partial_risks:
    for r in prediction.partial_risks[:2]:
        queries.append(f"{r['failure_type']} 원인과 점검 방법")
```

변경 후:

```python
if prediction and prediction.evidence_hints:
    for hint in prediction.evidence_hints[:2]:
        queries.extend(hint.queries)
elif prediction and prediction.partial_risks:
    for r in prediction.partial_risks[:2]:
        queries.append(f"{r.failure_type} 원인과 점검 방법")
```

이제 EvidenceAgent는 `PredictionResult.evidence_hints`를 우선 사용한다.

예상 쿼리 예:

- 원래 사용자 질문
- `HDF 원인과 점검 방법`
- `HDF heat dissipation failure 온도차 저속 회전`
- `TWF 원인과 점검 방법`
- `TWF tool wear failure 공구마모 공구 수명`

누락 feature가 있을 경우 `"{failure_type} 진단에 필요한 누락 입력 ..."` 쿼리도 추가될 수 있다.

### 5. SafetyAgent 활용 방식 개선

기존 `evaluate_safety()`는 `partial_risks`에서 `level == "high"`만 확인했다.

변경 후에는 `PredictionResult.safety_hints` 존재 여부를 high risk 판단의 주 입력으로 사용한다.

또한 안전 노트에 다음 정보가 추가된다.

- high risk 사유
- 필요한 점검 항목

예:

```text
HDF high: 온도차=5.0K, rpm=1300; 필요 점검: 공정온도와 공기온도 센서 확인, 냉각/환기 상태 점검, 저속 운전 조건 확인
```

### 6. 멀티턴 stale feature 기록

`PredictionAgent`는 `AgentContextPacket.selected_context["stale"]` 값을 읽어 `PredictionResult.used_stale_features`에 저장한다.

이 값은 `FinalAnswerNode`에서 최종 답변에 반영된다.

예:

```text
[맥락] 이전 턴 값 사용: ['air_temperature', 'process_temperature']
```

이제 “토크만 60으로 바꿔서 다시” 같은 멀티턴 요청에서 어떤 값이 현재 턴 값이고 어떤 값이 이전 턴 보완값인지 결과에 남는다.

### 7. confidence와 limitations 추가

`run_prediction()`은 다음 기준으로 confidence를 부여한다.

- `high`: 전체 feature가 모두 있음.
- `medium`: 전체 feature는 부족하지만 부분 risk 계산 가능.
- `low`: risk 계산도 어려움.

누락 feature가 있으면 `limitations`에 설명을 남긴다.

예:

```python
limitations = ["전체 예측에 필요한 입력 누락: ['type', 'tool_wear']"]
```

`FinalAnswerNode`는 이 내용을 `[제약]` 섹션으로 출력한다.

## 이후 흐름

변경 후 `PredictionResult` 활용 흐름은 다음과 같다.

```text
PredictionAgent
  -> PredictionResult
      -> PredictionGate
          - full_prediction_available
          - partial_risks
          - missing_features
      -> EvidenceAgent
          - evidence_hints 우선 사용
          - fallback으로 partial_risks 사용
      -> SafetyAgent
          - safety_hints 사용
      -> FinalAnswerNode
          - summary
          - partial_risks
          - missing_features
          - used_stale_features
          - limitations
      -> MemoryWriterNode
          - summary 저장
```

## 검증 내용

다음 검증을 수행했다.

1. 노트북 JSON 로딩 확인

```text
json_ok 52
```

2. 모든 code cell Python 문법 검사

```text
code_cells_syntax_ok
```

3. 샘플 state 기반 경로 실행

검증 경로:

```text
prediction_agent
-> PredictionResult 생성
-> adaptive_retrieve에서 evidence_hints 소비
-> evaluate_safety에서 safety_hints 소비
```

결과:

```text
prediction_ok OK high HDF
queries_ok 5
safety_ok high True
```

추가로 `PredictionResult`에 `ml_prediction` 속성이 없는 것도 확인했다.

## 남은 고려사항

현재 `summary`는 여전히 LLM 또는 StubLLM으로 생성한다. 다만 핵심 판단과 downstream routing은 구조화 필드 기반으로 동작하므로 summary 텍스트 품질에 의존하지 않는다.

향후 실제 운영 수준으로 올리려면 다음 보완이 가능하다.

- `failure_type`, `level`, `confidence`를 `Literal` 또는 Enum으로 제한.
- `EvidenceHint.queries`의 중복 제거 및 쿼리 수 제한.
- stale feature가 일정 시간 이상 오래된 경우 confidence를 자동 하향.
- `SafetyHint.required_checks`를 안전 표준 문서 citation과 연결.
