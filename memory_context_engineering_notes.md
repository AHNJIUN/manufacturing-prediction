# LangGraph 메모리와 컨텍스트 엔지니어링 정리

이 문서는 `manufacturing_agent.ipynb`를 기준으로 현재 코드의 메모리 구조, 컨텍스트 엔지니어링 방식, `ConversationStore`와 LangGraph `Store`의 차이, 그리고 보완 방향을 정리한 것이다.

## 1. 현재 코드의 큰 구조

현재 노트북은 메모리를 크게 세 종류로 나누어 사용한다.

```text
1. LangGraph checkpointer
   - 현재 코드: SqliteSaver 또는 MemorySaver
   - 역할: 그래프 실행 상태 저장/복원
   - 기준 ID: thread_id

2. 직접 만든 장기 메모리
   - 현재 코드: ConversationStore
   - 역할: 대화 이력, 설비값, 이전 판단 요약 저장
   - 기준 ID: 현재는 session_id

3. 지식 검색 저장소
   - 현재 코드: ChromaDB 또는 fallback keyword search
   - 역할: 제조 매뉴얼/안전 문서 검색
   - 기준: query, metadata type
```

즉 현재 코드는 단순히 "단기 메모리"만 쓰는 구조가 아니라, 다음과 같은 하이브리드 구조다.

```text
SqliteSaver          -> LangGraph checkpoint
ConversationStore   -> 제조 도메인 장기 메모리
ChromaDB            -> RAG 지식 베이스
```

단, LangGraph 공식 `store`/`namespace` 기반 장기 메모리는 현재 노트북에서 아직 사용하지 않는다.

## 2. ConversationStore란 무엇인가

`ConversationStore`는 노트북에서 직접 만든 SQLite 기반 장기 메모리 저장소다.

저장 파일:

```python
agent_data/longterm_memory.sqlite
```

내부 테이블은 세 개다.

```text
turns
machine_values
summaries
```

### 2.1 turns

`turns`는 사용자와 assistant의 대화 이력을 저장한다.

저장 함수:

```python
add_turn(session_id, role, content)
```

조회 함수:

```python
recent_turns(session_id, limit=8)
```

저장되는 값:

```text
session_id
role
content
created_at
```

이 기능은 최근 대화를 컨텍스트로 요약할 때 사용된다.

### 2.2 machine_values

`machine_values`는 사용자가 입력한 제조 설비 feature 값을 저장한다.

저장 함수:

```python
add_machine_values(session_id, values)
```

조회 함수:

```python
latest_machine_values(session_id)
```

저장되는 값:

```text
session_id
name
value
unit
created_at
```

이 기능이 현재 `ConversationStore`에서 가장 중요하다.

예를 들어 첫 번째 턴에서 사용자가 이렇게 말했다고 하자.

```text
type L, rpm 1200, 토크 50, 공구마모 180이야.
```

그 다음 턴에서 사용자가 이렇게 물을 수 있다.

```text
그럼 토크만 70으로 바꾸면?
```

이때 현재 질문에는 `torque=70`만 있다. 하지만 이전 턴의 `type`, `rpm`, `tool_wear`가 `ConversationStore`에 저장되어 있으므로, 컨텍스트 엔지니어링 단계에서 현재 질문에 없는 값을 보완할 수 있다.

즉 `ConversationStore`는 단순한 채팅 로그가 아니라, "이전 설비 상태를 기억하고 다음 질문에서 재사용하는 장기 메모리" 역할을 한다.

### 2.3 summaries

`summaries`는 이전 예측 결과와 안전 판단 요약을 저장한다.

저장 함수:

```python
add_summary(session_id, kind, content)
```

조회 함수:

```python
latest_summary(session_id, kind)
```

현재 사용하는 `kind`는 다음과 같다.

```text
prediction
safety
```

예를 들어 이전 턴에서 HDF 위험이 높다고 판단했다면, 다음 턴에서 그 요약을 참고할 수 있다.

## 3. RunStore란 무엇인가

`RunStore`는 실행 이력을 저장한다.

