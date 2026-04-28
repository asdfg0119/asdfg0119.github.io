---
layout: post
title: "AI Agent의 핵심: Function Calling, LangChain, LangGraph에서 LLM에게 전달되는 것과 LLM이 판단하는 것"
date: 2026-04-28 12:00:00 +0900
categories:
  - LLM
  - AgentMethods
tags: [AIAgent, ToolCall, LangChain, LangGraph, LLM]
math: false
---

> 주제: AI Agent의 동작 원리 — LLM에게 무엇이 전달되고, LLM은 무엇을 판단하는가?
> 대상: LLM/에이전트 개발에 관심 있는 개발자

---

## 1. 도입: 왜 이 주제가 중요한가

### 1.1 LLM의 진화: 텍스트 생성기에서 에이전트로

초기 LLM은 입력 텍스트에 대해 다음 토큰을 예측하는 단순한 텍스트 생성기였습니다. 그러나 **Function Calling**이 도입되면서 LLM은 단순히 답을 생성하는 것을 넘어, 외부 시스템과 상호작용하며 "행동하는 에이전트"로 진화했습니다.

오늘날 우리가 부르는 "AI Agent"는 본질적으로 다음 루프를 돕니다.

```
[사용자 입력] → [LLM이 판단] → [도구 호출 또는 응답] → [결과 관찰] → [다시 LLM 판단] → ...
```

이 루프의 모든 단계에서 **LLM은 무언가를 결정**하고, 그 결정을 기반으로 외부 코드(프레임워크)가 실제 행동을 수행합니다.

### 1.2 핵심 질문

본 세미나에서 답하고자 하는 질문은 다음 세 가지입니다.

1. **LLM에게는 정확히 무엇이 입력으로 전달되는가?**
2. **LLM은 그 입력을 보고 무엇을 판단하는가?**
3. **LangChain과 LangGraph는 그 판단을 어떻게 활용하는가?**

이 질문에 답할 수 있어야 비로소 에이전트의 동작을 디버깅하고, 프롬프트와 도구 설계를 의도대로 할 수 있게 됩니다.

### 1.3 흔한 오해

많은 개발자가 LangChain이나 LangGraph를 사용하면서 다음과 같은 오해를 합니다.

- "LLM이 함수를 직접 실행한다" → **틀렸습니다.** LLM은 절대 코드를 실행하지 않습니다.
- "LangChain의 Agent는 마법처럼 동작한다" → **틀렸습니다.** 결국 프롬프트 조립과 파싱일 뿐입니다.
- "에이전트가 똑똑한 건 LLM이 똑똑하기 때문이다" → **반쪽만 맞습니다.** 도구 description, 프롬프트 구조, 상태 관리 설계가 동등하게 중요합니다.

이 세미나를 통해 이 오해들을 깨끗이 정리합니다.

---

## 2. Function Calling의 본질

### 2.1 Function Calling이란 무엇인가

Function Calling(또는 Tool Use)은 LLM이 미리 정의된 함수 목록 중에서 **어떤 함수를, 어떤 인자로 호출해야 하는지를 구조화된 형식(주로 JSON)으로 출력하는 기능**입니다.

여기서 가장 중요한 사실은 다음과 같습니다.

> **LLM은 함수를 실행하지 않습니다. LLM은 "이 함수를 이런 인자로 호출하면 좋겠다"는 의도를 출력할 뿐입니다.**
>
> **실제 실행은 개발자(또는 프레임워크)의 코드가 합니다.**

### 2.2 LLM에게 실제로 전달되는 것 (Raw API 레벨)

OpenAI 또는 Anthropic API를 직접 호출할 때, LLM에게 전달되는 페이로드를 살펴봅시다.

