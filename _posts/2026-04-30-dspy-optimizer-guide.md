---
layout: post
title: "DSPy — COPRO·MIPROv2·GEPA 가이드"
date: 2026-04-30 12:00:00 +0900
categories:
  - LLM
  - DSPy
tags: [DSPy, GEPA, MIPROv2, COPRO, PromptOptimization, LLM, VLM, PromptEngineering]
math: false
---

> {: .prompt-tip }
> **요약:** DSPy는 "프롬프트를 사람이 손으로 작성하는" 시대에서 벗어나 LLM을 **프로그래밍하는** 프레임워크입니다. Signature(입출력 선언) → Module(추론 전략) → Optimizer(자동 최적화)의 3단계로 구성되며, 특히 **GEPA**는 단순 점수뿐만 아니라 **자연어 피드백**을 반영해 프롬프트를 진화시키는 차세대 옵티마이저입니다. 이 글에서는 DSPy가 무엇인지, 각 옵티마이저가 어떻게 다른지, 그리고 실제 이미지 분류 예시에 어떻게 적용하는지를 설명합니다.

---

단어 하나를 바꾸면 성능이 올라가다가 다시 내려가고, 어떤 조합이 최선인지 감이 잡히지 않는 상황. 이 막막함을 해결해주는 것이 바로 **DSPy**입니다.

---

## 1. 프롬프트 엔지니어링의 한계

LLM을 활용하는 전통적인 방식은 이렇습니다.

```python
# 기존 방식 — 사람이 직접 프롬프트를 작성
prompt = """
당신은 식물 분류 전문가입니다.
이미지를 보고 A 또는 B로만 답하세요.
...세부 규칙 12가지...
"""
output = llm(prompt + image)
```

이 방식의 핵심 문제는 세 가지입니다.

**문제 1: 성능 개선이 "감"에 의존한다**
어떤 문장을 추가하면 성능이 올라갈지, 어떤 표현이 더 정확한지 사람이 직관으로 판단합니다. 수십 번 시도해도 천장에 부딪히는 순간이 옵니다.

**문제 2: 모델을 바꾸면 처음부터 다시 해야 한다**
GPT-4를 Claude로 바꾸거나, 대형 모델을 소형 모델로 교체하면 공들여 만든 프롬프트가 전혀 다른 결과를 냅니다. 모델마다 "잘 통하는" 표현 방식이 다르기 때문입니다.

**문제 3: 멀티스텝 파이프라인에서 복잡도가 폭발한다**
검색 → 분석 → 요약처럼 여러 단계로 이루어진 파이프라인에서는 어느 단계의 프롬프트가 문제인지 찾는 것조차 어렵습니다.

DSPy는 이 세 가지 문제를 모두 해결합니다.

---

## 2. DSPy란 무엇인가?

**DSPy(Declarative Self-improving Python)**는 Stanford NLP 연구팀이 개발하고 Databricks가 지원하는 LLM 프로그래밍 프레임워크입니다. 핵심 철학은 한 문장입니다.

> **"Programming, not prompting — LMs."**
> 프롬프트를 작성하는 것이 아니라, LM을 프로그래밍한다.

창시자 Omar Khattab의 표현을 빌리면, DSPy는 어셈블리에서 C 언어로 올라가는 것과 같은 추상화 도약입니다. 개발자가 매번 비트를 직접 조작하지 않듯, DSPy에서는 프롬프트 문자열을 직접 조작하지 않습니다.

### DSPy의 핵심 3요소

DSPy 프로그램은 세 가지 개념으로 구성됩니다.

```
Signature  ───  "무엇을 할 것인가"    (입출력 명세)
    ↓
Module     ───  "어떻게 풀 것인가"    (추론 전략 선택)
    ↓
Optimizer  ───  "어떻게 개선할 것인가" (프롬프트 자동 최적화)
```

비유로 설명하면 이렇습니다.

