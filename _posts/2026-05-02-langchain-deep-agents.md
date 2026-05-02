---
layout: post
title: "LangChain Deep Agents — 단순 챗봇을 넘어 진짜 일하는 AI 에이전트"
date: 2026-05-02 15:00:00 +0900
categories:
  - LLM
  - Agent
tags: [LangChain, DeepAgents, LangGraph, AIAgent, ContextEngineering, MCP]
math: false
---

> {: .prompt-tip }
> **요약:** LangChain Deep Agents는 2025년 7월 출시된 오픈소스 에이전트 프레임워크입니다. 기존 에이전트가 단순 도구 호출 루프에 그쳤다면, Deep Agents는 **계획(Planning), 파일시스템 기반 컨텍스트 관리, 서브에이전트 위임**을 내장해 복잡하고 장기적인 태스크를 처리합니다. Claude Code와 Deep Research를 역설계한 결과물로, 단 세 줄의 코드로 시작할 수 있습니다.

---

"AI 에이전트를 만들었는데 간단한 질문에는 잘 답하는데, 복잡한 작업을 시키면 금방 길을 잃어버려요."

LLM 기반 앱을 개발해본 사람이라면 한 번쯤 겪는 상황입니다. 검색 → 분석 → 정리 → 파일 저장처럼 여러 단계로 이어지는 작업을 시키면 에이전트는 중간에 맥락을 잃거나, 컨텍스트 윈도우가 넘쳐 앞서 수행한 작업을 잊어버립니다.

LLM을 루프 안에서 도구를 호출하게 하는 것은 에이전트의 가장 단순한 형태입니다. 하지만 이 구조는 "얕은(shallow)" 에이전트를 만들어, 더 길고 복잡한 태스크를 계획하고 실행하는 데 실패합니다.

이 문제를 해결하기 위해 LangChain이 내놓은 것이 바로 **Deep Agents**입니다.

---

## 1. Deep Agents란 무엇인가?

Deep Agents는 LangChain의 CEO인 Harrison Chase가 2025년 7월에 출시했습니다. 2026년 3월 대규모 업데이트 당시 단 5시간 만에 GitHub 스타 9,900개를 기록하며 커뮤니티의 뜨거운 관심을 받았습니다.

Deep Agents는 'agent harness'라고 불리는 독립적인 라이브러리로, LangChain의 에이전트 구성 요소 위에 만들어지고 LangGraph 런타임으로 구동됩니다. 새로운 추론 모델이나 별도의 런타임을 도입하는 것이 아니라, 표준 도구 호출 루프 주변에 기본값과 내장 도구들을 패키징한 것입니다.

한 마디로 정의하면 이렇습니다.

> **"복잡하고 긴 태스크를 처음부터 끝까지 스스로 해내는 에이전트를 만들기 위한 배터리 포함 프레임워크"**

### Claude Code에서 영감을 받다

이 프로젝트는 주로 Claude Code에서 영감을 받았으며, 초기에는 Claude Code를 범용적으로 만드는 요소가 무엇인지 파악하고 그것을 더 발전시키려는 시도였습니다.

Harrison Chase는 Anthropic의 Claude Code를 역설계해 에이전트를 진정으로 효과적으로 만드는 것이 무엇인지 파악했습니다. 그의 결론은 이것입니다. 차이를 만드는 것은 모델 자체가 아니라, **모델을 둘러싼 아키텍처**라는 것입니다.

"에이전트가 실수를 할 때는 올바른 컨텍스트를 갖지 못해서입니다. 성공할 때는 올바른 컨텍스트를 가지고 있기 때문입니다." — Harrison Chase, LangChain CEO

---

## 2. 기존 에이전트의 한계: 왜 "얕은" 에이전트가 되는가?

Deep Agents를 이해하려면 먼저 기존 에이전트의 구조적 한계를 알아야 합니다.

### 기존 에이전트의 작동 방식

일반적인 LLM 에이전트는 다음 루프를 반복합니다.