저장 파일은 `ConversationStore`와 같은 SQLite DB를 사용한다.

```python
agent_data/longterm_memory.sqlite
```

테이블:

```text
runs
```

저장 함수:

```python
run_store.save(request_id, session_id, trace)
```

저장되는 값:

```text
request_id
session_id
trace_json
created_at
```

현재는 주로 gate 결과와 retry count를 저장한다.

```python
{
  "gate_reports": ...,
  "retry_counts": ...
}
```

이것은 사용자 장기 기억보다는 디버깅, 감사, 실행 관측에 가깝다.

## 4. long_term_saver란 무엇인가

`long_term_saver`는 LangGraph의 checkpointer다.

현재 코드에서는 다음과 같이 만들어진다.

```python
from langgraph.checkpoint.sqlite import SqliteSaver

_ctx_mgr = SqliteSaver.from_conn_string(CHECKPOINT_DB)
long_term_saver = _ctx_mgr.__enter__()
app = build_graph(checkpointer=long_term_saver)
```

저장 파일:

```python
agent_data/checkpoints.sqlite
```

이 저장소는 대화 기억을 사람이 읽기 좋은 형태로 저장하는 것이 아니라, LangGraph 실행 state를 저장한다.

예를 들어 다음과 같은 state가 저장된다.

```text
request_id
session_id
user_message
context_packet
agent_contexts
prediction_result
evidence_bundle
safety_decision
final_answer
gate_reports
retry_counts
```

따라서 `long_term_saver`는 "장기 대화 메모리"라기보다 "장기 checkpoint"라고 보는 것이 정확하다.

### 4.1 SqliteSaver와 MemorySaver의 차이

현재 코드에는 두 종류의 saver가 있다.

```python
short_term_saver = MemorySaver()
```

그리고 SQLite saver가 설치되어 있으면:

```python
long_term_saver = SqliteSaver.from_conn_string(CHECKPOINT_DB)
```

차이는 다음과 같다.

```text
MemorySaver
  - 메모리 안에만 저장
  - 프로세스가 종료되면 사라짐
  - 단기 checkpoint

SqliteSaver
  - SQLite 파일에 저장
  - 노트북/프로세스가 종료되어도 남음
  - 장기 checkpoint
```

현재 그래프는 `long_term_saver`로 compile된다.

```python
app = build_graph(checkpointer=long_term_saver)
```

따라서 SQLite saver가 정상 사용 가능하면, 현재 코드는 checkpoint 관점에서는 장기 저장 구조다.

## 5. thread_id는 왜 필요한가

`thread_id`는 LangGraph checkpointer가 상태를 구분하기 위한 ID다.

현재 코드:

```python
config = {"configurable": {"thread_id": session_id}}
result = app.invoke(state_in, config=config)
```

즉 현재는 `session_id`를 그대로 `thread_id`로 사용하고 있다.

`thread_id`가 필요한 이유는 LangGraph가 여러 대화 흐름을 구분해야 하기 때문이다.

예:

```text
thread_id = "chat_1"
  -> chat_1의 checkpoint state

thread_id = "chat_2"
  -> chat_2의 checkpoint state
```

만약 `thread_id`가 없으면 checkpointer는 어떤 대화의 상태를 이어가야 하는지 알 수 없다.

### 5.1 thread_id는 user_id와 다르다

`thread_id`는 보통 "대화 thread" 또는 "실행 흐름"을 의미한다.

반면 `user_id`는 "장기 기억의 소유자"를 의미한다.

예:

```text
user_id = "kim"

thread_id = "kim_chat_001"
thread_id = "kim_chat_002"
```

같은 유저가 여러 대화 thread를 가질 수 있다. 이 경우 checkpoint는 thread별로 나뉘지만, 장기 기억은 user별로 공유되는 것이 자연스럽다.

## 6. user_id, session_id, thread_id는 모두 필요한가

항상 세 개가 모두 필요한 것은 아니다.

표준적으로는 다음처럼 구분한다.