#### Anthropic API 예시

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "특정 도시의 현재 날씨를 조회합니다. 사용자가 날씨, 기온, 강수 확률 등을 물어볼 때 사용하세요.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "도시 이름 (예: 서울, 부산)"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "온도 단위"
                    }
                },
                "required": ["city"]
            }
        }
    ],
    messages=[
        {"role": "user", "content": "오늘 서울 날씨 어때?"}
    ]
)
```

#### LLM이 보는 것의 정체

위 코드에서 LLM이 실제로 보는 것은 다음 세 가지입니다.

1. **시스템/사용자 메시지**: `"오늘 서울 날씨 어때?"`
2. **사용 가능한 도구 목록**: `get_weather` 함수의 이름, description, parameter schema
3. **이전 대화 이력**: (있다면) 누적된 messages

LLM은 이 정보만으로 다음을 판단합니다.

- 사용자가 무엇을 원하는가?
- 그 요구를 충족시키려면 어떤 도구가 필요한가?
- 그 도구를 호출하려면 어떤 인자를 채워야 하는가?

#### LLM이 반환하는 것

LLM은 다음과 같은 구조화된 응답을 반환합니다.

```json
{
  "id": "msg_01ABC...",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01XYZ...",
      "name": "get_weather",
      "input": {
        "city": "서울",
        "unit": "celsius"
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

여기서 핵심은 **`stop_reason: "tool_use"`** 입니다. LLM은 "나는 답변을 끝낸 게 아니라, 이 도구를 호출해줘"라는 신호를 보낸 것이고, 실행 책임은 호출자에게 넘어갑니다.

### 2.3 다음 턴: 도구 결과를 LLM에게 다시 전달

개발자가 실제로 `get_weather("서울", "celsius")`를 실행해서 결과를 받았다면, 그 결과를 다음 턴에 LLM에게 다시 전달해야 합니다.

```python
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=[...],  # 동일한 tools 정의
    messages=[
        {"role": "user", "content": "오늘 서울 날씨 어때?"},
        {
            "role": "assistant",
            "content": [
                {
                    "type": "tool_use",
                    "id": "toolu_01XYZ...",
                    "name": "get_weather",
                    "input": {"city": "서울", "unit": "celsius"}
                }
            ]
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "tool_result",
                    "tool_use_id": "toolu_01XYZ...",
                    "content": "서울 현재 기온 18도, 맑음, 강수확률 10%"
                }
            ]
        }
    ]
)
```

이 시점에 LLM이 보는 것은 **누적된 전체 대화**입니다. 사용자 질문 → 자신의 도구 호출 → 도구 결과까지가 모두 컨텍스트에 포함되며, LLM은 이를 바탕으로 최종 자연어 응답을 생성하거나 추가 도구 호출을 결정합니다.

### 2.4 Function Calling에서 LLM이 판단하는 것

정리하면, Function Calling에서 LLM이 판단하는 것은 단 네 가지입니다.

| 판단 항목 | 설명 |
|---|---|
| **도구를 호출할지 여부** | 사용자 요청이 일반 답변으로 충분한가, 외부 도구가 필요한가? |
| **어떤 도구를 호출할지** | 여러 도구 중 가장 적합한 것은 무엇인가? |
| **어떤 인자로 호출할지** | 사용자 입력에서 어떤 값을 추출해 어느 파라미터에 넣을지? |
| **도구 결과를 본 후의 다음 행동** | 추가 도구를 호출할지, 최종 답변을 할지? |

**그 외의 모든 것** — 함수의 실제 실행, 에러 처리, 재시도, 결과 파싱, 루프 종료 조건 — 은 전부 개발자나 프레임워크의 코드가 담당합니다.

### 2.5 도구 description의 중요성

LLM이 도구 선택 판단을 내릴 때 가장 크게 의존하는 것은 **함수 이름과 description**입니다. parameter schema도 중요하지만, "이 도구를 언제 써야 하는가"는 description에서 얻습니다.

#### 나쁜 예
```python
{
    "name": "func1",
    "description": "데이터를 가져옵니다.",
    "input_schema": {...}
}
```

#### 좋은 예
```python
{
    "name": "get_user_orders",
    "description": "특정 사용자의 최근 주문 내역을 조회합니다. 사용자가 '내 주문', '최근 구매', '주문 내역' 등을 물어볼 때 사용하세요. 주문 ID, 상품명, 주문일, 금액, 배송 상태를 반환합니다.",
    "input_schema": {...}
}
```

좋은 description의 조건:
- **언제 사용해야 하는지** 명시
- **무엇을 반환하는지** 명시
- **모호한 케이스의 처리 지침** 포함 (예: "사용자가 ID를 지정하지 않으면 현재 로그인된 사용자로 간주합니다")

이것이 에이전트 성능을 좌우하는 가장 큰 요소 중 하나이며, 프롬프트 엔지니어링의 일부로 다뤄야 합니다.

---

## 3. LangChain에서 LLM에게 전달되는 것

### 3.1 LangChain이 하는 일

LangChain은 본질적으로 **프롬프트 조립기 + 도구 실행기 + 루프 컨트롤러**입니다. 마법이 아니라, raw API로 직접 짜야 할 코드를 추상화한 라이브러리입니다.

LangChain Agent의 동작은 다음 흐름을 따릅니다.

```
1. 사용자 입력 수신
2. 도구 정의(@tool)를 JSON Schema로 변환
3. 시스템 프롬프트 + 사용자 입력 + 도구 정의를 LLM에 전달
4. LLM의 응답을 파싱
5. tool_call이 있으면 → 해당 함수 실행 → 결과를 messages에 추가 → 3번으로 복귀
6. 최종 답변이면 → 반환
```

### 3.2 @tool 데코레이터: 함수에서 스키마로의 변환

LangChain에서 도구를 정의하는 가장 흔한 방식은 `@tool` 데코레이터입니다.

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str, unit: str = "celsius") -> str:
    """특정 도시의 현재 날씨를 조회합니다.

    Args:
        city: 도시 이름 (예: 서울, 부산)
        unit: 온도 단위 (celsius 또는 fahrenheit)
    """
    # 실제 API 호출 로직
    return f"{city} 현재 기온 18도, 맑음"
```

이 함수가 LLM에 전달될 때는 다음과 같이 변환됩니다.

```json
{
  "name": "get_weather",
  "description": "특정 도시의 현재 날씨를 조회합니다.\n\nArgs:\n    city: 도시 이름 (예: 서울, 부산)\n    unit: 온도 단위 (celsius 또는 fahrenheit)",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {"type": "string"},
      "unit": {"type": "string", "default": "celsius"}
    },
    "required": ["city"]
  }
}
```

여기서 알 수 있는 사실:
- **함수 이름** → 도구 이름
- **docstring** → description
- **타입 힌트** → JSON Schema의 type
- **기본값** → default

따라서 LangChain에서 **docstring과 타입 힌트는 단순한 문서가 아니라, LLM에게 직접 전달되는 프롬프트의 일부**입니다.

### 3.3 create_tool_calling_agent의 내부

LangChain의 표준 에이전트 생성 함수는 다음과 같이 사용됩니다.

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain.agents import create_tool_calling_agent, AgentExecutor

llm = ChatAnthropic(model="claude-opus-4-7")

prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 친절한 어시스턴트입니다. 필요하면 도구를 사용하세요."),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, [get_weather], prompt)
executor = AgentExecutor(agent=agent, tools=[get_weather], verbose=True)

result = executor.invoke({"input": "서울 날씨 알려줘"})
```