| DSPy 개념 | 일반 프로그래밍 비유 | 예시 태스크에서 |
| :--- | :--- | :--- |
| **Signature** | 함수 시그니처 (입출력 명세) | `잎 사진 → 분류 결과 (MP 또는 OK)` |
| **Module** | 함수 본문 (로직 구현) | `dspy.Predict(ClassifyLeaf)` |
| **Optimizer** | 컴파일러 (코드를 최적 기계어로) | `COPRO`, `MIPROv2`, `GEPA` |

---

## 3. Signature와 Module — 기초 개념

### Signature: 태스크를 선언한다

Signature는 LLM 호출의 입력과 출력을 **선언**합니다. 프롬프트 문장이 아니라, "이 태스크는 무엇을 받아서 무엇을 반환해야 하는가"를 정의합니다.

```python
from typing import Literal
import dspy

# 방법 1: 인라인 — 빠른 프로토타이핑
qa = dspy.Predict("question -> answer")

# 방법 2: 클래스 기반 — 타입, 설명, 문서화까지
class ClassifyLeaf(dspy.Signature):
    """식물 잎 사진을 보고 잎의 과(科)를 분류한다."""

    image_path: str = dspy.InputField(desc="분류할 잎 이미지 경로")
    label: Literal["MP", "OK"] = dspy.OutputField(
        desc="MP: 단풍과 / OK: 참나무과"
    )
```

> {: .prompt-info }
> **Signature의 docstring이 프롬프트의 지시문이 됩니다.** DSPy는 클래스명, docstring, 필드명과 desc를 바탕으로 실제 프롬프트를 자동 생성합니다. 즉, 프롬프트 대신 구조화된 명세를 씁니다.

### Module: 추론 전략을 선택한다

Module은 Signature를 받아 어떤 방식으로 LLM을 호출할지 결정합니다.

| Module | 설명 | 적합한 태스크 |
| :--- | :--- | :--- |
| `dspy.Predict` | 기본 LLM 호출 | 분류, 단순 Q&A |
| `dspy.ChainOfThought` | 단계별 추론(CoT) 자동 추가 | 논리 추론, 복잡한 판단 |
| `dspy.ReAct` | 도구 호출을 반복하며 추론 | 검색 에이전트 |
| `dspy.Refine` | 출력을 반복적으로 자기 개선 | 고품질 생성 태스크 |

```python
import dspy

dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))

# 기본 예측
predictor = dspy.Predict(ClassifyLeaf)
result = predictor(image_path="leaf_001.png")
print(result.label)  # "MP" 또는 "OK"

# CoT: 단계별 추론 과정도 함께 반환
cot = dspy.ChainOfThought(ClassifyLeaf)
result = cot(image_path="leaf_001.png")
print(result.reasoning)  # 추론 과정
print(result.label)      # 최종 분류
```

---

## 4. Optimizer — 자동으로 더 나은 프롬프트를 찾는다

Optimizer가 DSPy의 핵심입니다. "학습 데이터(질문-정답 쌍)"와 "점수 함수(메트릭)"를 주면, 그 데이터에서 가장 점수를 잘 받는 프롬프트(또는 예시)를 자동으로 찾아줍니다.

```
입력:
  ① 학습 예제 N개 (이미지 + 정답 라벨)
  ② 점수 함수 (예측 == 정답 → 1점)
  ③ 초기 DSPy 프로그램

  ↓ Optimizer.compile()

출력:
  더 나은 instruction + 최적의 Few-shot 예시를 가진
  "컴파일된 프로그램"
```

현재 DSPy에는 여러 종류의 Optimizer가 있습니다. 어떤 것을 선택하느냐가 성능에 직접적인 영향을 줍니다.

---

## 5. 세 가지 주요 Optimizer: COPRO, MIPROv2, GEPA

세 Optimizer의 차이를 이해하는 가장 좋은 방법은 시험 선생님 비유입니다.

