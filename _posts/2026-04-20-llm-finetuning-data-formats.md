---
layout: post
title: "오픈소스 LLM 파인튜닝 데이터 형식 — Alpaca부터 DPO까지"
date: 2026-04-20 18:00:00 +0900
categories:
  - LLM
  - Fine-tuning
tags: [LLM, FineTuning, Alpaca, ShareGPT, ChatML, DPO, SFT, Dataset]
math: false
---

> {: .prompt-tip }
> **요약:** 오픈소스 LLM을 파인튜닝할 때 학습 데이터의 **형식(format)** 선택은 성능에 직접적인 영향을 줍니다. 이 글에서는 **Alpaca, ShareGPT, ChatML, DPO Preference** 형식의 구조와 특징, 그리고 실전에서 어떤 상황에 무엇을 선택해야 하는지 코드와 함께 정리합니다.

---

Llama 3, Mistral, Qwen 같은 오픈소스 모델을 파인튜닝하려면 가장 먼저 맞닥뜨리는 질문이 있습니다.

> **"학습 데이터를 어떤 형식으로 만들어야 하지?"**

JSON 파일을 열어보면 어떤 데이터셋은 `instruction`, `output` 필드를 쓰고, 어떤 건 `conversations` 리스트를, 또 어떤 건 `messages` 배열을 씁니다. 이 형식들이 왜 다른지, 어떤 상황에 무엇을 써야 하는지 명확히 이해하지 못하면 학습 자체가 잘못된 방향으로 흐를 수 있습니다.

이 글에서는 현재 가장 많이 쓰이는 **4가지 데이터 형식**을 하나씩 해부합니다.

---

## 0. 파인튜닝 데이터 형식이 왜 중요한가?

LLM 파인튜닝은 크게 두 단계로 구성됩니다.

```
1단계: SFT (Supervised Fine-Tuning)
   → 원하는 행동 패턴을 직접 학습
   → Alpaca / ShareGPT / ChatML 형식 사용

2단계: 선호도 최적화 (Preference Optimization)
   → "더 나은 답변"을 선택하도록 정렬(alignment)
   → DPO Preference 형식 사용
```

각 단계에 맞는 형식을 선택해야 합니다. 형식이 틀리면 학습 라이브러리(TRL, Axolotl, LLaMA-Factory 등)가 데이터를 제대로 파싱하지 못하거나, 토큰 손실이 엉뚱한 구간에 적용됩니다.

---

## 1. Alpaca 형식

### 탄생 배경

2023년 스탠퍼드 대학이 Meta의 LLaMA 모델을 파인튜닝해 **Alpaca**를 공개하면서 이 형식이 널리 퍼졌습니다. OpenAI의 `text-davinci-003`을 사용해 52,000개의 instruction-following 데이터를 자동 생성했으며, 이 데이터셋의 구조가 사실상 표준으로 자리 잡았습니다.

### 구조

```json
{
  "instruction": "다음 문장을 영어로 번역해줘.",
  "input": "오늘 날씨가 정말 좋다.",
  "output": "The weather is really nice today."
}
```

세 가지 필드로 구성됩니다.

| 필드 | 설명 | 필수 여부 |
| :--- | :--- | :--- |
| `instruction` | 모델에게 내리는 지시사항 | 필수 |
| `input` | 지시와 함께 제공하는 입력 컨텍스트 | 선택 (없으면 빈 문자열 `""`) |
| `output` | 모델이 생성해야 하는 정답 | 필수 |

`input`이 없는 경우는 이렇게 씁니다.

```json
{
  "instruction": "피보나치 수열의 정의를 설명해줘.",
  "input": "",
  "output": "피보나치 수열은 첫 두 항이 0과 1이고, 이후 각 항이 바로 앞 두 항의 합으로 이루어지는 수열입니다. 즉, F(n) = F(n-1) + F(n-2) 로 정의됩니다."
}
```

학습 시 내부적으로는 다음과 같은 프롬프트 템플릿으로 합쳐집니다.

```
Below is an instruction that describes a task, paired with an input that
provides further context. Write a response that appropriately completes the request.

### Instruction:
{instruction}

### Input:
{input}

### Response:
{output}
```

### 실전 코드: Alpaca 형식 데이터셋 로드 및 학습