#### `agent_scratchpad`의 정체

`{agent_scratchpad}`는 LangChain Agent의 핵심 메커니즘입니다. 이것은 **이전 도구 호출과 그 결과를 누적해서 LLM에게 다시 보여주는 자리**입니다.

첫 번째 LLM 호출 시:
```
system: 당신은 친절한 어시스턴트입니다...
human: 서울 날씨 알려줘
[agent_scratchpad는 비어있음]
```

LLM이 `get_weather(city="서울")` 호출을 결정 → LangChain이 실제로 함수 실행 → 결과 받음 → 두 번째 LLM 호출 시:

```
system: 당신은 친절한 어시스턴트입니다...
human: 서울 날씨 알려줘
[agent_scratchpad에 누적]:
  - AIMessage (tool_use: get_weather, {"city": "서울"})
  - ToolMessage (content: "서울 현재 기온 18도, 맑음")
```

LLM은 이를 보고 "도구 결과를 받았으니 이제 사용자에게 자연어로 답하자"고 판단하여 최종 답변을 생성합니다.

### 3.4 verbose=True로 실제 흐름 관찰하기

`AgentExecutor(verbose=True)`로 실행하면 다음과 같은 로그를 볼 수 있습니다.

```
> Entering new AgentExecutor chain...

Invoking: `get_weather` with `{'city': '서울'}`

서울 현재 기온 18도, 맑음

오늘 서울은 기온 18도에 맑은 날씨입니다. 외출하기 좋은 날이네요!

> Finished chain.
```