```
생각 → 도구 호출 → 결과 관찰 → 생각 → 도구 호출 → ...
```

이 구조는 간단한 질문-답변에는 충분하지만, 세 가지 치명적인 한계를 가집니다.

**한계 1: 계획이 없다**

복잡한 태스크를 받으면 에이전트는 바로 첫 번째 도구 호출로 뛰어들어 버립니다. 전체 작업을 단계로 나누는 명시적 계획이 없으니 중간에 방향을 잃기 쉽습니다.

**한계 2: 컨텍스트 윈도우가 넘친다**

검색 결과, 중간 분석, 생성된 코드가 모두 대화창에 쌓입니다. 작업이 길어질수록 컨텍스트 윈도우가 가득 차고, 에이전트는 앞서 수행한 작업을 잊어버립니다.

**한계 3: 복잡한 서브태스크를 위임할 수 없다**

모든 작업을 하나의 에이전트가 처리하다 보니, 서로 다른 성격의 태스크가 하나의 컨텍스트에 뒤섞입니다. 예를 들어 "리서치하고 코드도 작성해줘"를 동시에 하려면 두 역할의 컨텍스트가 충돌합니다.

---

## 3. Deep Agents의 4가지 핵심 구성 요소

Deep Research, Manus, Claude Code 같은 애플리케이션들은 **계획 도구, 서브에이전트, 파일시스템 접근, 상세한 프롬프트** 이 네 가지를 조합해 이 한계를 극복했습니다. deepagents는 이것을 범용적인 방식으로 구현해 누구나 쉽게 Deep Agent를 만들 수 있도록 한 패키지입니다.

### 구성 요소 1: 명시적 계획 (write_todos)

Deep Agents에는 `write_todos`라는 내장 계획 도구가 있습니다. 이를 통해 에이전트는 복잡한 태스크를 구체적인 단계로 분해하고, 진행 상황을 추적하며, 새로운 정보가 나타나면 계획을 수정할 수 있습니다.

기술적으로 `write_todos`는 **"no-op"** 도구입니다. 직접 어떤 작업을 수행하지 않습니다. 하지만 이것이 오히려 핵심입니다. 에이전트가 실행에 뛰어들기 전에 **반드시 생각을 구조화**하도록 강제합니다.

```
사용자: "최신 AI 논문을 조사하고 요약 리포트를 작성해줘"

에이전트의 첫 번째 행동 (write_todos):
  [ ] 1. arxiv에서 최신 AI 논문 검색
  [ ] 2. 각 논문의 핵심 내용 정리
  [ ] 3. 논문별 summary.md 파일 작성
  [ ] 4. 전체 리포트 통합 및 최종 저장

→ 이 계획이 가시적으로 유지되면서 에이전트가 길을 잃지 않음
```

이 단순한 no-op 계획 도구가 장기적 일관성을 극적으로 개선합니다.

### 구성 요소 2: 파일시스템 기반 컨텍스트 관리

파일시스템 도구들은 에이전트가 대용량 컨텍스트를 활성 프롬프트 윈도우 안에 유지하는 대신 스토리지로 오프로드할 수 있게 해줍니다. 에이전트는 메모, 생성된 코드, 중간 리포트, 검색 결과를 파일에 저장하고 나중에 다시 불러올 수 있습니다.

Deep Agents에 내장된 파일시스템 도구들:

| 도구 | 역할 |
| :--- | :--- |
| `write_file` | 중간 결과물을 파일로 저장 |
| `read_file` | 저장된 파일 다시 불러오기 |
| `edit_file` | 기존 파일 수정 |
| `ls` | 현재 파일 목록 확인 |
| `glob` | 패턴으로 파일 검색 |
| `grep` | 파일 내용 검색 |