| Optimizer | 선생님 비유 |
| :--- | :--- |
| **COPRO** | 시험지에 ○/× 만 적어주고 학생이 알아서 공부하게 함 |
| **MIPROv2** | ○/× + 정답 예시 몇 개 제공 |
| **GEPA** | ○/× + **왜 틀렸는지 한국어 코멘트**까지 적어주고, 그 코멘트를 보고 교과서 자체를 다시 쓰게 함 |

### 5-1. COPRO (Coordinate Prompt Optimization)

**핵심 아이디어:** "instruction 문장 자체를 LLM에게 다시 써보라고 시키자."

```
[현재 instruction]
    ↓
LLM이 N가지 변형 생성
    ↓
각 변형으로 학습셋 평가 → 점수 확인
    ↓
가장 좋은 변형 채택
    ↓
다시 변형 → 평가 → 반복
```

```python
from dspy.teleprompt import COPRO

copro = COPRO(
    metric=accuracy_metric,
    breadth=3,   # 한 번에 생성할 후보 instruction 수
    depth=2,     # 개선 반복 횟수
)
compiled_copro = copro.compile(my_program, trainset=trainset)
```

**장점:** 단순하고 빠르며 구현이 가볍습니다.

**한계:**
- 점수가 0/1(맞다/틀리다)만 전달됩니다. "왜 틀렸는지"는 전혀 반영되지 않습니다.
- 변형이 무작위에 가깝습니다. 핵심 키워드가 변형 과정에서 사라질 수 있습니다.
- 시각적으로 비슷한 두 클래스를 구분하는 태스크처럼 "왜 헷갈리는지"가 중요한 문제에는 약합니다.

### 5-2. MIPROv2 (Multi-prompt Instruction PRoposal Optimizer v2)

**핵심 아이디어:** COPRO + 베이지안 탐색 + Few-shot 예시 최적화를 동시에.

```
instruction 후보 생성
+ few-shot 예시 선택
──────────────────────
베이지안 최적화로
"instruction × few-shot 조합" 공간 탐색
──────────────────────
가장 좋은 조합 채택
```

```python
from dspy.teleprompt import MIPROv2

mipro = MIPROv2(
    metric=accuracy_metric,
    auto="light",     # "light"(~$2, ~20분) | "medium" | "heavy"(~$20+)
)
compiled_mipro = mipro.compile(
    my_program,
    trainset=trainset,
    requires_permission_to_run=False,
)
```

**장점:** COPRO보다 훨씬 체계적이고, instruction과 few-shot을 동시에 최적화합니다.

**한계:**
- 여전히 0/1 점수 기반입니다.
- Few-shot이 핵심인데, **이미지 입력 환경**에서는 few-shot 이미지를 모두 끼워 넣기 어렵습니다(컨텍스트 폭발, OOM 위험). 강점 절반을 못 쓰는 셈입니다.
- 데이터가 많을수록 효과가 커지는 구조입니다.

### 5-3. GEPA (Genetic-Pareto Reflective Prompt Evolution)

**핵심 아이디어:** "점수만 보는 것이 아니라, 자연어 피드백을 반영해 프롬프트를 진화시킨다."

GEPA는 2024~2025년 DSPy에 도입된 차세대 옵티마이저입니다. COPRO/MIPROv2와 근본적으로 다른 점은 **Reflection(반성)** 메커니즘입니다.

```
[학습셋 평가]
    ↓
실패 케이스 수집
"MP가 정답인데 OK로 예측함"
    ↓
메트릭이 자연어 피드백 생성
"WRONG. Expected MP — 잎이 손바닥 모양으로 갈라져 있어야 합니다..."
    ↓
reflection_lm(gpt-4o-mini)이 피드백 분석
"현재 instruction에서 어떤 규칙이 모호해서 모델이 헷갈렸는가?"
    ↓
더 명확한 instruction 재작성
    ↓
Pareto frontier(다양한 우수 후보군) 보존
    ↓
유전 알고리즘으로 세대 진화
    ↓
최종 best instruction 채택
```