```text
user_id
  - 장기 기억 owner
  - LangGraph Store namespace에 사용

thread_id
  - LangGraph checkpoint 구분자
  - 대화 thread/실행 흐름 단위

session_id
  - 애플리케이션 세션 ID
  - 필요하면 thread_id와 같게 둘 수 있음
```

LangGraph 공식 패턴에서도 단기 메모리는 checkpointer와 `thread_id`로 관리하고, 장기 메모리는 store와 `user_id` 기반 namespace로 관리한다.

작은 데모나 노트북에서는 다음처럼 단순화해도 된다.

```text
session_id = thread_id = user_id처럼 사용
```

현재 노트북이 이 방식에 가깝다.

하지만 실제 앱이나 유저별 장기 기억이 중요한 구조에서는 다음처럼 분리하는 것이 좋다.

```text
user_id   = 장기 기억 기준
thread_id = checkpoint 기준
session_id = 앱 세션 기준 또는 thread_id와 동일
```

실무적으로는 최소한 다음 두 개는 분리하는 것이 좋다.

```text
user_id
thread_id
```

그리고 `session_id`는 앱에서 별도 의미가 있을 때만 둔다.

## 7. 현재 코드에서 장기 기억은 유저별로 분리되는가

현재 코드는 유저별 분리라기보다 `session_id`별 분리다.

`ConversationStore`의 모든 주요 조회/저장은 `session_id`를 기준으로 한다.

```python
conversation_store.add_turn(sid, ...)
conversation_store.latest_machine_values(session_id)
conversation_store.recent_turns(session_id)
conversation_store.latest_summary(session_id, ...)
```

따라서 `session_id`를 유저별로 다르게 주면 유저별처럼 분리된다.

하지만 엄밀히 말하면:

```text
유저별 메모리 X
session_id별 메모리 O
```

이다.

문제는 같은 유저가 여러 세션을 만들었을 때다.

```text
user_id = kim
session_id = kim_session_1
session_id = kim_session_2
```

현재 구조에서는 두 세션의 장기 기억이 서로 공유되지 않는다.

따라서 유저별 장기 기억이 필요하다면 `ConversationStore`에 `user_id` 컬럼을 추가하거나, LangGraph `Store`를 `user_id` namespace 기준으로 추가하는 것이 좋다.

## 8. 현재 컨텍스트 엔지니어링 흐름

현재 노트북의 컨텍스트 엔지니어링은 다음 흐름으로 되어 있다.

```text
ConversationStore 조회
  -> select_context()
  -> normalize_context()
  -> pack_contexts()
  -> agent_contexts로 agent별 전달
```

핵심은 이전 대화를 전부 프롬프트에 넣지 않는다는 점이다.

현재 코드의 정책:

```text
1. ContextManager는 항상 실행한다.
2. 전체 이전 대화를 Agent에게 그대로 전달하지 않는다.
3. 현재 입력값이 이전 입력값보다 우선한다.
4. 현재값이 없는 feature만 이전 대화에서 보완한다.
5. 이전 citation은 재사용하지 않는다.
6. EvidenceAgent는 현재 질문 기준으로 문서를 다시 검색한다.
7. prompt injection성 context는 제거한다.
8. Safety 이전 판단은 참고만 하고 현재 질문 기준으로 재판단한다.
9. 오래된 센서값은 stale 표시한다.
10. token budget 초과 시 설비값/직전 PredictionResult/SafetyDecision 요약을 우선한다.
```

### 8.1 Selector

`select_context()`는 현재 질문과 장기 메모리에서 필요한 정보를 고른다.

```python
current_vals = extract_machine_values(user_message)
previous_vals = store.latest_machine_values(session_id)
recent = store.recent_turns(session_id, limit=6)
clean_recent = [t for t in recent if not detect_injection(t["content"])]
```

가져오는 정보:

```text
현재 질문에서 추출된 설비값
이전 설비값의 feature별 최신값
최근 대화 일부
이전 prediction 요약
이전 safety 요약
현재 입력의 prompt injection 의심 여부
```

### 8.2 Normalizer

`normalize_context()`는 현재값과 이전값을 병합한다.

핵심 규칙:

```text
현재값 우선
현재값이 없는 feature만 이전값으로 보완
이전값은 stale 표시
현재값과 이전값이 충돌하면 warning 추가
```

예:

```text
이전 기억:
  rpm = 1200
  torque = 40
  tool_wear = 180

현재 질문:
  토크만 60이면?

결과:
  torque = 60       current
  rpm = 1200        previous/stale
  tool_wear = 180   previous/stale
```

### 8.3 Packer

`pack_contexts()`는 전체 컨텍스트를 agent별로 나눠서 포장한다.

`prediction_agent`에게는 다음이 전달된다.

```text
features
missing
sources
stale
```

`evidence_agent`에게는 현재 구현상 주로 warning이 전달된다.

`safety_agent`에게는 이전 safety summary가 전달된다.

`final_answer`에게는 최근 대화 요약과 warning이 전달된다.

이 구조의 장점은 agent마다 필요한 컨텍스트만 주입한다는 것이다.

## 9. 현재 컨텍스트 엔지니어링의 장점

현재 구조의 장점은 다음과 같다.

```text
이전 대화 전체를 무작정 넣지 않음
현재값과 이전값의 우선순위가 명확함
설비값을 구조화해서 관리함
이전값을 stale로 표시함
prompt injection 의심 발화를 제거함
agent별로 컨텍스트를 다르게 포장함
EvidenceAgent가 이전 citation을 재사용하지 않고 새로 검색함
```

특히 제조 설비 agent에서는 "토크만 바꾸면?" 같은 질문이 자주 나올 수 있으므로, 현재값 없는 feature를 이전 메모리에서 보완하는 구조가 매우 유용하다.

## 10. 현재 컨텍스트 엔지니어링의 보완점

### 10.1 user_id와 session_id 분리

현재는 `session_id`가 장기 기억 키와 checkpoint key 역할을 모두 한다.

보완 방향:

```text
user_id   -> 장기 기억 기준
thread_id -> checkpoint 기준
session_id -> 필요하면 앱 세션 기준
```

### 10.2 stale 시간 정책 강화

현재는 이전값이면 무조건 `is_stale=True`만 붙는다.

보완 방향:

```text
10분 이내   -> 사용 가능
1시간 이내  -> warning과 함께 사용
1시간 초과  -> 예측에는 사용하지 않고 재입력 요청
```

제조 설비에서는 센서값의 freshness가 중요하므로, 단순 stale flag만으로는 부족하다.

### 10.3 feature 범위 검증과 단위 처리

현재는 값 추출 후 범위 검증이 약하다.

보완해야 할 것:

```text
feature별 허용 범위
단위 변환: Celsius/Kelvin, rpm, Nm
비현실적 값 차단
음수 torque 같은 invalid value 차단
```

예:

```text
rpm = 999999
torque = -5
air_temperature = 30C인지 30K인지 모호함
```

이런 값은 예측 전에 validation이 필요하다.

### 10.4 최근 대화 요약 개선

현재 `recent_summary`는 최근 3개 발화 앞 40자를 단순히 이어붙인다.

```python
recent_summary = " | ".join(...)
```

보완 방향:

```text
running summary
confirmed facts
user corrections
open questions
```

같은 구조화된 요약을 저장하면 더 안정적인 컨텍스트가 된다.

### 10.5 EvidenceAgent 컨텍스트 확장

현재 `evidence_agent`에는 warning 중심의 컨텍스트만 전달된다.

보완 방향:

```text
current_question
predicted_failure_types
risk_level
selected_machine_values
missing_features
```

예를 들어 HDF 위험이 높게 나온 경우, EvidenceAgent가 HDF 관련 문서를 우선 검색하도록 만들 수 있다.

### 10.6 SafetyAgent 컨텍스트 확장

현재 `safety_agent`는 이전 safety summary만 참고한다.

보완 방향:

```text
prediction_result
risk_level
failure_types
stale_features
missing_features
current user intent
previous_safety_summary
```

특히 사용자가 "그래도 계속 운전해도 돼?"라고 물을 때는 이전 판단보다 현재 위험 상태가 우선되어야 한다.