```
[기존 에이전트 방식]
검색결과 A + 검색결과 B + 분석 C + 코드 D → 모두 대화창에 쌓임
→ 컨텍스트 윈도우 초과 → 앞 내용 잊어버림 ❌

[Deep Agents 방식]
검색결과 A → search_a.md 저장
검색결과 B → search_b.md 저장
분석 C    → analysis.md 저장
→ 필요할 때만 read_file로 불러옴
→ 컨텍스트 윈도우는 항상 여유 있음 ✅
```

Deep Agents는 이 가상 파일시스템에 대해 여러 백엔드를 지원합니다. StateBackend(기본값, LangGraph 상태에 임시 저장), FilesystemBackend, LocalShellBackend, StoreBackend, CompositeBackend가 있습니다.

추가로 컨텍스트 윈도우가 길어지면 오래된 대화 메시지를 자동으로 압축하는 자동 요약 기능도 내장되어 있어, 에이전트가 장시간 세션에서도 효과적으로 작동합니다.

### 구성 요소 3: 서브에이전트 위임 (task tool)

복잡한 서브태스크를 전문화된 에이전트에게 위임합니다. 이렇게 하면 컨텍스트가 격리됩니다. 메인 에이전트는 중간의 모든 검색이나 처리 단계가 아닌 최종 결과만 봅니다.

```python
# 메인 에이전트의 시각에서 서브에이전트 위임
agent = create_deep_agent(
    system_prompt="""
    리서치는 research-agent에게, 코딩은 code-agent에게 위임하세요.
    """,
    subagents=[research_subagent, code_subagent],
)

# 내부적으로 일어나는 일:
# [메인 에이전트] → task("research-agent", "AI 논문 5편 찾기")
#                         ↓
#               [research-agent] 10번 검색 수행
#                         ↓
#               메인 에이전트에게 깔끔한 요약 3단락 반환
# 메인 에이전트의 컨텍스트에는 최종 요약만 남음 ✅
```

별도 설정 없이도 모든 Deep Agent는 자동으로 범용 서브에이전트를 하나 갖습니다. 이는 메인 에이전트와 동일한 시스템 프롬프트, 동일한 도구, 동일한 모델을 가지며, 전문 서브에이전트를 정의하지 않아도 컨텍스트 격리를 위해 사용할 수 있습니다.

### 구성 요소 4: 상세한 시스템 프롬프트 (Context Engineering)

Deep Agents의 시스템 프롬프트는 여러 컴포넌트로 구성되어 동적으로 조립됩니다. 사용자 정의 system_prompt, 기본 에이전트 프롬프트, todo 도구 사용 가이드, 메모리 프롬프트(AGENTS.md), 스킬 목록, 가상 파일시스템 문서, 서브에이전트 사용 가이드, 미들웨어 프롬프트 등이 이 순서대로 조합됩니다. LLM은 매번 호출될 때 이 모든 컨텍스트를 봅니다.

이것이 "프롬프트 엔지니어링은 죽었다"는 이야기와 반대로 가는 지점입니다. Deep Agents는 오히려 **정교하게 설계된 시스템 프롬프트**가 에이전트 성능의 핵심 레버라는 것을 보여줍니다.

---

## 4. 전체 아키텍처 한눈에 보기

```
┌─────────────────────────────────────────────────────────┐
│                    Deep Agent                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  시스템 프롬프트 (동적 조립)                     │   │
│  │  사용자 지시 + 도구 가이드 + 메모리 + 스킬       │   │
│  └─────────────────────────────────────────────────┘   │
│                        ↕                               │
│  ┌──────────┐  ┌─────────────────┐  ┌──────────────┐  │
│  │ 계획 도구 │  │ 파일시스템 도구  │  │ 서브에이전트  │  │
│  │write_todos│  │read/write/edit  │  │  task tool   │  │
│  └──────────┘  └─────────────────┘  └──────────────┘  │
│                        ↕                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  LangGraph 런타임                               │   │
│  │  (스트리밍 / 상태 관리 / 지속성 / HITL)         │   │
│  └─────────────────────────────────────────────────┘   │
│                        ↕                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │ 장기 메모리│  │ 자동 요약 │  │  MCP 도구 연동      │  │
│  │ (크로스    │  │(컨텍스트  │  │  (외부 서비스)      │  │
│  │  스레드)  │  │ 압축)    │  │                    │  │
│  └──────────┘  └──────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                        ↕
              어떤 LLM이든 연결 가능
         (GPT-5, Claude, Gemini, Llama, ...)
```