```python
from datasets import load_dataset
from transformers import AutoTokenizer, AutoModelForCausalLM
from trl import SFTTrainer, SFTConfig

# 데이터셋 로드 (Alpaca 형식)
dataset = load_dataset("yahma/alpaca-cleaned", split="train")

# 프롬프트 템플릿 적용 함수
ALPACA_TEMPLATE = """Below is an instruction that describes a task{input_part}.
Write a response that appropriately completes the request.

### Instruction:
{instruction}
{input_section}
### Response:
{output}"""

def format_alpaca(example):
    has_input = example["input"] and example["input"].strip()
    input_part = ", paired with an input that provides further context" if has_input else ""
    input_section = f"\n### Input:\n{example['input']}\n" if has_input else "\n"
    
    return {
        "text": ALPACA_TEMPLATE.format(
            input_part=input_part,
            instruction=example["instruction"],
            input_section=input_section,
            output=example["output"],
        )
    }

# 전처리 적용
formatted_dataset = dataset.map(format_alpaca, remove_columns=dataset.column_names)

# 모델 및 토크나이저 로드
model_id = "meta-llama/Llama-3.1-8B"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")

# SFT 학습 설정
training_args = SFTConfig(
    output_dir="./alpaca-finetuned",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=2e-5,
    logging_steps=10,
    save_steps=100,
)

trainer = SFTTrainer(
    model=model,
    train_dataset=formatted_dataset,
    args=training_args,
    tokenizer=tokenizer,
    dataset_text_field="text",
    max_seq_length=2048,
)

trainer.train()
```

### 언제 쓰는가?

- **단순 지시 따르기(instruction following)** 태스크를 학습할 때
- 멀티턴 대화 없이 **단발성 질문-답변** 쌍이 데이터의 전부일 때
- 레거시 베이스 모델(LLaMA 1/2, Mistral base)을 처음 instruction-following 모델로 만들 때

> {: .prompt-warning }
> **주의:** Alpaca 형식은 **싱글턴(single-turn)** 구조입니다. 대화 히스토리를 학습시키려면 ShareGPT 또는 ChatML 형식으로 전환해야 합니다.

---

## 2. ShareGPT 형식

### 탄생 배경

ChatGPT가 등장한 이후, 사용자들이 자신의 대화 내용을 공유하는 플랫폼인 **ShareGPT**에서 수집된 데이터가 오픈소스 커뮤니티에 퍼지면서 이 형식이 생겨났습니다. Vicuna, WizardLM, OpenHermes 같은 초기 오픈소스 챗 모델들이 이 형식으로 학습됐습니다.

### 구조

```json
{
  "conversations": [
    {
      "from": "system",
      "value": "당신은 친절한 AI 어시스턴트입니다."
    },
    {
      "from": "human",
      "value": "파이썬에서 리스트와 튜플의 차이점이 뭐야?"
    },
    {
      "from": "gpt",
      "value": "리스트(list)는 변경 가능한(mutable) 자료형으로 요소를 추가, 삭제, 수정할 수 있습니다. 반면 튜플(tuple)은 변경 불가능한(immutable) 자료형으로 한 번 생성되면 수정이 불가합니다..."
    },
    {
      "from": "human",
      "value": "그럼 성능 차이도 있어?"
    },
    {
      "from": "gpt",
      "value": "네, 성능 차이가 있습니다. 튜플은 불변이기 때문에 파이썬 인터프리터가 최적화를 적용할 수 있어 리스트보다 약간 더 빠릅니다..."
    }
  ]
}
```

| 필드 | 설명 |
| :--- | :--- |
| `conversations` | 대화 턴(turn)의 리스트 |
| `from` | 발화자. `system` / `human` / `gpt` 세 가지 값 |
| `value` | 해당 턴의 발화 내용 |

### 실전 코드: ShareGPT 형식 → ChatML 변환 후 학습

