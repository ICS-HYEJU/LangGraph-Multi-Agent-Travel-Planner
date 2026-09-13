# Multi-Agent Travel Planner

## 1. 프로젝트 개요

이 프로젝트는 사용자의 여행 요청을 바탕으로 개인화된 여행 계획을 생성하는 한국어 멀티에이전트 여행 플래너입니다.

단일 LLM이 바로 여행 일정을 작성하는 방식이 아니라, 여러 Agent가 역할을 나누어 다음 과정을 수행합니다.

- 사용자 요청 분석
- 여행 선호 정보 구조화
- 지역 추천 정보 생성
- 여행 계획 생성
- 계획 검토
- 수정 또는 추가 질문
- 최종 응답 생성

전체 실행 흐름은 LangGraph Orchestrator가 제어합니다.

---

## 2. 핵심 구조

시스템은 총 4개의 Agent로 구성됩니다.

```text
Agent 1: User Chat Manager
Agent 2: Plan Agent
Agent 3: Critic Agent
Agent 4: Local Expert / Recommender Agent
```

### Agent 1: User Chat Manager

사용자-facing 대화 관리자입니다.

주요 역할:

- 사용자 자연어 요청 분석
- 여행지, 여행 기간, 관심사 등 구조화
- 초기 필수 정보 부족 여부 판단
- 사용자에게 추가 질문 생성
- 최종 응답 정리

Agent 1은 전체 내부 흐름을 직접 제어하지 않습니다.  
전체 흐름 제어는 LangGraph Orchestrator가 담당합니다.

### Agent 2: Plan Agent

여행 계획을 생성하는 Agent입니다.

주요 역할:

- 사용자 선호 기반 여행 계획 생성
- Agent 4의 지역 추천 정보 반영
- Agent 3의 피드백을 반영하여 계획 수정

### Agent 3: Critic Agent

Agent 2가 생성한 여행 계획을 검토하는 Agent입니다.

검토 결과는 반드시 다음 중 하나입니다.

```text
approved
need_revision
need_user_info
```

Agent 3는 계획을 직접 수정하지 않고, 피드백 또는 추가 질문 필요 여부만 반환합니다.

### Agent 4: Local Expert / Recommender Agent

지역 추천 정보를 제공하는 Agent입니다.

주요 역할:

- 관광지 추천
- 맛집 추천
- 숙소 권역 추천
- 이동 팁 제공
- 주의사항 제공

Agent 4는 전체 여행 계획을 직접 작성하지 않습니다.

---

## 3. 전체 실행 흐름

기본 흐름은 다음과 같습니다.

```text
User Input
   ↓
Agent 1: User Chat Manager
   ↓
LangGraph routing
   ↓
Agent 4: Local Expert / Recommender Agent
   ↓
Agent 2: Plan Agent
   ↓
Agent 3: Critic Agent
   ↓
Final Response
```

Agent 3가 `need_revision`을 반환하면 Agent 2가 계획을 다시 수정합니다.

```text
Agent 2 → Agent 3 → Agent 2
```

이 수정 루프는 `MAX_REVISION_COUNT`까지만 허용됩니다.

---

## 4. 모델 서버 구조

이 프로젝트는 두 개의 OpenAI-compatible LLM endpoint를 사용하는 구조를 가정합니다.

```text
Light model server
→ Agent 1
→ Agent 4

Heavy model server
→ Agent 2
→ Agent 3
```

모델 서버 주소, 모델명, API key는 코드에 직접 작성하지 않습니다.  
모든 설정은 `.env` 또는 환경변수에서 읽습니다.

---

## 5. 프로젝트 구조

```text
multi-agent-travel-planner/
├─ AGENTS.md
├─ README.md
├─ .env.example
├─ .gitignore
├─ pyproject.toml
├─ requirements.txt
├─ docs/
│  ├─ codex_task.md
│  ├─ code_writing_plan.md
│  ├─ architecture.md
│  ├─ agent_io_schema.md
│  └─ design_summary.md
├─ references/
├─ src/
│  └─ travel_agent_system/
│     ├─ __init__.py
│     ├─ main.py
│     ├─ config.py
│     ├─ llm_clients.py
│     ├─ graph.py
│     ├─ schemas.py
│     ├─ prompts/
│     ├─ agents/
│     ├─ services/
│     └─ utils/
└─ tests/
```

