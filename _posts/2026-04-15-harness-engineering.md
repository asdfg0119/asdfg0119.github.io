---
layout: post
title: "Harness Engineering: AI 에이전트를 길들이는 새로운 엔지니어링 패러다임"
date: 2026-04-15 11:00:00 +0900
categories:
  - LLM
  - AgentEngineering
tags: [HarnessEngineering, AIAgent, ContextEngineering, LLM, DevOps, OpenAI]
math: false
---

> {: .prompt-tip }
> **요약:** Harness Engineering은 AI 에이전트를 프로덕션 환경에서 **신뢰할 수 있게 만드는 인프라 설계 규율**입니다. 모델 자체가 아니라 모델을 둘러싼 **제약(Constraints), 피드백 루프(Feedback Loops), 컨텍스트 관리, 검증 시스템** 전체를 설계하는 것이 핵심입니다.


---

2026년 초, OpenAI의 엔지니어링 팀은 놀라운 사실을 공개했습니다. 100만 줄이 넘는 코드로 구성된 내부 프로덕트를 출시했는데, 그 중 **단 한 줄도 인간이 직접 작성하지 않았다는 것**입니다. 엔지니어들의 역할은 코드를 쓰는 것이 아니라, AI 에이전트가 코드를 안정적으로 작성할 수 있는 시스템을 설계하는 것이었습니다. 그 시스템에는 이름이 있습니다. 바로 **Harness(하네스)** 입니다.


---

## 1. Harness Engineering이란 무엇인가?

**Harness Engineering**은 AI 에이전트를 프로덕션 환경에서 신뢰할 수 있게 만드는 인프라와 제약 시스템을 설계하는 새로운 공학 규율(discipline)입니다. 2026년 2월, Mitchell Hashimoto(HashiCorp, Terraform 공동 창업자)가 AI 에이전트와 작업하면서 발전시킨 개념으로, 이후 OpenAI와 Anthropic이 관련 내용을 공개하며 업계 표준 용어로 자리 잡았습니다.

핵심 등식은 단순합니다:

```
Agent = Model + Harness
```

여기서 **Model(모델)**은 AI의 두뇌, 즉 Claude나 GPT 같은 LLM 자체입니다. 그리고 **Harness(하네스)**는 그 모델을 감싸는 모든 것, 즉 모델이 받는 컨텍스트, 접근할 수 있는 도구, 실수를 막는 가드레일, 오류를 잡는 검증 시스템, 그리고 실패를 복구하는 메커니즘 전체를 의미합니다.

컴퓨터 과학적 비유로 설명하면, 모델은 CPU, 컨텍스트 윈도우는 RAM, 그리고 **Harness는 운영체제(OS)**입니다. 마치 CPU 위에 OS 없이 소프트웨어를 실행하지 않듯, AI 에이전트도 하네스 없이는 프로덕션 환경에서 작동하기 어렵습니다.


---

## 2. 왜 지금 Harness Engineering이 필요한가?

AI 에이전트에는 구조적인 약점이 존재합니다. 이를 방치하면 프로덕션 환경에서 심각한 문제를 일으킵니다.

- **메모리 없음**: 모델은 세션 간 기억이 없습니다. 이전 작업에서 무엇을 했는지 완전히 잊은 채로 다음 작업을 시작합니다.
- **자신감 있는 실수**: LLM은 "모른다"고 말하지 않습니다. 그럴듯하지만 틀린 결과를 아무렇지 않게 내놓고, 검증 루프가 없으면 그 오류는 조용히 전파됩니다.
- **무제한 도구 접근의 위험성**: 쉘 접근 권한이 있는 에이전트는 파일을 삭제하거나 데이터베이스를 덮어쓸 수 있습니다. 가드레일 없는 자율성은 곧 리스크입니다.
- **규모가 오류를 증폭시킴**: 에이전트 한 대의 작은 실수는 관리 가능합니다. 하지만 10개의 에이전트가 병렬로 실행되며 각각 작은 실수를 저지르면, 디버깅이 거의 불가능한 연쇄 장애가 발생합니다.

Harness Engineering은 이 모든 문제를 시스템적으로 해결합니다.


---