### 10.7 prompt injection 방어 강화

현재는 정규식 기반 패턴으로 injection을 탐지한다.

보완 방향:

```text
instruction-like content 제거
사용자 발화와 시스템/정책 지시 분리
retrieved document를 untrusted context로 취급
safety policy는 사용자 발화로 override 불가
```

## 11. LangGraph Store란 무엇인가

LangGraph `Store`는 공식 장기 메모리 인터페이스다.

대표적으로 다음 저장소가 있다.

```text
InMemoryStore
SqliteStore
PostgresStore
```

현재 환경에서는 `SqliteStore`도 사용 가능하다.

```python
from langgraph.store.sqlite import SqliteStore, AsyncSqliteStore
```

기본 사용 방식:

```python
namespace = ("users", user_id, "memories")
store.put(namespace, "memory_1", {"data": "사용자는 HDF 설명을 자주 요청한다."})
memories = store.search(namespace, query="HDF 위험")
```

그래프에 붙일 때는 다음처럼 compile할 수 있다.

```python
graph = builder.compile(
    checkpointer=checkpointer,
    store=store,
)
```

중요한 점은 `SqliteSaver`와 `SqliteStore`가 다르다는 것이다.

```text
SqliteSaver
  - checkpointer
  - thread_id 기준 그래프 state 저장

SqliteStore
  - long-term store
  - user_id + namespace 기준 장기 memory 저장
```

## 12. ConversationStore와 LangGraph Store의 차이

둘 다 장기 메모리처럼 보이지만, 강점이 다르다.

```text
ConversationStore
  - 도메인 상태 DB
  - 구조화된 제조 설비값에 강함

LangGraph Store
  - 유저 장기 기억 DB
  - user_id + namespace 기반 memory에 강함
```

비교:

```text
ConversationStore
  장점:
    - 설비값, prediction summary, safety summary처럼 구조화된 데이터에 적합
    - SQL로 feature별 최신값 조회가 쉬움
    - 디버깅과 감사가 쉬움
    - 제조 도메인에 맞게 테이블을 설계할 수 있음

LangGraph Store
  장점:
    - LangGraph 공식 장기 메모리 패턴
    - user_id + namespace 분리가 자연스러움
    - 검색 가능한 memory에 적합
    - InMemoryStore, SqliteStore, PostgresStore 등으로 확장 가능
    - Runtime을 통해 node 내부에 store 주입 가능
```

### 12.1 ConversationStore가 더 적합한 데이터

```text
최근 rpm 값
최근 torque 값
feature별 최신 설비값
prediction summary
safety summary
실행 trace
gate report
```

이런 데이터는 테이블 기반으로 관리하는 것이 명확하다.

### 12.2 LangGraph Store가 더 적합한 데이터

```text
사용자의 답변 선호
사용자가 자주 묻는 고장 유형
사용자의 기본 설비 환경
사용자가 기억하라고 명시한 사실
장기적으로 검색해야 하는 user memory
```

이런 데이터는 namespace 기반 memory store가 자연스럽다.

## 13. 둘을 같이 쓰는 것이 좋은가

현재 프로젝트 기준으로는 둘을 구분해서 같이 쓰는 것이 가장 깔끔하다.

추천 구조:

```text
SqliteSaver
  -> LangGraph checkpoint
  -> thread_id 기준

ConversationStore
  -> 제조 도메인 데이터
  -> user_id/session_id 기준

SqliteStore
  -> 유저 장기 memory
  -> user_id + namespace 기준
```

파일 예:

```text
agent_data/checkpoints.sqlite
  -> SqliteSaver

agent_data/longterm_memory.sqlite
  -> ConversationStore

agent_data/user_memory.sqlite
  -> LangGraph SqliteStore
```

이렇게 하면 역할이 명확하다.

```text
ConversationStore는 정확한 도메인 상태에 강하고,
LangGraph Store는 유저별 검색 가능한 장기 기억에 강하다.
```

## 14. 컨텍스트 엔지니어링에 ConversationStore가 필수인가

필수는 아니다.