---

## 5. 코드로 살펴보기

### 가장 빠른 시작 — 단 3줄

```python
from deepagents import create_deep_agent

agent = create_deep_agent()

result = agent.invoke({
    "messages": [
        {"role": "user", "content": "LangGraph를 조사하고 요약을 summary.md에 저장해줘"}
    ]
})
```

에이전트는 계획을 세우고, 파일을 읽고 쓰며, 스스로 컨텍스트를 관리합니다. 이 한 번의 호출로 에이전트는 내부적으로 검색 쿼리를 생성하고, 결과를 파일에 저장하고, 정리된 요약을 작성합니다.

### 모델 · 도구 · 프롬프트 커스터마이징

```python
from langchain.chat_models import init_chat_model
from deepagents import create_deep_agent

# 커스텀 도구 정의
def search_papers(query: str) -> str:
    """arXiv에서 AI 논문을 검색합니다."""
    # 실제 구현
    return f"'{query}'에 대한 논문 검색 결과..."

def get_citation_count(paper_id: str) -> int:
    """논문의 인용 수를 반환합니다."""
    return 42

# Deep Agent 생성
agent = create_deep_agent(
    # 어떤 LLM이든 사용 가능
    model=init_chat_model("openai:gpt-4o"),

    # 커스텀 도구 추가
    tools=[search_papers, get_citation_count],

    # 역할 지정
    system_prompt="당신은 AI 분야 리서치 어시스턴트입니다.",
)

result = agent.invoke({
    "messages": [{"role": "user",
                  "content": "2026년 최신 멀티모달 LLM 논문 5편을 찾아 비교 분석해줘"}]
})

# 최종 응답
print(result["messages"][-1].content)

# 에이전트가 생성한 파일 목록
print(result.get("files", {}))
```

### 전문 서브에이전트 구성

```python
from langchain.chat_models import init_chat_model
from deepagents import create_deep_agent

# 리서치 전문 서브에이전트
research_agent = create_deep_agent(
    model=init_chat_model("openai:gpt-4o"),
    tools=[search_papers],
    system_prompt="당신은 학술 리서치 전문가입니다. 논문을 찾고 핵심 내용을 정리합니다.",
)

# 코드 작성 전문 서브에이전트
code_agent = create_deep_agent(
    model=init_chat_model("openai:gpt-4o"),
    tools=[],
    system_prompt="당신은 Python 전문 개발자입니다. 코드를 작성하고 문서화합니다.",
)

# 두 서브에이전트를 가진 오케스트레이터
orchestrator = create_deep_agent(
    model=init_chat_model("openai:gpt-4o"),
    system_prompt="""
    당신은 프로젝트 매니저 에이전트입니다.
    - 리서치 작업은 반드시 research-agent에게 위임하세요
    - 코드 작성은 반드시 code-agent에게 위임하세요
    - 결과를 통합해 최종 리포트를 작성합니다
    """,
    subagents=[research_agent, code_agent],
)

result = orchestrator.invoke({
    "messages": [{"role": "user",
                  "content": "RAG 시스템을 조사하고, 간단한 Python 구현 코드도 작성해줘"}]
})
```

### 장기 메모리 설정 (크로스 스레드)