## 3. 핵심 구성 요소: 5가지 기둥

Harness는 다음 다섯 가지 핵심 레이어로 구성됩니다.

### 3-1. Context Engineering (컨텍스트 관리)
에이전트에게 올바른 정보를 올바른 시점에 제공하는 것입니다. 에이전트의 관점에서, **컨텍스트에 없는 것은 존재하지 않습니다.** Slack 스레드, 구글 독스의 결정, 누군가의 머릿속 지식은 에이전트에게 보이지 않습니다. 따라서 에이전트가 필요로 하는 모든 것은 에이전트가 읽을 수 있는 형태로 **저장소(repository) 안에 존재해야 합니다.**

### 3-2. Tool Orchestration (도구 오케스트레이션)
에이전트가 접근할 수 있는 도구, 호출 방식, 필요한 권한을 명시적으로 정의합니다. 파일 시스템 접근, 쉘 명령어, API 호출, DB 쿼리 등 모든 도구 레이어는 경계가 명확해야 합니다. 에이전트는 자신이 무엇을 할 수 있고, 할 수 없으며, 어떤 행동이 인간의 승인이 필요한지 알아야 합니다.

### 3-3. Guardrails & Validation (가드레일과 검증)
가드레일은 에이전트가 절대 넘어서는 안 되는 결정론적(deterministic) 규칙입니다. 린터(linter), 타입 체커(type checker), 단위 테스트는 잘못된 행동을 물리적으로 차단합니다. 프롬프트와 달리, 이 제약은 비결정론적이지 않습니다. **에이전트가 원하든 원하지 않든 구조적으로 작동합니다.**

### 3-4. State Persistence & Memory (상태 지속성과 메모리)
컨텍스트 윈도우가 초기화되더라도 작업의 진행 상황을 보존합니다. 벡터 데이터베이스를 통한 동적 히스토리 검색, 구조화된 진행 로그, 체크포인트 메커니즘을 통해 에이전트가 "기억 상실" 없이 긴 작업을 수행할 수 있게 합니다.

### 3-5. Observability & Feedback Loops (관측 가능성과 피드백 루프)
에이전트의 행동을 추적하고, 실패 패턴을 클러스터링하며, 수정 내용을 다시 하네스에 주입합니다. Harness Engineering의 핵심 철학 중 하나는 **"에이전트가 실수를 했을 때, 그 출력을 고치는 것이 아니라 그 실수가 다시는 발생하지 않도록 시스템을 고친다"**는 것입니다.


---

## 4. 실전 코드: AGENTS.md와 Python 하네스 구현

### Step 1: AGENTS.md - 에이전트의 '직원 핸드북'

`AGENTS.md`는 저장소 루트에 위치하는 기계 판독 가능 문서로, 에이전트의 세션 시작 시 시스템 프롬프트에 주입됩니다. OpenAI 팀이 실제로 사용한 방식입니다.

```markdown
# AGENTS.md

## Project Context
이 저장소는 결제 처리 마이크로서비스입니다.
주요 언어: Python 3.11, FastAPI, PostgreSQL

## Architecture Rules (MUST FOLLOW)
- 의존성 방향: Types → Config → Repo → Service → Runtime → UI (단방향 강제)
- 외부 서비스 호출은 반드시 `services/` 레이어에서만 수행
- 데이터베이스 직접 접근 금지 (반드시 Repository 패턴 경유)

## Commands
- 테스트 실행: `pytest tests/ -v`
- 린팅: `ruff check . && mypy src/`
- 서버 시작: `uvicorn src.main:app --reload`
- 서비스 기동 확인 기준: 800ms 이내 응답 (필수 조건)

## Guardrails
- `rm -rf` 또는 `DROP TABLE` 명령 절대 사용 금지
- 프로덕션 환경 변수 직접 수정 금지
- 새 구현 시작 전, 반드시 계획(plan)을 먼저 작성하고 승인받을 것
- 모든 진행 상황은 `progress_log.md`에 기록

## Definition of Done
- [ ] 모든 단위 테스트 통과
- [ ] 타입 체크 오류 없음 (mypy)
- [ ] 린트 오류 없음 (ruff)
- [ ] 서비스 800ms 이내 기동 확인
```