컨텍스트 엔지니어링에 필수인 것은 특정 저장소가 아니라 다음 흐름이다.

```text
필요한 정보 수집
-> 관련 있는 것만 선택
-> 충돌/오래됨/누락 정리
-> agent나 LLM에 맞게 포장
```

즉 필수 컴포넌트는 저장소가 아니라 다음에 가깝다.

```text
Selector
Normalizer
Packer
Policy
```

`ConversationStore`는 이 흐름에 필요한 데이터를 가져오는 수단 중 하나다.

다른 방식도 가능하다.

```text
LangGraph state만 사용
checkpointer에서 messages를 가져와 trim/summarize
LangGraph Store에서 user memory 검색
Vector DB에서 관련 문서 검색
외부 DB/API에서 실시간 상태 조회
현재 입력만 사용
```

제조 설비처럼 구조화된 feature 값이 중요한 경우에는 `ConversationStore`가 매우 유용하다.

하지만 일반 챗봇이나 유저 선호 기억 중심 앱에서는 LangGraph Store만으로도 충분할 수 있다.

## 15. 현재 코드의 추천 발전 방향

현재 코드를 크게 무너뜨리지 않고 발전시키려면 다음 순서가 좋다.

### 15.1 1단계: ID 설계 정리

현재:

```text
session_id가 thread_id 역할도 하고 장기 기억 키 역할도 함
```

추천:

```text
user_id
thread_id
```

예:

```python
def run_turn(user_message: str, user_id: str, thread_id: str, request_id: str):
    config = {"configurable": {"thread_id": thread_id}}
```

`ConversationStore`는 `user_id` 또는 `user_id + thread_id` 기준으로 조회하게 바꾼다.

### 15.2 2단계: stale 정책 추가

`machine_values.created_at`을 활용해서 freshness를 계산한다.

예:

```text
fresh
stale_warning
expired
```

`expired`인 설비값은 예측에 자동 보완하지 않고 사용자에게 재입력을 요청한다.

### 15.3 3단계: feature validation 추가

feature별 검증 규칙을 둔다.

예:

```text
type: L/M/H만 허용
rotational_speed: 0보다 커야 함
torque: 0보다 커야 함
tool_wear: 0 이상
air_temperature/process_temperature: 단위 명확화
```

### 15.4 4단계: Evidence/Safety context 확장

EvidenceAgent에 prediction 결과와 고장 유형을 넘긴다.

SafetyAgent에 risk level, stale features, missing features를 넘긴다.

### 15.5 5단계: SqliteStore 추가

LangGraph `SqliteStore`를 추가해서 user memory를 분리한다.

예:

```text
("users", user_id, "preferences")
("users", user_id, "memories")
("users", user_id, "machine_defaults")
```

이때 `ConversationStore`를 없애는 것이 아니라, 역할을 나눈다.

```text
ConversationStore
  -> 구조화된 제조 도메인 데이터

SqliteStore
  -> 검색 가능한 유저 장기 memory
```

## 16. 최종 정리

현재 노트북은 다음처럼 이해하면 된다.

```text
ConversationStore
  = 제조 도메인 장기 메모리
  = 대화, 설비값, 이전 판단 요약 저장

long_term_saver / SqliteSaver
  = LangGraph checkpoint
  = thread_id 기준 그래프 state 저장/복원

thread_id
  = checkpoint를 구분하는 대화 thread ID

session_id
  = 현재 코드에서는 장기 메모리 키이자 thread_id로도 사용됨

LangGraph Store
  = user_id + namespace 기반 공식 장기 memory 저장소
  = 현재 코드에는 아직 미적용
```

권장 최종 구조:

```text
thread_id
  -> SqliteSaver checkpoint

user_id
  -> LangGraph SqliteStore namespace

ConversationStore
  -> 제조 설비 feature, prediction summary, safety summary, run trace
```

한 줄로 요약하면:

```text
ConversationStore는 정확한 도메인 상태를 위한 DB이고,
LangGraph Store는 유저별 검색 가능한 장기 기억을 위한 DB이며,
checkpointer는 그래프 실행 상태를 저장하는 장치다.
```