```python
import dspy
from dspy.teleprompt import GEPA  # dspy.GEPA로도 접근 가능

# 핵심: 메트릭이 점수뿐만 아니라 자연어 피드백을 반환
def gepa_metric(gold, pred, trace=None, pred_name=None, pred_trace=None):
    score = 1.0 if pred.label == gold.label else 0.0

    if score == 0.0 and gold.label == "MP":
        feedback = (
            "WRONG. Expected MP (단풍과) but predicted OK. "
            "MP의 핵심 특징: 잎이 손바닥 모양으로 여러 갈래(lobe)로 갈라지고, "
            "잎맥이 잎 밑동에서 갈래 끝으로 뻗는 손바닥맥(palmate venation) 구조. "
            "잎 가장자리가 톱니처럼 뾰족한지 다시 확인하세요."
        )
    elif score == 0.0 and gold.label == "OK":
        feedback = (
            "WRONG. Expected OK (참나무과) but predicted MP. "
            "OK의 핵심 특징: 잎이 깃털 모양으로 한 가운데 굵은 주맥에서 "
            "양옆으로 잎살이 뻗는 깃맥(pinnate venation) 구조. "
            "갈래가 둥근 곡선인지, 단풍처럼 손바닥형인지 구별하세요."
        )
    else:
        feedback = "Correct."

    return dspy.Prediction(score=score, feedback=feedback)

# task_lm: 실제 분류 수행 (로컬 모델)
# reflection_lm: instruction을 다시 써주는 "교사" 역할 (텍스트 강력한 모델)
reflection_lm = dspy.LM("openai/gpt-4o-mini", temperature=0.7, max_tokens=2000)

gepa = dspy.GEPA(
    metric=gepa_metric,
    auto="light",                  # 탐색 강도 조절
    reflection_lm=reflection_lm,  # ← GEPA의 핵심 구성 요소
)

compiled_gepa = gepa.compile(
    student=LeafPredictor(),
    trainset=train_examples,
    valset=val_examples,
)
```

**장점:**
- 자연어 피드백 덕분에 "왜 틀렸는지"를 직접 instruction 개선에 반영합니다.
- 데이터가 적어도(50~100건) 효과적입니다. 피드백 한 줄이 0/1 점수보다 정보 밀도가 훨씬 높기 때문입니다.
- 이미지 입력 환경에서도 few-shot 없이 instruction만 개선하므로 안정적입니다.
- 최근 벤치마크에서 COPRO/MIPROv2 대비 5~15%p 추가 향상이 보고되고 있습니다.

**한계:**
- reflection_lm 호출 비용이 추가로 발생합니다 (컴파일 1회당 약 $0.10~0.50).
- 메트릭 작성에 공이 필요합니다. 단순 0/1이 아니라 의미 있는 자연어 피드백을 작성해야 효과가 납니다.

---

## 6. 세 Optimizer 비교

| 항목 | COPRO | MIPROv2 | **GEPA** |
| :--- | :---: | :---: | :---: |
| **최적화 대상** | instruction | instruction + few-shot | instruction |
| **사용 신호** | 0/1 점수 | 0/1 점수 | **점수 + 자연어 피드백** |
| **변형 방식** | LLM 무작위 변형 | 베이지안 + LLM | **Reflection + 유전 + Pareto** |
| **데이터 요구량** | 중간 | 큼 | **작음** |
| **이미지 입력 친화도** | △ | △ | **○** |
| **"왜 틀렸나" 활용** | ✗ | ✗ | **✓** |
| **비용** | 중간 | 큼 | 중간 |
| **적합한 상황** | 빠른 프로토타이핑 | 데이터 충분 + few-shot 활용 | 소량 데이터 + 시각적으로 유사한 클래스 |

---

## 7. 실전 적용: 이미지 이진 분류에 GEPA 쓰기

VLM(Vision-Language Model)으로 시각적으로 비슷한 두 클래스를 구분하는 일반적 시나리오를 가정해 봅시다. 예시 도메인은 **식물 잎 사진을 단풍과(MP)와 참나무과(OK)로 분류**하는 태스크입니다.

### 문제 상황