또한 LangSmith 트레이싱을 켜면 **각 LLM 호출에 정확히 어떤 메시지가 전달되었는지**를 시각적으로 확인할 수 있습니다. 이것이 LangChain 디버깅의 가장 강력한 도구입니다.

### 3.5 ReAct 에이전트와 Tool Calling 에이전트의 차이

LangChain에는 역사적으로 두 가지 에이전트 패턴이 있습니다.

#### ReAct (Reasoning + Acting) 패턴

LLM이 Function Calling을 지원하지 않던 시절의 방식으로, 프롬프트에 다음과 같은 형식을 강제했습니다.

```
Thought: 사용자가 날씨를 물어봤으니 get_weather 도구를 써야겠다.
Action: get_weather
Action Input: {"city": "서울"}
Observation: 서울 현재 기온 18도, 맑음
Thought: 답변할 수 있는 정보를 얻었다.
Final Answer: 오늘 서울은 기온 18도에 맑은 날씨입니다.
```

LLM은 자연어로 위 형식을 출력하고, LangChain이 이를 **정규식으로 파싱**하여 도구를 실행했습니다. 파싱 에러가 빈번했고, 토큰 낭비도 컸습니다.

#### Tool Calling 패턴 (현재 표준)

요즘 LLM은 모두 native function calling을 지원하므로, 구조화된 JSON으로 도구 호출을 받습니다. 파싱 에러가 거의 없고 안정적입니다.

새 프로젝트에서는 `create_tool_calling_agent`를 사용하는 것이 권장됩니다.

### 3.6 LangChain에서 LLM에게 전달되는 것 — 정리

| 시점 | LLM에게 전달되는 것 |
|---|---|
| **첫 호출** | system prompt + 사용자 입력 + 도구 정의(JSON Schema) |
| **N번째 호출** | system prompt + 사용자 입력 + 도구 정의 + 누적된 (AIMessage tool_use, ToolMessage) 페어들 |
| **최종** | 위 모든 것 + LLM이 자연어 답변 생성 |

LangChain은 이 누적과 루프를 자동화해줄 뿐, **LLM에게 전달되는 본질은 raw API와 동일**합니다.

---

## 4. LangGraph에서 LLM에게 전달되는 것

### 4.1 LangGraph는 무엇이 다른가

LangChain의 `AgentExecutor`는 내부적으로 고정된 루프(`while llm_wants_more_tools: run_tool()`)를 가지고 있습니다. 이는 단순한 에이전트에는 충분하지만, 다음과 같은 요구사항이 생기면 한계에 부딪힙니다.

- 조건에 따라 다른 노드(에이전트)로 분기하고 싶다
- 사람의 승인을 중간에 받고 싶다 (Human-in-the-loop)
- 여러 에이전트가 협업하는 복잡한 흐름을 짜고 싶다
- 상태(state)를 명시적으로 관리하고 싶다

**LangGraph**는 이를 위해 등장한 **상태 그래프(State Graph) 기반 프레임워크**입니다. 에이전트의 흐름을 노드(Node)와 엣지(Edge)로 모델링합니다.

### 4.2 핵심 개념: State, Node, Edge

#### State (상태)

LangGraph에서 가장 중요한 개념입니다. 그래프 전체에서 공유되는 데이터 구조이며, 보통 다음과 같이 정의합니다.

```python
from typing import TypedDict, Annotated
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
```

`add_messages`는 reducer 함수로, 새 메시지가 들어오면 기존 리스트에 **append**합니다. 이것이 LangGraph의 메시지 누적 메커니즘입니다.

#### Node (노드)