```python
from datasets import load_dataset
from transformers import AutoTokenizer, AutoModelForCausalLM
from trl import SFTTrainer, SFTConfig

# ShareGPT 형식 데이터셋 로드
dataset = load_dataset("philschmid/guanaco-sharegpt-style", split="train")

model_id = "Qwen/Qwen2.5-7B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)

def sharegpt_to_chatml(example):
    """
    ShareGPT 형식을 토크나이저의 apply_chat_template()에 맞는
    messages 리스트로 변환합니다.
    """
    role_map = {"human": "user", "gpt": "assistant", "system": "system"}
    
    messages = []
    for turn in example["conversations"]:
        role = role_map.get(turn["from"], turn["from"])
        messages.append({"role": role, "content": turn["value"]})
    
    # 토크나이저의 채팅 템플릿을 적용해 최종 학습 텍스트 생성
    text = tokenizer.apply_chat_template(
        messages,
        tokenize=False,
        add_generation_prompt=False,
    )
    return {"text": text}

formatted_dataset = dataset.map(
    sharegpt_to_chatml,
    remove_columns=dataset.column_names
)

model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")

training_args = SFTConfig(
    output_dir="./sharegpt-finetuned",
    num_train_epochs=2,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,
    learning_rate=1e-5,
    warmup_ratio=0.05,
    lr_scheduler_type="cosine",
    bf16=True,                   # A100/H100에서는 bf16 권장
    logging_steps=10,
)

trainer = SFTTrainer(
    model=model,
    train_dataset=formatted_dataset,
    args=training_args,
    tokenizer=tokenizer,
    dataset_text_field="text",
    max_seq_length=4096,
)

trainer.train()
```

### 언제 쓰는가?

- **멀티턴 대화** 데이터를 학습할 때
- 기존 ChatGPT 대화 데이터를 그대로 활용할 때
- 시스템 프롬프트(페르소나)가 포함된 대화를 학습할 때

> {: .prompt-tip }
> **팁:** 현재 Axolotl, LLaMA-Factory 등 주요 파인튜닝 프레임워크가 ShareGPT 형식을 네이티브로 지원하므로 별도 변환 없이 config에 `dataset_type: sharegpt`만 지정해도 됩니다.

---

## 3. ChatML 형식

### 탄생 배경

**ChatML(Chat Markup Language)**은 OpenAI가 GPT-3.5/4의 Chat API를 설계하면서 만든 형식입니다. `<|im_start|>`, `<|im_end|>` 같은 특수 토큰을 사용해 역할과 발화를 명확히 구분합니다. 현재 가장 널리 쓰이는 표준 형식으로, Qwen, Yi, DeepSeek, Mistral 등 대부분의 최신 오픈소스 모델이 이 방식을 채택했습니다.

### 구조 (JSON)

```json
{
  "messages": [
    {
      "role": "system",
      "content": "당신은 파이썬 전문가 AI 어시스턴트입니다."
    },
    {
      "role": "user",
      "content": "데코레이터(decorator) 패턴을 간단한 예시로 설명해줘."
    },
    {
      "role": "assistant",
      "content": "데코레이터는 기존 함수를 수정하지 않고 기능을 추가하는 패턴입니다.\n\n```python\ndef log_call(func):\n    def wrapper(*args, **kwargs):\n        print(f'{func.__name__} 호출됨')\n        return func(*args, **kwargs)\n    return wrapper\n\n@log_call\ndef greet(name):\n    print(f'안녕하세요, {name}!')\n\ngreet('철수')  # '호출됨' 로그 후 '안녕하세요, 철수!' 출력\n```"
    }
  ]
}
```

### 토크나이저 적용 후 실제 형태

```
<|im_start|>system
당신은 파이썬 전문가 AI 어시스턴트입니다.<|im_end|>
<|im_start|>user
데코레이터(decorator) 패턴을 간단한 예시로 설명해줘.<|im_end|>
<|im_start|>assistant
데코레이터는 기존 함수를 수정하지 않고 기능을 추가하는 패턴입니다.
...<|im_end|>
```

### 실전 코드: ChatML 형식 데이터 직접 구성 및 학습

```python
import json
from datasets import Dataset
from transformers import AutoTokenizer, AutoModelForCausalLM
from trl import SFTTrainer, SFTConfig

# -------------------------------------------------------
# ChatML 형식의 커스텀 데이터셋 직접 구성
# -------------------------------------------------------
raw_data = [
    {
        "messages": [
            {"role": "system", "content": "당신은 SQL 전문가입니다."},
            {"role": "user", "content": "INNER JOIN과 LEFT JOIN의 차이점은?"},
            {"role": "assistant", "content": "INNER JOIN은 두 테이블에서 조건을 만족하는 행만 반환합니다. LEFT JOIN은 왼쪽 테이블의 모든 행을 반환하고, 오른쪽 테이블에 매칭되는 행이 없으면 NULL로 채웁니다."},
        ]
    },
    {
        "messages": [
            {"role": "system", "content": "당신은 SQL 전문가입니다."},
            {"role": "user", "content": "인덱스(Index)를 언제 써야 해?"},
            {"role": "assistant", "content": "WHERE 조건, JOIN, ORDER BY에 자주 등장하는 컬럼에 인덱스를 추가하면 조회 성능이 크게 향상됩니다. 단, INSERT/UPDATE/DELETE가 잦은 컬럼은 인덱스 유지 비용도 함께 늘어납니다."},
        ]
    },
]