```
모델: 로컬 4B급 VLM (단일 GPU, ~22GB VRAM)
데이터: 학습 약 55건 (소량)
목표: 잎 사진을 MP / OK 로 이진 분류
수동 프롬프트 한계: 약 77%
DSPy 기본 Predict 적용 시: 약 50% (거의 랜덤)
```

왜 DSPy 기본 적용에서 성능이 50%로 떨어질까요? DSPy의 `Predict`는 Signature의 짧은 docstring을 instruction으로 사용합니다. 기존 수동 프롬프트에 담겨 있던 "잎의 갈래 모양", "잎맥 패턴", "가장자리 톱니" 같은 세부 규칙이 모두 버려진 것이 원인입니다.

### 해결 전략: 수동 프롬프트를 GEPA의 시작점으로

```python
# ❌ 잘못된 접근: 빈 docstring에서 시작 → 50%로 추락
class ClassifyLeafBad(dspy.Signature):
    """MP 또는 OK로 분류한다."""   # 너무 짧음
    label: Literal["MP", "OK"] = dspy.OutputField()

# ✅ 올바른 접근: 수동 프롬프트를 시작점으로 주입
INITIAL_INSTRUCTION = """
당신은 식물 잎 형태를 분류하는 전문가입니다.
주어진 이미지는 잎 사진(단일 잎 또는 가지에 붙은 잎)입니다.
이 잎이 MP(단풍과)인지 OK(참나무과)인지 분류하세요.

## 형상 정의
- MP (단풍과): 잎이 손바닥 모양으로 여러 갈래로 갈라지고, 잎 밑동에서
  갈래 끝으로 뻗는 손바닥맥 구조. 갈래 끝이 비교적 뾰족하다.
- OK (참나무과): 잎이 가운데 굵은 주맥을 따라 깃털 모양으로 펼쳐지며,
  갈래가 둥근 곡선이거나 가장자리 굴곡이 부드럽다.

## 판단 원칙
- 잎의 갈래(lobe) 배치, 잎맥 흐름, 가장자리 형태 세 가지를 동시에 본다.
- 일부만 보이는 잎이라도 잎맥 구조와 갈래 형태로 추론한다.

## 출력 규칙
- 반드시 "MP" 또는 "OK" 두 글자만 출력
- 설명, 이유, 마침표, 한국어 단어 일절 금지
"""

# Signature에 수동 프롬프트를 시작점으로 주입
class _ClassifyLeafBase(dspy.Signature):
    """placeholder"""
    label: Literal["MP", "OK"] = dspy.OutputField(desc="분류 결과")

ClassifyLeaf = _ClassifyLeafBase.with_instructions(INITIAL_INSTRUCTION)
```

> {: .prompt-tip }
> **수동으로 짜던 프롬프트는 버리는 게 아닙니다.** GEPA의 출발점으로 그대로 씁니다. 이래야 50%로 추락하지 않고 77% 위에서 더 끌어올릴 수 있습니다.

### 이미지를 시그니처 밖으로 분리하는 이유

DSPy의 LM 인터페이스는 본래 텍스트 LLM 설계라, 이미지를 Signature 필드에 직접 넣으면 직렬화와 어댑터 호환이 깨집니다. 해결책은 **thread-local**로 이미지를 별도 채널로 흘리는 것입니다.

```python
import threading

_IMAGE_CONTEXT = threading.local()

def set_current_image(path: str):
    """현재 처리할 이미지 경로를 thread-local에 저장."""
    _IMAGE_CONTEXT.path = path

def get_current_image() -> str | None:
    """thread-local에서 현재 이미지 경로를 꺼냄."""
    return getattr(_IMAGE_CONTEXT, "path", None)

class LeafPredictor(dspy.Module):
    def __init__(self):
        super().__init__()
        self.predict = dspy.Predict(ClassifyLeaf)

    def forward(self, image_path: str) -> dspy.Prediction:
        # 이미지를 thread-local에 올려두고 predict 호출
        # 실제 이미지 주입은 커스텀 LM 어댑터에서 처리
        set_current_image(image_path)
        return self.predict()

# 결론: DSPy는 instruction만 최적화하고,
#       이미지는 시그니처 외부 채널로 흐릅니다.
```