State를 입력받아 State의 일부를 업데이트해서 반환하는 함수입니다.

```python
def call_llm(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}
```

#### Edge (엣지)

노드 간의 연결을 정의합니다. 일반 엣지는 무조건적이고, **Conditional Edge**는 함수의 반환값에 따라 다음 노드를 결정합니다.

### 4.3 표준 ReAct 그래프 만들기

LangGraph로 가장 기본적인 도구 사용 에이전트를 구성해봅시다.

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-opus-4-7")
tools = [get_weather]
llm_with_tools = llm.bind_tools(tools)

# 노드 정의
def agent_node(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

tool_node = ToolNode(tools)

# 분기 함수
def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return END

# 그래프 구성
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "agent")

app = graph.compile()
```

### 4.4 LangGraph에서 LLM에게 전달되는 것

각 노드에서 LLM이 호출될 때 무엇이 전달되는지 단계별로 추적해봅시다.

#### 단계 1: 사용자 입력

```python
result = app.invoke({"messages": [("user", "서울 날씨 알려줘")]})
```

이 시점의 State:
```python
{
    "messages": [
        HumanMessage(content="서울 날씨 알려줘")
    ]
}
```

#### 단계 2: agent_node에서 첫 LLM 호출

`llm_with_tools.invoke(state["messages"])`가 실행되며, LLM에게 전달되는 것:

```
messages: [HumanMessage("서울 날씨 알려줘")]
tools: [get_weather schema]
```

LLM 응답:
```python
AIMessage(
    content="",
    tool_calls=[{"name": "get_weather", "args": {"city": "서울"}, "id": "..."}]
)
```

State 업데이트 후:
```python
{
    "messages": [
        HumanMessage("서울 날씨 알려줘"),
        AIMessage(tool_calls=[{...}])
    ]
}
```

#### 단계 3: Conditional Edge 평가

`should_continue` 함수가 마지막 메시지를 확인하여 `tool_calls`가 있으므로 `"tools"` 반환 → ToolNode로 이동.

#### 단계 4: ToolNode가 도구 실행

ToolNode는 마지막 AIMessage의 tool_calls를 보고 실제로 `get_weather(city="서울")`를 호출합니다. 결과를 `ToolMessage`로 감싸 State에 추가합니다.

State 업데이트:
```python
{
    "messages": [
        HumanMessage("서울 날씨 알려줘"),
        AIMessage(tool_calls=[{...}]),
        ToolMessage(content="서울 현재 기온 18도, 맑음", tool_call_id="...")
    ]
}
```

#### 단계 5: 다시 agent_node로 복귀

엣지 `("tools" → "agent")`에 따라 다시 LLM이 호출됩니다. 이때 LLM에게 전달되는 것:

```
messages: [
    HumanMessage("서울 날씨 알려줘"),
    AIMessage(tool_calls=[...]),
    ToolMessage("서울 현재 기온 18도, 맑음")
]
tools: [get_weather schema]
```

LLM은 도구 결과를 보고 더 이상 도구를 호출할 필요가 없다고 판단 → 자연어 답변 생성.

#### 단계 6: 종료

`should_continue`가 `tool_calls`가 없는 것을 보고 `END` 반환 → 그래프 종료.

### 4.5 LangGraph만의 특징

#### 4.5.1 LLM의 판단이 곧 라우팅 결정

LangChain의 AgentExecutor에서는 LLM의 판단이 "다음 도구 호출"로만 쓰였습니다. LangGraph에서는 LLM의 판단이 **그래프의 다음 노드를 결정**합니다.

예를 들어, 라우터 노드를 만들어 사용자 의도를 분류한 뒤 서로 다른 전문 에이전트로 보낼 수 있습니다.

```python
def router_node(state: AgentState) -> dict:
    response = llm.invoke([
        SystemMessage("사용자 의도를 분류하세요: 'weather', 'news', 'general'"),
        state["messages"][-1]
    ])
    return {"intent": response.content}

def route_by_intent(state: AgentState) -> str:
    return state["intent"]