---

### Step 2: Python 하네스 - 가드레일과 피드백 루프

```python
import subprocess
import json
from pathlib import Path
from typing import Callable
from anthropic import Anthropic

client = Anthropic()

# ---------------------------------------------------------
# 1. Guardrail: 위험 명령어 차단 (결정론적 제약)
# ---------------------------------------------------------
FORBIDDEN_PATTERNS = ["rm -rf", "DROP TABLE", "DELETE FROM", "> /dev/null 2>&1"]

def check_guardrails(command: str) -> bool:
    """에이전트가 실행하려는 명령이 금지된 패턴을 포함하는지 검사합니다."""
    for pattern in FORBIDDEN_PATTERNS:
        if pattern in command:
            print(f"[GUARDRAIL BLOCKED] 금지된 명령 감지: '{pattern}'")
            return False
    return True

# ---------------------------------------------------------
# 2. Tool: 안전한 쉘 명령 실행기
# ---------------------------------------------------------
def safe_shell(command: str) -> dict:
    """가드레일을 통과한 명령만 실행합니다."""
    if not check_guardrails(command):
        return {"success": False, "error": "가드레일에 의해 차단된 명령입니다."}
    
    try:
        result = subprocess.run(
            command, shell=True, capture_output=True,
            text=True, timeout=30
        )
        return {
            "success": result.returncode == 0,
            "stdout": result.stdout,
            "stderr": result.stderr,
        }
    except subprocess.TimeoutExpired:
        return {"success": False, "error": "명령 실행 시간 초과 (30초)"}

# ---------------------------------------------------------
# 3. State Persistence: 진행 상황 저장 (컨텍스트 윈도우 초기화 대비)
# ---------------------------------------------------------
PROGRESS_LOG = Path("progress_log.md")

def log_progress(task: str, status: str, note: str = ""):
    """에이전트의 작업 진행 상황을 파일에 기록합니다."""
    entry = f"\n- [{status}] {task}"
    if note:
        entry += f"\n  → {note}"
    with open(PROGRESS_LOG, "a", encoding="utf-8") as f:
        f.write(entry)
    print(f"[PROGRESS LOG] {task}: {status}")

def load_context() -> str:
    """AGENTS.md와 진행 로그를 읽어 에이전트 컨텍스트로 제공합니다."""
    agents_md = Path("AGENTS.md").read_text(encoding="utf-8") \
        if Path("AGENTS.md").exists() else "AGENTS.md 없음"
    progress = PROGRESS_LOG.read_text(encoding="utf-8") \
        if PROGRESS_LOG.exists() else "진행 기록 없음"
    return f"## 프로젝트 규칙 (AGENTS.md)\n{agents_md}\n\n## 진행 기록\n{progress}"

# ---------------------------------------------------------
# 4. Feedback Loop: 실수 → 하네스 업데이트
# ---------------------------------------------------------
def update_guardrail(mistake: str, new_rule: str):
    """에이전트가 실수를 저질렀을 때, 출력을 수정하는 대신 규칙을 업데이트합니다."""
    FORBIDDEN_PATTERNS.append(new_rule)
    with open("AGENTS.md", "a", encoding="utf-8") as f:
        f.write(f"\n- 학습된 규칙: {mistake} → 금지 패턴 추가: `{new_rule}`")
    print(f"[HARNESS UPDATED] 새 가드레일 추가: '{new_rule}'")

# ---------------------------------------------------------
# 5. 하네스 런타임: 에이전트 실행
# ---------------------------------------------------------
def run_agent(task: str):
    """
    컨텍스트, 도구, 가드레일이 갖춰진 하네스 위에서 에이전트를 실행합니다.
    """
    tools = [
        {
            "name": "run_shell",
            "description": "가드레일이 적용된 안전한 쉘 명령 실행기",
            "input_schema": {
                "type": "object",
                "properties": {
                    "command": {"type": "string", "description": "실행할 쉘 명령"}
                },
                "required": ["command"]
            }
        }
    ]
    
    system_prompt = f"""당신은 소프트웨어 엔지니어링 에이전트입니다.
    아래의 프로젝트 규칙과 진행 기록을 반드시 따르십시오.
    
    {load_context()}
    
    중요: 구현 시작 전 반드시 계획을 먼저 제시하십시오.
    모든 작업 후 결과를 명확히 보고하십시오.
    """
    
    messages = [{"role": "user", "content": task}]
    log_progress(task, "STARTED")
    
    # 에이전트 루프 (도구 호출 처리)
    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )
        
        if response.stop_reason == "end_turn":
            final_text = next(
                (b.text for b in response.content if hasattr(b, "text")), ""
            )
            print(f"\n[AGENT OUTPUT]\n{final_text}")
            log_progress(task, "COMPLETED")
            break
        
        # 도구 호출 처리
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                cmd = block.input.get("command", "")
                print(f"[TOOL CALL] run_shell: {cmd}")
                result = safe_shell(cmd)
                
                # 피드백 루프: 실패 시 로그 기록
                if not result["success"]:
                    log_progress(cmd, "FAILED", result.get("error", result.get("stderr", "")))
                
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(result, ensure_ascii=False)
                })
        
        # 대화 히스토리 누적 (컨텍스트 유지)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})


# ---------------------------------------------------------
# 실행 예시
# ---------------------------------------------------------
if __name__ == "__main__":
    run_agent(
        "결제 모듈의 단위 테스트를 실행하고, "
        "실패한 테스트가 있다면 원인을 분석한 후 수정 계획을 제시해줘."
    )
```