### 전체 파이프라인 흐름

```
[학습 데이터 55건]
       ↓ 80:20 분할
  train(44건)          val(11건)
       │                  │
       └──────┬───────────┘
              │
              ▼
  ┌──────────────────────────────────────────┐
  │  GEPA 컴파일                             │
  │                                          │
  │  ① task_lm(로컬 4B VLM)                 │
  │     → 이미지 실제 분류 수행              │
  │                                          │
  │  ② 실패 케이스에 자연어 피드백 생성      │
  │     "OK인데 MP로 분류 — 잎맥이 손바닥형…"│
  │                                          │
  │  ③ reflection_lm(gpt-4o-mini)           │
  │     → 피드백을 읽고 instruction 재작성   │
  │                                          │
  │  ④ 새 instruction으로 재평가             │
  │     → Pareto frontier 갱신 → 세대 진화  │
  └──────────────────────────────────────────┘
              │
              ▼
     compiled_gepa (진화된 instruction 내장)
              │
              ▼
     테스트셋 최종 평가
```

### 비용 구조

| 역할 | 모델 | 비용 |
| :--- | :--- | :--- |
| **task_lm** (실제 분류 수행) | 로컬 4B VLM (GPU) | GPU 시간만 |
| **reflection_lm** (instruction 진화) | gpt-4o-mini | 컴파일 1회당 약 $0.10~0.50 |

reflection_lm은 이미지를 전혀 보지 않습니다. instruction 텍스트와 피드백 텍스트만 다룹니다.

### 실행 순서 및 예상 결과

```python
# 전체 실행 흐름 요약
import dspy
from dspy.teleprompt import GEPA
from dspy.evaluate import Evaluate

# 1. LM 설정
task_lm       = LocalVLMAdapter(model, tokenizer)   # 로컬 VLM 어댑터
reflection_lm = dspy.LM("openai/gpt-4o-mini")
dspy.configure(lm=task_lm)

# 2. 데이터 준비
train_examples = [
    dspy.Example(image_path="img/leaf_001.png", label="MP").with_inputs("image_path"),
    dspy.Example(image_path="img/leaf_002.png", label="OK").with_inputs("image_path"),
    # ... (총 44건)
]
val_examples = [ ... ]  # 11건

# 3. GEPA 컴파일 (~30~90분, auto="light")
gepa = dspy.GEPA(
    metric=gepa_metric,
    auto="light",
    reflection_lm=reflection_lm,
)
compiled_gepa = gepa.compile(
    student=LeafPredictor(),
    trainset=train_examples,
    valset=val_examples,
)

# 4. 테스트셋 평가
evaluator = Evaluate(devset=test_examples, metric=label_metric)
print(evaluator(LeafPredictor()))    # 수동 프롬프트: ~77%
print(evaluator(compiled_gepa))      # GEPA 진화 후: ~82%+ (목표)

# 5. 저장
compiled_gepa.save("leaf_gepa_compiled.json")
# leaf_gepa_best_prompt.md: 사람이 읽을 수 있는 진화된 instruction 텍스트
```

예상 결과:
```
=== 최종 비교 ===
  수동 프롬프트                   : ~77%
  DSPy 기본 Predict (시작점)      : ~73% (val 기준)
  GEPA 진화 후                    : ~82% (목표)
```

### 흔한 에러와 해결