graph.add_conditional_edges("router", route_by_intent, {
    "weather": "weather_agent",
    "news": "news_agent",
    "general": "general_agent",
})
```

#### 4.5.2 명시적 상태 관리

State는 messages 외에도 임의의 필드를 가질 수 있습니다.

```python
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    user_id: str
    retry_count: int
    intermediate_results: dict
```

이를 통해 멀티 에이전트 간 정보 공유, 메모리 관리, 컨텍스트 유지가 깔끔해집니다.

#### 4.5.3 Human-in-the-Loop

특정 노드 앞에서 그래프를 일시 중단하고 사람의 승인을 기다릴 수 있습니다.

```python
app = graph.compile(checkpointer=memory, interrupt_before=["tools"])
```

도구 실행 직전에 멈추고, 사람이 검토 후 재개하는 패턴을 깔끔하게 구현할 수 있습니다.

#### 4.5.4 사이클(Cycle) 지원

LangGraph는 그래프에 사이클을 명시적으로 허용합니다. 이는 LangChain의 고정 루프와 달리, 임의의 반복 패턴(자기 비판, 재시도, 합의 등)을 자유롭게 구성할 수 있게 해줍니다.

### 4.6 LangGraph에서 LLM에게 전달되는 것 — 정리

| 측면 | 내용 |
|---|---|
| **컨텍스트** | State의 `messages` 필드 전체가 매 LLM 호출마다 전달됨 |
| **누적 방식** | `add_messages` reducer가 각 노드에서 추가된 메시지를 자동 append |
| **도구 결과** | `ToolNode`가 도구를 실행해 `ToolMessage`로 감싸 State에 주입 |
| **LLM의 판단의 역할** | tool_calls 유무가 다음 노드(라우팅)를 결정 |

핵심 차이는 **"LLM의 판단이 단순히 도구 호출이 아니라, 그래프의 흐름 자체를 결정한다"**는 점입니다.

---

## 5. 세 가지를 관통하는 핵심: LLM은 무엇을 판단하는가?

Raw Function Calling, LangChain, LangGraph 세 가지를 모두 살펴봤습니다. 결론적으로 LLM이 판단하는 것은 본질적으로 동일합니다.

### 5.1 LLM이 실제로 판단하는 네 가지

| # | 판단 항목 | 입력 정보 | 출력 형식 |
|---|---|---|---|
| 1 | **도구 호출 여부** | 사용자 메시지 + 도구 목록 | `stop_reason: "tool_use"` 또는 `"end_turn"` |
| 2 | **어떤 도구를 호출할지** | 도구 description + 사용자 의도 | `tool_use.name` |
| 3 | **어떤 인자로 호출할지** | parameter schema + 메시지 내용 | `tool_use.input` (JSON) |
| 4 | **결과를 본 후의 다음 행동** | 누적 messages + 도구 결과 | 추가 tool_use 또는 자연어 응답 |

이 네 가지가 LLM이 하는 모든 "결정"입니다.

### 5.2 LLM이 판단하지 않는 것

반대로, 다음은 LLM이 절대 하지 않습니다.

- **함수의 실제 실행** — 코드 인터프리터, 외부 API 호출 등은 모두 호스트 코드(프레임워크)가 함
- **에러 핸들링** — 도구가 예외를 던지면 그 처리는 프레임워크가 하고, LLM에게는 결과(또는 에러 메시지)만 전달됨
- **루프 종료 조건** — 무한 루프 방지, 최대 반복 횟수 등은 프레임워크 설정
- **상태 저장 및 복원** — 메모리, 체크포인터 등도 프레임워크 책임
- **권한 검사 및 보안** — 위험한 도구 실행 차단, 인증 등도 호스트 책임

### 5.3 LLM이 판단을 잘하게 만드는 요소

LLM의 판단 품질을 결정하는 것은 다음 세 가지입니다.

#### 1) 도구 description의 품질
- 언제 사용해야 하는지 명확히 명시
- 무엇을 반환하는지 명시
- 모호한 케이스 처리 지침 포함

#### 2) Parameter Schema의 명료성
- 적절한 type 지정 (enum 활용)
- description으로 각 파라미터의 의미 설명
- required 필드 정확히 지정

#### 3) 시스템 프롬프트와 컨텍스트
- 에이전트의 역할과 규칙 명시
- 도구 사용 정책 (언제 사용/미사용)
- 누적된 messages가 너무 길면 LLM의 판단력 저하 → 컨텍스트 관리 필요

### 5.4 프레임워크 선택의 기준

| 상황 | 권장 |
|---|---|
| 단일 에이전트, 단순한 도구 사용 | Raw API 또는 LangChain |
| 빠른 프로토타이핑, 표준 ReAct 패턴 | LangChain |
| 복잡한 분기, 멀티 에이전트, HITL | **LangGraph** |
| 최대한의 제어와 투명성 | Raw API |

LangChain은 진입 장벽이 낮고, LangGraph는 복잡한 워크플로우에 적합합니다. 모든 경우에 대해 raw API로 직접 짜는 것도 충분히 합리적인 선택입니다 — 추상화 비용이 없고 디버깅이 가장 쉽기 때문입니다.

---

## 6. 실전 비교 데모

같은 작업을 세 가지 방식으로 구현해서 LLM에게 전달되는 프롬프트를 비교해봅시다.

**작업**: "오늘 서울 날씨 알려주고 우산 챙길지 조언해줘"

### 6.1 Raw API 버전

```python
import anthropic