> **참고:** 위 코드는 Anthropic Claude API와 `anthropic` Python SDK를 활용한 예시입니다. 실제 프로젝트에서는 가드레일 패턴과 도구 목록을 비즈니스 요구사항에 맞게 확장해야 합니다.


---

## 5. 결과: 하네스는 실제로 얼마나 효과적인가?

LangChain의 Deep Agents 팀은 이를 수치로 증명했습니다. 동일한 GPT 모델을 사용하면서 하네스만 개선했을 때, Terminal Bench 2.0 벤치마크 점수가 **52.8%에서 66.5%로 향상**되었고 순위는 Top 30에서 **Top 5**로 올라섰습니다. 모델은 그대로였습니다. 하네스만 바꿨습니다.

| 구분 | 하네스 없음 | 하네스 적용 | 개선 효과 |
| :--- | :--- | :--- | :--- |
| LangChain Terminal Bench 2.0 | 52.8% | 66.5% | Top 30 → Top 5 |
| OpenAI 내부 프로젝트 생산성 | 기준 | 3.5 PR/엔지니어/일 | 팀 확장에도 생산성 유지 |
| 에이전트 워크플로우 신뢰도 | 기준 | 2~5배 향상 | 2026 사례 연구 기준 |

또한 OpenAI의 내부 프로젝트에서는 **엔지니어 3명이 5개월 동안 하루 평균 3.5개의 PR을 머지**했으며, 팀이 성장해도 생산성이 유지되는 확장성을 보여주었습니다.


---

## 6. 마치며: 엔지니어의 역할이 바뀐다

Harness Engineering은 "AI가 코드를 쓰는 시대"가 아닌, **"엔지니어가 AI가 코드를 쓸 수 있는 환경을 설계하는 시대"**를 선언합니다. 

2026년의 기술 모자이크에서 모델은 이미 상향 평준화되었습니다. Claude, GPT, Gemini 모두 표준 벤치마크에서 좁은 범위 내에서 경쟁합니다. 경쟁 우위는 더 이상 어떤 모델을 쓰느냐가 아니라, **그 모델 주변에 어떤 시스템을 구축했느냐**에서 나옵니다.

Prompt Engineering이 AI에게 "무엇을 물어볼지"를 고민했다면, Context Engineering은 "모델에게 무엇을 보여줄지"를 고민했습니다. 그리고 이제 Harness Engineering은 **"AI 에이전트가 작동하는 세계 전체를 어떻게 설계할지"**를 묻습니다.

코드를 직접 짜는 사람에서, 코드를 짜는 시스템을 짜는 사람으로. 이것이 AI 시대의 소프트웨어 엔지니어가 나아가는 방향입니다.


---