model_id = "Qwen/Qwen2.5-7B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)

def apply_chat_template(example):
    """각 예시에 토크나이저의 채팅 템플릿을 적용합니다."""
    text = tokenizer.apply_chat_template(
        example["messages"],
        tokenize=False,
        add_generation_prompt=False,
    )
    return {"text": text}

dataset = Dataset.from_list(raw_data)
formatted_dataset = dataset.map(apply_chat_template)

model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")

training_args = SFTConfig(
    output_dir="./chatml-finetuned",
    num_train_epochs=3,
    per_device_train_batch_size=2,
    learning_rate=2e-5,
    bf16=True,
)

trainer = SFTTrainer(
    model=model,
    train_dataset=formatted_dataset,
    args=training_args,
    tokenizer=tokenizer,
    dataset_text_field="text",
    max_seq_length=4096,
)

trainer.train()
```

### Alpaca vs ShareGPT vs ChatML 비교

| 항목 | Alpaca | ShareGPT | ChatML |
| :--- | :---: | :---: | :---: |
| 멀티턴 대화 지원 | ❌ | ✅ | ✅ |
| 시스템 프롬프트 지원 | ❌ | ✅ | ✅ |
| 최신 모델과의 호환성 | 낮음 | 보통 | **높음** |
| 현재 사용 권장도 | 레거시 | 보통 | **권장** |
| 툴 호출(Function Calling) | ❌ | 부분적 | ✅ |

---

## 4. DPO Preference 형식

### 탄생 배경

SFT만으로는 모델이 "어떤 답이 더 나은지"를 배울 수 없습니다. 선호도 최적화(Preference Optimization)는 동일한 질문에 대해 **선택된(chosen) 답변**과 **거부된(rejected) 답변**을 쌍으로 학습시켜, 모델이 더 나은 답을 선호하도록 유도합니다.

**DPO(Direct Preference Optimization)**는 복잡한 보상 모델(reward model) 없이 직접 이 선호도를 학습하는 방식으로, 2023년 이후 RLHF의 실용적인 대안으로 자리 잡았습니다.

### 구조

```json
{
  "prompt": "한국의 수도는 어디인가요?",
  "chosen": "한국의 수도는 서울입니다. 서울은 한강을 중심으로 발전한 도시로, 약 950만 명의 인구가 거주하는 대한민국의 정치·경제·문화 중심지입니다.",
  "rejected": "서울인 것 같습니다."
}
```

멀티턴 대화의 경우 아래처럼 `messages` 리스트 형태로 구성하기도 합니다.

```json
{
  "prompt": [
    {"role": "system", "content": "당신은 도움이 되는 어시스턴트입니다."},
    {"role": "user", "content": "지구 온난화의 주요 원인을 설명해줘."}
  ],
  "chosen": [
    {
      "role": "assistant",
      "content": "지구 온난화의 주요 원인은 온실가스 증가입니다. 특히 화석연료 연소로 발생하는 이산화탄소(CO₂)가 가장 큰 비중을 차지하며, 메탄(CH₄), 아산화질소(N₂O)도 중요한 역할을 합니다. 이 가스들이 대기에 축적되면 태양열이 지구 밖으로 방출되는 것을 막아 기온이 상승합니다."
    }
  ],
  "rejected": [
    {
      "role": "assistant",
      "content": "태양이 뜨거워져서 그런 것 같아요."
    }
  ]
}
```

### 선호도 데이터셋을 만드는 방법

| 방법 | 설명 | 비용 |
| :--- | :--- | :--- |
| **인간 어노테이션** | 실제 사람이 두 답변 중 더 나은 것을 선택 | 높음 |
| **강모델 vs 약모델** | GPT-4 응답을 `chosen`, 약한 모델 응답을 `rejected`로 | 중간 |
| **온도(temperature) 샘플링** | 같은 모델에서 낮은 온도(정확) vs 높은 온도(창의적) 응답 쌍 생성 | 낮음 |
| **LLM-as-judge** | 강력한 LLM이 두 응답을 자동 평가해 순위 결정 | 낮음 |

### 실전 코드: DPO 학습

```python
from datasets import load_dataset
from transformers import AutoTokenizer, AutoModelForCausalLM
from trl import DPOTrainer, DPOConfig