client = anthropic.Anthropic()

def get_weather(city: str) -> dict:
    return {"city": city, "temp": 18, "condition": "흐림", "rain_prob": 70}

tools = [{
    "name": "get_weather",
    "description": "도시의 현재 날씨, 기온, 강수확률을 조회합니다.",
    "input_schema": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
    }
}]

messages = [{"role": "user", "content": "오늘 서울 날씨 알려주고 우산 챙길지 조언해줘"}]

while True:
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    if response.stop_reason == "end_turn":
        print(response.content[0].text)
        break

    # tool_use 처리
    for block in response.content:
        if block.type == "tool_use":
            result = get_weather(**block.input)
            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result)
                }]
            })
```

**LLM에게 전달되는 것**: 우리가 직접 messages 리스트를 관리하므로 100% 투명합니다.

### 6.2 LangChain 버전

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langchain_core.prompts import ChatPromptTemplate
from langchain.agents import create_tool_calling_agent, AgentExecutor

@tool
def get_weather(city: str) -> str:
    """도시의 현재 날씨, 기온, 강수확률을 조회합니다."""
    return f"{city}: 18도, 흐림, 강수확률 70%"

llm = ChatAnthropic(model="claude-opus-4-7")
prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 도움이 되는 어시스턴트입니다."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, [get_weather], prompt)
executor = AgentExecutor(agent=agent, tools=[get_weather], verbose=True)

result = executor.invoke({"input": "오늘 서울 날씨 알려주고 우산 챙길지 조언해줘"})
```

**LLM에게 전달되는 것**: LangSmith로 추적해보면 raw API 버전과 거의 동일한 messages가 보내지는 것을 확인할 수 있습니다. 차이는 단지 LangChain이 자동으로 루프를 돌려준다는 점입니다.

### 6.3 LangGraph 버전

```python
from typing import TypedDict, Annotated
from langchain_core.messages import BaseMessage
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode

class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]

llm_with_tools = ChatAnthropic(model="claude-opus-4-7").bind_tools([get_weather])

def agent(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

def should_continue(state: State):
    return "tools" if state["messages"][-1].tool_calls else END

graph = StateGraph(State)
graph.add_node("agent", agent)
graph.add_node("tools", ToolNode([get_weather]))
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "agent")

app = graph.compile()
result = app.invoke({"messages": [("user", "오늘 서울 날씨 알려주고 우산 챙길지 조언해줘")]})
```

**LLM에게 전달되는 것**: State.messages 전체. 그래프가 사이클을 돌 때마다 messages가 누적되며, 매 LLM 호출마다 전체가 전달됩니다.

### 6.4 세 버전의 비교표

