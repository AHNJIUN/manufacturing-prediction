# Supervisor 중심 오케스트레이션 변경사항

작성일: 2026-06-18

## 배경

기존 그래프는 `Supervisor`가 초기에 한 번만 라우팅하고, 이후에는 각 `Gate` 뒤의 conditional edge가 다음 노드를 직접 결정하는 구조였다.

기존 흐름:

```text
input_gate
-> context_manager
-> supervisor
-> prediction_agent
-> prediction_gate
-> evidence_agent
-> evidence_gate
-> safety_agent
-> safety_gate
-> final_answer
```

이 구조에서는 `prediction_agent` 이후에 `supervisor`로 돌아가지 않는다. 따라서 전체 오케스트레이션이 supervisor 중심이라기보다 gate 중심 파이프라인에 가까웠다.

이번 변경은 `PredictionAgent`와 `EvidenceAgent`의 실행/검증 결과가 다시 `Supervisor`로 모이고, `Supervisor`가 다음 예측/근거 수집 단계를 중앙 결정하도록 바꾸는 것이다.

`SafetyAgent`/`SafetyGate`는 supervisor loop의 반복 대상이 아니라 최종 답변 직전 guard로 둔다.

## 변경 후 구조

변경 후 흐름:

```text
input_gate
-> context_manager
-> supervisor
   -> prediction_agent
   -> prediction_gate
-> supervisor
   -> evidence_agent
   -> evidence_gate
-> supervisor
   -> safety_agent
   -> safety_gate
   -> final_answer
   -> output_gate
   -> memory_writer
   -> END
```

핵심은 다음과 같다.

- `PredictionAgent`와 `EvidenceAgent`는 자기 판단만 수행한다.
- `PredictionGate`와 `EvidenceGate`는 검증 후 `supervisor`로 돌아간다.
- 예측/근거 단계의 다음 노드 선택은 `Supervisor`가 담당한다.
- `SafetyAgent`와 `SafetyGate`는 최종 답변 전 한 번 통과하는 guard다.
- `safety_gate`는 `supervisor`로 돌아가지 않고 `final_answer`로 연결된다.

## Supervisor 책임

`supervisor(state)`는 다음 정보를 보고 `RouteDecision`을 만든다.

- `input_flags`
- 현재 `intent`
- 마지막 `GateReport`
- `retry_counts`

결정 결과는 `state["route"]`에 저장된다.

```python
route = RouteDecision(next_node=next_node, reason=reason)
return {"intent": intent, "route": route}
```

`route_after_supervisor()`는 더 이상 자체 판단을 하지 않고 `state["route"].next_node`를 그대로 따른다.

```python
def route_after_supervisor(state) -> str:
    route = state.get("route")
    return route.next_node if route else "safety_agent"
```

## Gate별 Supervisor 판단

### PredictionGate 이후

`prediction_gate`가 끝나면 supervisor가 다음을 결정한다.

- `PASS`, `PASS_WITH_PARTIAL_RESULT`: `evidence_agent`
- `ASK_MISSING_INPUT`: `safety_agent`
- 그 외 실패: prediction retry 가능하면 `prediction_agent`, 아니면 `safety_agent`

### EvidenceGate 이후

`evidence_gate`가 끝나면 supervisor가 다음을 결정한다.

- `PASS`: `safety_agent`
- `RETRY...`: retry 가능하면 `evidence_agent`, 아니면 `safety_agent`
- `INSUFFICIENT_EVIDENCE`: `safety_agent`

근거가 부족해도 안전 판단은 진행한다. 최종 답변에서 근거 부족 상태를 함께 반영할 수 있기 때문이다.

### SafetyGate 이후

`safety_gate`가 끝나면 supervisor로 돌아가지 않고 바로 `final_answer`로 간다.

`SafetyDecision.blocked` 여부와 무관하게 최종 답변 조립은 `FinalAnswerNode`가 담당한다. 이때 차단 메시지나 안전 권고가 최종 답변에 포함된다.

## Graph 변경

기존에는 gate마다 conditional edge가 있었다.

```python
g.add_conditional_edges("prediction_gate", route_after_prediction_gate, ...)
g.add_conditional_edges("evidence_gate", route_after_evidence_gate, ...)
g.add_conditional_edges("safety_gate", route_after_safety_gate, ...)
```

변경 후에는 gate가 supervisor로 단순 edge를 가진다.

```python
g.add_edge("prediction_gate", "supervisor")
g.add_edge("evidence_gate", "supervisor")
g.add_edge("safety_gate", "final_answer")
```

그리고 supervisor의 conditional edge는 예측/근거 흐름과 최종 safety guard 진입점만 포함한다.

```python
g.add_conditional_edges(
    "supervisor",
    route_after_supervisor,
    {
        "prediction_agent": "prediction_agent",
        "evidence_agent": "evidence_agent",
        "safety_agent": "safety_agent",
    },
)
```

## 제거한 역할

다음 함수들은 더 이상 사용하지 않으므로 제거했다.

- `route_after_prediction_gate`
- `route_after_evidence_gate`
- `route_after_safety_gate`

이제 `prediction_gate`와 `evidence_gate` 이후 다음 노드 결정 로직은 `supervisor()` 안에만 있다. `safety_gate` 이후에는 분기하지 않고 `final_answer`로 간다.

## 기대 효과

- 라우팅 책임이 `Supervisor` 하나로 모인다.
- Gate는 검증 책임만 갖는다.
- Prediction/Evidence agent는 독립 판단 책임만 갖는다.
- retry, redirect, final 전환 정책을 한 곳에서 볼 수 있다.
- 향후 planner, cost-aware routing, human approval, interrupt 같은 정책을 supervisor에 추가하기 쉬워진다.

## 주의점

현재 `supervisor`는 `safety_agent`로 넘긴 뒤 다시 호출되지 않는다. 이후는 `safety_gate -> final_answer -> output_gate -> memory_writer -> END`로 종료한다.

또한 `input_gate` 실패는 아직 supervisor를 거치지 않고 바로 `final_answer`로 간다. 이는 빈 입력처럼 context 구성 자체가 의미 없는 경우를 빠르게 종료하기 위한 기존 정책을 유지한 것이다.