---

## 6. 환경 설정

### 6.1 Python 버전

Python 3.10 이상을 사용합니다.

### 6.2 패키지 설치

```bash
pip install -r requirements.txt
```

또는 개발 모드로 설치할 경우:

```bash
pip install -e .
```

### 6.3 환경변수 설정

먼저 `.env.example`을 복사하여 `.env`를 만듭니다.

Windows cmd:

```cmd
copy .env.example .env
```

Linux 또는 macOS:

```bash
cp .env.example .env
```

그다음 `.env` 파일에 실제 서버 주소와 모델명을 입력합니다.

예시:

```env
LIGHT_LLM_BASE_URL=http://SERVER2_IP:8000/v1
LIGHT_LLM_MODEL=light-model-name
LIGHT_LLM_API_KEY=dummy

HEAVY_LLM_BASE_URL=http://SERVER4_IP:8000/v1
HEAVY_LLM_MODEL=heavy-model-name
HEAVY_LLM_API_KEY=dummy

REQUEST_TIMEOUT=60
MAX_REVISION_COUNT=2
MAX_USER_QUESTION_COUNT=2
LOG_DIR=logs
```

주의:

- `.env` 파일은 GitHub에 올리지 않습니다.
- 실제 서버 IP, API key, 모델명은 코드에 직접 작성하지 않습니다.
- `.env.example`은 템플릿 용도로만 사용합니다.

---

## 7. 실행 방법

구현 완료 후 CLI에서 다음과 같이 실행합니다.

```bash
python -m travel_agent_system.main "부산 2박 3일 맛집 위주로 여유롭게 여행 계획 짜줘"
```

중간 Agent 결과를 보고 싶을 경우 `--debug` 옵션을 사용할 수 있습니다.

```bash
python -m travel_agent_system.main "부산 2박 3일 맛집 위주로 여유롭게 여행 계획 짜줘" --debug
```

기본 실행에서는 최종 응답만 출력합니다.

---

## 8. 테스트

테스트는 다음 명령어로 실행합니다.

```bash
pytest
```

테스트 목표:

- Schema validation 확인
- State merge 동작 확인
- LangGraph routing 동작 확인
- 실제 LLM 호출 없이 mock 기반 Agent I/O 확인

---

## 9. 주요 문서

| 파일 | 설명 |
|---|---|
| `AGENTS.md` | Codex가 따라야 할 프로젝트 규칙 |
| `docs/codex_task.md` | 전체 구현 작업지시서 |
| `docs/code_writing_plan.md` | 코드 작성 순서와 구현 단계 |
| `docs/architecture.md` | 시스템 구조와 Agent 흐름 |
| `docs/agent_io_schema.md` | Agent 간 입출력 schema |
| `docs/design_summary.md` | 전체 설계 배경과 논의 정리 |

---

## 10. 구현 원칙

- Agent 역할을 섞지 않습니다.
- Agent 1은 사용자-facing manager입니다.
- LangGraph Orchestrator가 전체 흐름을 제어합니다.
- Agent 3는 계획을 직접 수정하지 않고 피드백만 제공합니다.
- Agent 4는 전체 계획을 작성하지 않고 지역 추천 정보만 제공합니다.
- 사용자-facing 출력은 한국어로 작성합니다.
- Agent 간 전달 정보는 가능한 JSON 또는 Pydantic schema 기반으로 구조화합니다.
- 실제 LLM 서버 정보는 `.env`에서 관리합니다.

---

## 11. 현재 개발 단계

현재 저장소는 Codex 구현 작업을 시작하기 위한 초기 구조와 문서를 준비하는 단계입니다.

초기 Codex 작업은 다음 범위부터 시작하는 것을 권장합니다.

```text
Phase 1: 프로젝트 skeleton 확인 및 정리
Phase 2: schemas.py 구현
```

처음부터 전체 시스템을 한 번에 구현하지 않고, 단계별로 구현합니다.