model_id = "Qwen/Qwen2.5-7B-Instruct"  # SFT가 완료된 모델

# ----------------------------------------------------------
# DPO 형식 데이터셋 로드
# chosen / rejected 필드가 있는 preference 데이터셋 사용
# ----------------------------------------------------------
dataset = load_dataset("Intel/orca_dpo_pairs", split="train")

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")

# DPO는 참조 모델(reference model)이 필요합니다.
# ref_model을 None으로 설정하면 학습 시작 시점의 모델을 자동으로 참조 모델로 사용합니다.
dpo_config = DPOConfig(
    output_dir="./dpo-finetuned",
    num_train_epochs=1,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,
    learning_rate=5e-7,       # DPO는 SFT보다 훨씬 낮은 학습률 사용
    beta=0.1,                 # KL 발산 페널티 강도 (높을수록 보수적)
    bf16=True,
    logging_steps=10,
    remove_unused_columns=False,
)

trainer = DPOTrainer(
    model=model,
    ref_model=None,           # None이면 내부적으로 참조 모델 자동 생성
    args=dpo_config,
    train_dataset=dataset,
    tokenizer=tokenizer,
    beta=0.1,
    max_length=2048,
    max_prompt_length=512,
)

trainer.train()
```

> {: .prompt-tip }
> **팁:** DPO 학습은 반드시 **SFT가 완료된 모델** 위에서 진행해야 합니다. Base 모델에 바로 DPO를 적용하면 학습이 불안정해집니다.

---

## 5. 형식 선택 플로우차트

```
파인튜닝 목적을 결정합니다
          │
          ▼
   단순 지시 따르기?
   (질문 → 정답 1회)
     │          │
    Yes         No
     │          │
     ▼          ▼
  Alpaca     멀티턴 대화가
  형식       필요한가?
               │          │
              Yes         No
               │          │
               ▼          ▼
  ShareGPT / ChatML    [다시 검토]
  (최신 모델 → ChatML 권장)
          │
          ▼
   SFT 학습 완료 후,
   더 나은 답변을 선호하도록
   정렬(alignment)이 필요한가?
          │          │
         Yes         No
          │          │
          ▼          ▼
   DPO Preference   배포
   형식으로 추가 학습
```

---

## 6. 형식별 대표 공개 데이터셋

| 형식 | 데이터셋 | HuggingFace ID | 규모 |
| :--- | :--- | :--- | :--- |
| Alpaca | Alpaca Cleaned | `yahma/alpaca-cleaned` | 52K |
| Alpaca | OpenHermes-2.5 | `teknium/OpenHermes-2.5` | 1M |
| ShareGPT | Guanaco | `philschmid/guanaco-sharegpt-style` | 9.8K |
| ShareGPT | SlimOrca | `Open-Orca/SlimOrca` | 518K |
| ChatML | UltraChat | `HuggingFaceH4/ultrachat_200k` | 200K |
| DPO | Orca DPO Pairs | `Intel/orca_dpo_pairs` | 12.9K |
| DPO | UltraFeedback | `argilla/ultrafeedback-binarized-preferences` | 64K |

---

## 7. 정리

| 형식 | 구조 | 적합한 태스크 | 권장 도구 |
| :--- | :--- | :--- | :--- |
| **Alpaca** | instruction / input / output | 단발성 지시 따르기 | TRL SFTTrainer |
| **ShareGPT** | conversations (from/value) | 멀티턴 챗봇 | LLaMA-Factory, Axolotl |
| **ChatML** | messages (role/content) | 멀티턴 + 최신 모델 | TRL SFTTrainer |
| **DPO Preference** | prompt / chosen / rejected | 정렬, 안전성 향상 | TRL DPOTrainer |

파인튜닝 데이터 형식은 **"어떤 모델로 어떤 태스크를 학습하는가"**에 따라 달라집니다. 2026년 기준으로 최신 오픈소스 모델(Llama 3.1, Qwen 2.5, Mistral)을 파인튜닝한다면 **ChatML + DPO** 조합이 가장 강력한 결과를 냅니다. 레거시 파이프라인을 유지해야 하거나 단순한 태스크라면 Alpaca나 ShareGPT도 여전히 충분히 유효한 선택입니다.

---
