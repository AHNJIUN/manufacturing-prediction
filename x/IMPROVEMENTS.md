# 보완 사항 (Manufacturing AI Agent)

> 대상: `manufacturing_agent.ipynb` — README 설계서를 LangGraph로 구현한 실행 가능한 스켈레톤.
> 현재 상태: **아키텍처/흐름/Context Engineering/메모리 3계층/Gate는 end-to-end로 동작**한다.
> 다만 핵심 판단 로직 다수가 **데모용 stub/규칙 기반**이므로, 실서비스로 가려면 아래를 보완해야 한다.

---

## 0. 현재 구현 상태 요약

| 영역 | 상태 | 비고 |
|------|------|------|
| 그래프/라우팅/Gate | ✅ 동작 | 선형 + conditional edge, retry/​block 분기 포함 |
| Context Engineering | ✅ 동작 | 현재값 우선·stale 표시·injection 무력화 검증됨 |
| 단기 메모리(MemorySaver) | ✅ 동작 | thread(세션) working state |
| 장기 메모리(SqliteSaver + SQLite Store) | ✅ 동작 | 대화/설비값/요약 영속 |
| 벡터 스토어(ChromaDB) | ⚠️ 폴백 포함 | 미설치 시 키워드 검색, 시드 문서 7건뿐 |
| PredictionAgent | ⚠️ 규칙 기반 | 실제 ML 모델 아님 |
| EvidenceAgent(RAG) | ⚠️ 단순 | query fan-out만, grading/rerank 없음 |
| SafetyAgent | ⚠️ 정규식 | 패턴 매칭 기반 |
| LLM 호출 | ⚠️ Stub 폴백 | API 키 없으면 결정론적 요약 |

---

## 1. 핵심 판단 로직 (최우선)

### 1.1 PredictionAgent — 실제 ML 모델로 교체
- 현재 `prediction_service.compute_partial_risks()`는 AI4I 임계값 휴리스틱(규칙 기반)이다.
- **보완**: AI4I 2020 데이터셋으로 학습한 분류기(예: XGBoost/LightGBM)를 `predict_proba`로 호출.
  - TWF/HDF/PWF/OSF/RNF 멀티라벨 또는 고장유형별 이진 모델.
  - 모델 아티팩트 버전 관리(MLflow 등), 입력 feature 스케일링/검증.
  - 오차범위(confidence interval)를 휴리스틱이 아니라 모델 기반(분위수/캘리브레이션)으로 산출.
- **누락값 처리**: 현재는 부분 위험만 계산. 모델이 누락 feature에 민감하므로
  "몰래 평균 대입 금지"(README) 원칙 유지 + 부분 모델/대치 전략을 명시적으로 분리.

### 1.2 LLM 출력의 구조화(Structured Output)
- 현재 agent들은 LLM을 **요약 문장 생성**에만 쓰고, 핵심 결과는 코드가 만든다.
- **보완**: LLM이 판단에 더 관여한다면 `PredictionResult`/`SafetyDecision` 스키마를
  **tool calling / structured output**(pydantic 스키마 강제)으로 받아 파싱 오류·환각을 차단.
- `call_llm()`을 `call_llm_structured(schema, ...)`로 확장하고 검증 실패 시 재시도(self-heal).

### 1.3 SafetyAgent — 정규식 → 정책 + LLM 하이브리드
- 현재 `FORBIDDEN_PATTERNS`/`INJECTION_PATTERNS`는 한국어 정규식 몇 개뿐 → 우회 쉽다.
- **보완**:
  - 안전 정책을 구조화된 룰셋(YAML/DB)으로 외부화 + 버전 관리.
  - LLM 기반 의도 분류(위험 요청 탐지)를 정규식과 **2단계 방어**로 결합.
  - prompt injection 탐지는 전용 분류기/가드레일(예: Llama Guard류, 자체 분류기)로 강화.

---

## 2. RAG / EvidenceAgent

- **문서 코퍼스**: 현재 시드 7건뿐. 실제 매뉴얼/표준/사례를 적재하고 **청킹 + 메타데이터**(설비, 고장유형, 문서버전) 설계 필요.
- **임베딩 모델**: ChromaDB 기본 임베딩(all-MiniLM) 대신 도메인/한국어에 맞는 임베딩(예: 다국어 e5, OpenAI/Cohere embed)으로 교체.
- **Adaptive RAG 고도화**: 현재는 profile별 type 필터 + 단순 fan-out뿐.
  - query planning(LLM 기반 재작성/분해), **reranking**(cross-encoder), **evidence grading**(관련성/충분성 채점), 부족 시 `fallback_broad` 자동 확장 — README가 명시한 단계를 실제 구현.