| 증상 | 원인 | 해결 |
| :--- | :--- | :--- |
| `AdapterParseError: ChatAdapter failed to parse` | LM 출력이 ChatAdapter 마커 형식이 아님 | 어댑터에서 `[[ ## label ## ]]\nMP\n\n[[ ## completed ## ]]\n` 형식으로 wrap |
| `RuntimeError: CUDA out of memory` | VRAM 부족 | 4-bit 양자화로 모델 재로드 |
| `openai.AuthenticationError` | API 키 미설정 | 환경변수에 `OPENAI_API_KEY` 등록 |
| 베이스라인이 50%로 떨어짐 | docstring 동적 할당 실패 | `with_instructions(INITIAL_INSTRUCTION)` 패턴 사용 |
| 모든 예측이 한쪽 클래스로 편향 | instruction이 한쪽으로 치우침 | 학습 데이터 클래스 분포 확인 |
| GEPA가 너무 오래 걸림 | `auto` 설정이 큼 | `auto="light"` 유지 |

---

## 8. 최적화 결과 재사용

저장된 모듈을 다른 세션에서 쓰거나, 진화된 instruction을 수동 모드로 재활용하는 것이 GEPA의 가장 큰 실용적 가치입니다.

```python
# 방법 1: 저장된 모듈 직접 로드
predictor = LeafPredictor()
predictor.load("leaf_gepa_compiled.json")

set_current_image("path/to/new_leaf.png")
result = predictor.forward(image_path="path/to/new_leaf.png")
print(result.label)   # "MP" 또는 "OK"

# 방법 2: 진화된 instruction 텍스트를 기존 수동 프롬프트에 복사
# leaf_gepa_best_prompt.md의 내용을 그대로 SYSTEM_PROMPT에 붙여넣기
# → 런타임 재시작 후에도 DSPy 없이 영구 사용 가능
```

---

## 9. 언제 어떤 Optimizer를 선택할까?

```
내 태스크가...

소량 데이터 (<100건)이고 두 클래스가 시각적으로 비슷한가?
  └── YES → GEPA (feedback으로 정보 효율 극대화)
  └── NO  →

이미지/멀티모달 입력이고 few-shot 이미지 주입이 어려운가?
  └── YES → GEPA (instruction만 진화, few-shot 의존 없음)
  └── NO  →

데이터가 충분(200건+)하고 few-shot을 쓸 수 있는가?
  └── YES → MIPROv2 (instruction + few-shot 동시 최적화)
  └── NO  →

빠른 프로토타이핑이 목적인가?
  └── YES → COPRO (가볍고 빠름)
  └── NO  → MIPROv2 또는 GEPA
```

---

## 10. 정리

| 개념 | 역할 | 한 줄 요약 |
| :--- | :--- | :--- |
| **Signature** | 입출력 선언 | "이 태스크는 무엇을 받아서 무엇을 반환하는가" |
| **Module** | 추론 전략 | "어떤 방식으로 LLM을 호출할 것인가" |
| **COPRO** | 옵티마이저 | 점수 기반 무작위 변형, 빠르고 간단 |
| **MIPROv2** | 옵티마이저 | 베이지안 탐색, instruction + few-shot 동시 최적화 |
| **GEPA** | 옵티마이저 | 자연어 피드백 기반 반성적 진화, 소량 데이터에 강함 |

DSPy가 제안하는 패러다임 전환은 명확합니다. 수동으로 프롬프트를 다듬는 것은 어셈블리로 소프트웨어를 짜는 것과 같습니다. 동작은 하지만 유지보수가 어렵고, 시스템이 바뀔 때마다 처음부터 다시 해야 합니다.

특히 GEPA의 핵심 교훈은 이것입니다. "왜 틀렸는가"를 자연어로 정의할 수 있다면, 그 정보는 단순한 0/1 점수보다 훨씬 강력한 학습 신호가 됩니다. 메트릭에 자연어 피드백을 담는 수고가, 수십 번의 수동 프롬프트 튜닝을 대체할 수 있습니다.

> {: .prompt-info }
> **참고 자료**
> - 공식 문서: [https://dspy.ai](https://dspy.ai)
> - GitHub: [https://github.com/stanfordnlp/dspy](https://github.com/stanfordnlp/dspy)
> - 원논문: "DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines" (Khattab et al., 2023)
> - GEPA 논문: "Reflective Prompt Evolution Can Outperform Reinforcement Learning" (2025)