```python
from langgraph.store.memory import InMemoryStore
from deepagents import create_deep_agent

# 세션이 바뀌어도 기억 유지
memory_store = InMemoryStore()

agent = create_deep_agent(
    model="openai:gpt-4o",
    # 메모리 스토어 연결 — 이전 대화 내용을 기억
    store=memory_store,
)

# 첫 번째 대화
agent.invoke({
    "messages": [{"role": "user", "content": "내 이름은 김철수이고 Python 개발자야"}],
    "thread_id": "user_001",   # 사용자 식별자
})

# 나중 대화 (다른 세션이지만 같은 thread_id)
result = agent.invoke({
    "messages": [{"role": "user", "content": "내가 어떤 개발자라고 했지?"}],
    "thread_id": "user_001",
})
# → "Python 개발자라고 하셨습니다" 정확하게 기억
```

### CLI로 바로 사용하기

Deep Agents는 SDK뿐만 아니라 터미널에서 바로 쓸 수 있는 CLI도 제공합니다.

```bash
# 설치
pip install deepagents

# 터미널 에이전트 실행 (Claude Code / Cursor와 유사한 경험)
deepagents

# 비대화형 모드 (CI/CD 파이프라인, 크론 잡 통합용)
deepagents --no-interactive "프로젝트 구조를 분석하고 README.md를 업데이트해줘"
```

어떤 LLM으로도 구동되는 사전 구축된 코딩 에이전트를 터미널에서 사용할 수 있습니다. Claude Code나 Cursor와 유사한 경험을 제공합니다.

---

## 6. Deep Agents가 빛나는 상황

간단한 Q&A나 단일 도구 태스크에는 기본 에이전트로 충분합니다. Deep Agents는 태스크가 질문이 아닌 프로젝트처럼 느껴질 때 진가를 발휘합니다.

| 태스크 성격 | 일반 에이전트 | Deep Agents |
| :--- | :---: | :---: |
| 간단한 Q&A | ✅ 충분 | 과잉 |
| 단일 API 호출 | ✅ 충분 | 과잉 |
| 멀티스텝 리서치 | ⚠️ 불안정 | ✅ 적합 |
| 긴 문서 생성·편집 | ❌ 컨텍스트 초과 | ✅ 파일 오프로드 |
| 코드 생성 + 테스트 + 문서화 | ❌ 역할 혼재 | ✅ 서브에이전트 분리 |
| 세션 간 정보 기억 | ❌ 불가 | ✅ 장기 메모리 |
| 주기적 자동화 작업 | ❌ 불가 | ✅ 비대화형 CLI |

### 실제 활용 사례

LangChain은 내부적으로 이메일 어시스턴트에 Deep Agents를 사용해 일정 규칙과 사용자 습관을 기억합니다. 또한 자동화 테스트 실행, 릴리즈 노트 생성, 배포 워크플로우 관리에도 사용됩니다. 영업팀에서는 딥 에이전트가 소스를 모니터링하고, 잠재 고객 프로필을 컴파일하며, 업계 뉴스를 요약합니다. 모두 인간 개입 없이 스케줄 기반으로 실행됩니다.

2026년 3월 LangChain은 NVIDIA와 전략적 파트너십을 발표했습니다. AI-Q Blueprint는 NVIDIA의 병렬 실행 도구와 LangGraph 런타임을 결합해 Deep Agents 기반의 프로덕션 엔터프라이즈 리서치 시스템으로 만들어졌습니다.

---

## 7. MCP 연동 — 외부 서비스와 연결

MCP(Model Context Protocol)는 `langchain-mcp-adapters`를 통해 지원됩니다. MCP를 통해 Deep Agents는 수백 개의 외부 서비스(GitHub, Slack, Notion, 데이터베이스 등)와 연결할 수 있습니다.

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent

# MCP 서버 연결
async with MultiServerMCPClient({
    "github": {
        "url": "https://mcp.github.com",
        "transport": "streamable_http",
    },
    "notion": {
        "url": "https://mcp.notion.so",
        "transport": "streamable_http",
    }
}) as client:
    tools = await client.get_tools()

    agent = create_deep_agent(
        model="openai:gpt-4o",
        tools=tools,  # MCP 도구가 자동으로 포함됨
        system_prompt="GitHub과 Notion을 활용해 작업합니다.",
    )

    result = agent.invoke({
        "messages": [{"role": "user",
                      "content": "이번 주 PR을 정리해서 Notion 페이지에 저장해줘"}]
    })