- **Citation 신뢰성**: snippet-답변 정합성 검증(각 주장→근거 매핑), 환각 인용 차단.

---

## 3. 메모리 / Context Engineering

- **장기 기억 검색이 얕다**: `ConversationStore`는 *최신값/최근 N턴/마지막 요약*만 조회.
  - **보완**: 과거 대화의 **의미 기반 검색**(대화도 벡터화) + 주기적 **요약/압축(compaction)**으로 토큰 예산 관리.
- **Token budget**: README 정책 10번("budget 초과 시 우선순위")이 코드에 미구현. 실제 토크나이저로 길이 측정 후 절삭 로직 추가.
- **단위/타입 정규화 미흡**: `context_normalizer`가 단위 변환(℃↔K, rpm 등)을 거의 안 한다. 단위 사전 + 변환기 필요.
- **세션/스레드 설계**: 현재 `thread_id=session_id`로 단기 working state를 영속. 운영에선
  - 사용자/설비/세션 키 체계 정립, 멀티 유저 격리, TTL/만료 정책.
- **SqliteSaver 리소스 관리**: 노트북은 `__enter__`로 열고 닫지 않는다(데모 OK).
  서비스에선 `with` 컨텍스트/명시적 close, 또는 `AsyncSqliteSaver`/Postgres 체크포인터로 전환.
- **동시성**: `sqlite3` 직접 사용은 다중 요청 동시 쓰기에 약하다. WAL 모드/커넥션 풀 또는 Postgres로 이전.

---

## 4. 그래프 / 라우팅 견고성

- **Supervisor가 얇다**: 현재 거의 고정 경로. README가 요구한 retry/redirect/clarification/safe-block 판단을
  Gate 결과(`route_hint`) 기반으로 Supervisor가 실제 해석하도록 확장.
- **retry 루프**: `_wrap_retry` 카운터는 있으나 gate가 retry를 거의 트리거하지 않음.
  실패 주입 테스트로 retry/​fallback 경로를 실제 검증.
- **Human-in-the-loop**: `risk_level == critical`이거나 BLOCK 시 LangGraph `interrupt`로
  사람 승인 단계를 추가(고위험 제조 도메인에서 중요).
- **에러 처리**: 노드별 try/except + 에러 터미널 노드(부분 결과로 안전 응답) 보강.

---

## 5. 관측성 / 운영

- **Observability**: `RunStore`에 trace를 남기지만 latency/token/LLM 호출 단위가 비어 있음.
  - LangSmith 또는 OpenTelemetry 연동, 노드별 지연/토큰/비용 기록.
- **스트리밍**: `app.stream()`으로 단계별 진행 노출(UX/디버깅).
- **설정 관리**: 모델명/임계값/경로가 코드 상수에 흩어져 있음 → `config.yaml`/환경변수로 외부화.
- **재현성**: `DEFAULT_MODEL`, 임베딩 모델, ML 모델 버전을 함께 고정·기록.

---

## 6. 품질 보증

- **평가 하니스 없음**: 시나리오별 골든셋(예측 정확도, 안전 차단 재현율, injection 차단율, 인용 정확도) + 회귀 테스트.
- **단위 테스트**: `extract_machine_values`(조사/단위 변형), `normalize_context`(충돌/stale), gate 분기 등 핵심 함수 pytest.
- **레드팀**: prompt injection / jailbreak 코퍼스로 SafetyAgent·InputGate 정량 평가.

---

## 7. 보안 / 컴플라이언스

- prompt injection·jailbreak 방어 강화(§1.3).
- 민감정보(PII/설비 기밀) 마스킹 정책을 `MemoryWriterNode` 저장 단계에 추가.
- 접근 제어/감사 로그(누가 어떤 설비를 진단·조회했는지).
- 고위험 조치(운전 지속/안전장치 관련)는 항상 사람 승인 + 정책 근거 인용 강제.

---

## 8. 우선순위 제안

| 순위 | 작업 | 이유 |
|------|------|------|
| P0 | AI4I ML 모델 연동(§1.1) + 구조화 출력(§1.2) | 예측 신뢰성이 제품의 핵심 |
| P0 | SafetyAgent 정책+LLM 하이브리드(§1.3) | 안전 도메인, 오탐/우회 위험 |
| P1 | RAG 코퍼스·임베딩·rerank·grading(§2) | 근거 품질이 답변 신뢰의 절반 |
| P1 | 평가/테스트 하니스(§6) | 이후 개선의 안전망 |
| P2 | 토큰 예산·의미 기반 장기기억(§3) | 멀티턴 확장성 |
| P2 | 관측성/HITL/Postgres 전환(§4·§5) | 운영 준비도 |