| 항목 | Raw API | LangChain | LangGraph |
|---|---|---|---|
| 코드 양 | 가장 많음 | 적음 | 중간 |
| 투명성 | 최고 | 중간 (LangSmith로 보완 가능) | 중간 |
| 유연성 | 최고 | 표준 패턴에 적합 | 복잡 워크플로우에 최적 |
| 학습 곡선 | 낮음 (API만 알면 됨) | 중간 | 높음 |
| LLM에 전달되는 본질 | 명시적 messages 관리 | scratchpad에 누적 | State.messages에 누적 |

세 방식 모두 LLM이 보는 것은 **결국 같은 messages 시퀀스**입니다. 차이는 그것을 누가 어떻게 관리하느냐일 뿐입니다.

---

## 7. 정리 및 에이전트 설계 시 시사점

### 7.1 핵심 메시지 다섯 가지

1. **LLM은 함수를 실행하지 않는다.** 호출 의도를 출력할 뿐이며, 실행은 호스트 코드의 책임이다.
2. **LLM이 판단하는 것은 단 네 가지**다 — 호출 여부, 어떤 도구, 어떤 인자, 다음 행동.
3. **도구 description은 프롬프트의 일부**다 — 에이전트 성능을 좌우하는 핵심 요소.
4. **LangChain은 프롬프트 조립기 + 루프 컨트롤러**다 — 마법이 아니라 추상화일 뿐.
5. **LangGraph는 LLM의 판단을 그래프 라우팅과 결합**한다 — 복잡한 워크플로우를 위한 도구.

### 7.2 좋은 에이전트를 만들기 위한 체크리스트

- [ ] 각 도구의 description이 "언제 사용해야 하는지"를 명확히 설명하는가?
- [ ] Parameter schema가 type, enum, description을 충분히 활용하는가?
- [ ] 시스템 프롬프트가 에이전트의 역할과 도구 사용 정책을 명시하는가?
- [ ] 누적되는 messages가 컨텍스트 한계를 넘지 않도록 관리하는가?
- [ ] 도구 실행 에러를 LLM이 이해할 수 있는 형태로 전달하는가?
- [ ] 무한 루프 방지를 위한 최대 반복 횟수 제한이 있는가?
- [ ] 위험한 도구(파일 삭제, 결제 등)에 대한 사전 승인(HITL) 메커니즘이 있는가?
- [ ] LangSmith나 자체 로깅으로 LLM에게 전달되는 실제 페이로드를 추적할 수 있는가?

### 7.3 디버깅 시 가장 먼저 볼 것

에이전트가 의도대로 동작하지 않을 때, 가장 먼저 확인해야 할 것은:

> **"LLM에게 정확히 무엇이 전달되었는가?"**

이 한 가지 질문에 답할 수 있다면 대부분의 문제는 해결됩니다. LangSmith, verbose 로그, 또는 직접 messages를 print해서 보세요. LLM의 판단이 이상하다면, 거의 항상 입력에서 무언가가 잘못된 것입니다.

### 7.4 마무리

AI Agent의 본질은 신비롭지 않습니다. **LLM은 잘 정의된 입력을 받아 잘 정의된 판단을 내리는 함수**일 뿐이고, LangChain과 LangGraph는 그 함수를 호출하고 결과를 처리하는 인프라일 뿐입니다.

이 구조를 이해하면:
- 어떤 프레임워크든 빠르게 학습할 수 있고
- 디버깅이 압도적으로 쉬워지며
- 새로운 패턴(멀티 에이전트, HITL, 자기 비판 등)을 자유롭게 설계할 수 있습니다.

> **"프레임워크에 끌려가지 말고, 프레임워크를 부리세요. 그러기 위한 첫걸음은 LLM에게 정확히 무엇이 전달되는지 아는 것입니다."**

---

## 부록: 참고 자료

- Anthropic Tool Use 공식 문서: https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview
- LangChain Agents 가이드: https://python.langchain.com/docs/concepts/agents/
- LangGraph 공식 튜토리얼: https://langchain-ai.github.io/langgraph/
- LangSmith 트레이싱: https://docs.smith.langchain.com/
- ReAct 논문: "ReAct: Synergizing Reasoning and Acting in Language Models" (Yao et al., 2022)

---