```

---

## 8. 한계와 주의사항

모든 도구가 그렇듯 Deep Agents도 상황에 따라 한계가 있습니다.

**모델 의존성:** 에이전트 성능은 여전히 기반 LLM에 종속됩니다. 약한 모델은 덜 신뢰할 수 있는 에이전트를 만듭니다.

**토큰 비용:** 자율 에이전트는 많은 토큰을 소비합니다. 계획, 여러 번의 도구 호출, 서브에이전트 사용이 더해지면 비용이 빠르게 올라갈 수 있습니다.

**학습 곡선:** 단순한 LangGraph보다는 쉽지만, LangChain 생태계에 대한 기본 이해가 필요합니다. 노코드 도구는 아닙니다.

**상대적 성숙도:** 프로젝트는 2025년 7월에 출시됐습니다. v0.4가 상당한 개선을 가져왔지만 생태계는 아직 성장 중입니다.

---

## 9. 일반 에이전트 vs Deep Agents — 언제 무엇을 쓸까?

```
내 태스크가 한 번의 대화로 끝나는가?
  └── YES → 일반 LLM 호출 또는 기본 에이전트로 충분

여러 단계를 거치는 복잡한 작업인가?
  └── YES →
        파일을 생성하거나 대량 데이터를 다루는가?
          └── YES → Deep Agents (파일시스템 컨텍스트 관리)
        
        역할이 다른 여러 전문가가 필요한가?
          └── YES → Deep Agents (서브에이전트 위임)
        
        세션 간 기억이 필요한가?
          └── YES → Deep Agents (장기 메모리 + Memory Store)
        
        주기적으로 자동 실행이 필요한가?
          └── YES → Deep Agents CLI (비대화형 모드)

워크플로우를 세밀하게 제어해야 하는가?
  └── YES → 직접 LangGraph 그래프 구현
```

---

## 10. 정리

| 개념 | 설명 |
| :--- | :--- |
| **Deep Agents** | 계획·파일시스템·서브에이전트가 내장된 "배터리 포함" 에이전트 프레임워크 |
| **write_todos** | 작업 전 계획을 강제하는 내장 도구 (no-op이지만 강력한 효과) |
| **파일시스템 도구** | 컨텍스트 윈도우 초과를 방지하는 스토리지 오프로드 메커니즘 |
| **서브에이전트** | 전문화된 에이전트에게 서브태스크를 위임해 컨텍스트를 격리 |
| **LangGraph 런타임** | 스트리밍, 상태 관리, Human-in-the-loop를 지원하는 실행 엔진 |
| **장기 메모리** | 세션이 바뀌어도 정보를 유지하는 크로스 스레드 메모리 |
| **CLI** | Claude Code처럼 터미널에서 바로 쓸 수 있는 코딩 에이전트 |

Deep Agents가 말하는 핵심 인사이트는 단순합니다. **에이전트의 성패는 모델이 아니라 컨텍스트 설계에 달려 있다.** 올바른 정보를 올바른 시점에 에이전트에게 제공하는 구조, 즉 계획 → 실행 → 오프로드 → 위임의 사이클이 "얕은 에이전트"와 "깊은 에이전트"의 차이를 만듭니다.

> {: .prompt-info }
> **참고 자료**
> - 공식 문서: [https://docs.langchain.com/oss/python/deepagents/overview](https://docs.langchain.com/oss/python/deepagents/overview)
> - GitHub: [https://github.com/langchain-ai/deepagents](https://github.com/langchain-ai/deepagents)
> - PyPI: [https://pypi.org/project/deepagents](https://pypi.org/project/deepagents)
> - Harrison Chase 블로그: [On Agent Frameworks and Agent Observability](https://www.langchain.com/blog/on-agent-frameworks-and-agent-observability)
