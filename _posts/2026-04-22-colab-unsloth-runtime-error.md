---
layout: post
title: "Colab에서 Qwen 파인튜닝하다 만난 세 가지 에러"
date: 2026-04-22 12:00:00 +0900
categories:
  - LLM
  - Debugging
tags: [Unsloth, Colab, Qwen, PyTorch, torchcodec, Finetune, Debugging]
math: false
---

> {: .prompt-tip }
> **요약:** Google Colab에서 Unsloth로 Qwen 모델을 파인튜닝하는 도중 `RuntimeError: Could not load libtorchcodec`, `AttributeError: module 'torch._dynamo' has no attribute 'utils'`, `ValueError: torchcodec.__spec__ is not set` 세 가지 에러가 연쇄적으로 발생했습니다. 원인은 Colab 런타임이 개발 도중 자동으로 업데이트되어, **PyTorch 2.8.0(Nightly)** 이 설치된 최신 런타임으로 교체됐기 때문이었습니다. **2026-04 버전으로 런타임을 되돌리자 모든 에러가 사라졌습니다.**

---

Colab에서 Qwen 모델을 파인튜닝하다가 꽤 황당한 상황을 맞이했습니다. 코드는 전혀 건드리지 않았는데, 어느 날 갑자기 모델 로드 첫 줄부터 에러가 터졌습니다. 게다가 에러를 고치려는 시도마다 새로운 에러가 나타나서, 마치 두더지 잡기 게임처럼 에러를 하나 누르면 다른 에러가 튀어나오는 상황이 반복됐습니다.

이 글은 그 과정을 기록한 디버깅 노트입니다.

---

## 1. 문제의 시작: 아무것도 안 했는데 에러가?

### 실행 환경

```python
from unsloth import FastVisionModel

MODEL_NAME = "unsloth/Qwen3.5-4B"
LOAD_IN_4BIT = False

model, tokenizer = FastVisionModel.from_pretrained(
    MODEL_NAME,
    load_in_4bit=LOAD_IN_4BIT,
)
FastVisionModel.for_inference(model)
```

특별한 것 없는 평범한 모델 로드 코드입니다. 이전에 잘 돌아가던 코드였는데, 어느 날부터 첫 줄 `from unsloth import FastVisionModel` 에서부터 에러가 발생하기 시작했습니다.

---

## 2. 에러 ① — `RuntimeError: Could not load libtorchcodec`

### 에러 전문

```
RuntimeError: Could not load libtorchcodec.
Likely causes:
  1. FFmpeg is not properly installed in your environment.
     We support versions 4, 5, 6, 7, and 8.
  2. The PyTorch version (2.8.0.dev...) is not compatible
     with this version of TorchCodec.
  
[start of libtorchcodec loading traceback]
FFmpeg version 8:
  OSError: libavutil.so.60: cannot open shared object file:
  No such file or directory
FFmpeg version 7:
  OSError: libavutil.so.59: cannot open shared object file:
  No such file or directory
...
[end of libtorchcodec loading traceback]
```

### 무슨 뜻인가?

`torchcodec`은 PyTorch 생태계에서 비디오·오디오 파일을 디코딩하는 라이브러리입니다. 내부적으로 FFmpeg의 공유 라이브러리(`.so` 파일)를 동적으로 불러와 사용합니다.

에러 메시지를 풀어보면 이렇습니다.

```
"libavutil.so.60이라는 FFmpeg 라이브러리 파일을 찾을 수 없습니다."
```

`libavutil.so.60`에서 숫자 **60**은 FFmpeg의 major 버전 번호에 대응하는 라이브러리 버전입니다. 이 파일이 시스템에 없다는 것은, 설치된 FFmpeg 버전이 현재 `torchcodec`이 요구하는 버전과 맞지 않는다는 뜻입니다.

### 왜 갑자기 발생했나?

이 에러의 구조를 이해하려면 두 가지 컴포넌트 사이의 버전 관계를 알아야 합니다.

```
[Colab 시스템 레이어]
  FFmpeg (운영체제에 설치된 시스템 라이브러리)
      ↕ 버전이 맞아야 함
[Python 패키지 레이어]
  torchcodec (PyTorch가 설치한 패키지)
```

| 구성 요소 | 상태 |
| :--- | :--- |
| **Colab의 시스템 FFmpeg** | Ubuntu 22.04 기본 패키지 (상대적으로 구버전) |
| **새 런타임의 torchcodec** | PyTorch 2.8.0 Nightly와 함께 설치된 최신 버전 |

Colab 런타임이 최신 버전으로 업데이트되면서 **PyTorch 2.8.0 Nightly**가 설치됐고, 이 PyTorch에 딸려온 `torchcodec`은 더 높은 버전의 FFmpeg 라이브러리를 필요로 하는데, 시스템에는 구버전 FFmpeg만 있으니 발생한 버전 충돌입니다.

Unsloth의 `FastVisionModel`은 내부적으로 비전(이미지/비디오) 처리를 위해 `torchcodec`에 의존하고 있기 때문에, 이 에러가 `from unsloth import ...` 단계에서 즉시 터지는 것입니다.

### 해결 시도 (→ 실패)

첫 번째 반응은 `torchcodec`을 임포트 전에 비활성화해보는 것이었습니다.

```python
# sys.modules에 가짜 모듈을 끼워넣어 실제 임포트를 막는 시도
import sys
from unittest.mock import MagicMock

sys.modules['torchcodec'] = MagicMock()

from unsloth import FastVisionModel  # 이제 괜찮겠지?
```

결과는 새로운 에러였습니다.

---

## 3. 에러 ② — `AttributeError: module 'torch._dynamo' has no attribute 'utils'`

### 에러 전문

```
AttributeError: module 'torch._dynamo' has no attribute 'utils'

Traceback (most recent call last):
  File "<ipython-input>", line 3, in <module>
    from unsloth import FastVisionModel
  File ".../unsloth/__init__.py", line ...
    from torch._dynamo import utils as dynamo_utils
AttributeError: module 'torch._dynamo' has no attribute 'utils'
```

### 무슨 뜻인가?

`torch._dynamo`는 PyTorch의 `torch.compile()` 기능을 구동하는 핵심 모듈입니다. 이 모듈 안에 `utils`라는 서브모듈이 있어야 하는데, 없다는 에러입니다.

`torch._dynamo.utils`는 PyTorch 2.x 안정 버전에서는 표준으로 존재하는 경로입니다. 그런데 **Nightly 빌드**는 안정 릴리즈 전의 개발 중인 코드이기 때문에, 내부 모듈 구조가 언제든 바뀔 수 있습니다. 이 에러는 PyTorch 2.8.0 Nightly에서 `_dynamo` 모듈의 내부 구조가 리팩터링되는 과정에서 기존 경로가 변경됐고, Unsloth가 아직 그 변경을 반영하지 못해서 발생한 것입니다.

```
안정 버전 (stable): torch._dynamo.utils  → 존재함 ✅
Nightly 빌드:       torch._dynamo.utils  → 없거나 경로 변경됨 ❌
```

즉 첫 번째 에러를 우회하려는 시도가 두 번째 에러를 드러낸 것이 아니라, **Nightly 버전 자체가 이미 근본적인 호환성 문제를 안고 있었던 것**입니다.

### 해결 시도 (→ 실패)

`MagicMock`으로는 이 문제를 해결할 수 없었습니다. `torchcodec` 자체를 더 정교하게 패치해보기로 했습니다.

```python
import sys
import types

# 빈 모듈을 직접 만들어 끼워넣는 시도
fake_torchcodec = types.ModuleType('torchcodec')
sys.modules['torchcodec'] = fake_torchcodec

from unsloth import FastVisionModel
```

결과는 또 다른 에러였습니다.

---

## 4. 에러 ③ — `ValueError: torchcodec.__spec__ is not set`

### 에러 전문

```
ValueError: torchcodec.__spec__ is not set
  (probably because it was imported before importlib.machinery was imported)

Traceback (most recent call last):
  File ".../importlib/_bootstrap.py", line ...
    raise ValueError(...)
ValueError: torchcodec.__spec__ is not set
```

### 무슨 뜻인가?

Python의 모든 임포트 시스템은 `importlib`이 관리합니다. 모듈이 정상적으로 임포트되면 `__spec__` 이라는 속성이 자동으로 설정됩니다. 이것은 "이 모듈은 어디서 어떻게 불러왔는지"에 대한 메타정보를 담은 명세서(ModuleSpec)입니다.

직접 만든 `types.ModuleType` 객체는 `importlib`을 통한 정상 임포트 과정을 거치지 않아서 `__spec__`이 `None`인 상태입니다. 문제는 Python 3.12부터 임포트 시스템의 검증이 강화되어, 어떤 코드 경로에서 `__spec__`이 `None`인 모듈을 참조하면 이 에러를 발생시킨다는 것입니다.

```
Python 3.10/3.11: __spec__ = None 인 모듈 → 경고 없이 통과
Python 3.12+:     __spec__ = None 인 모듈 → ValueError 발생
```

Colab의 새 런타임은 Python 3.12를 사용하고 있어서 기존에 통하던 Mock 패치 방식이 막혀버린 것입니다.

`__spec__`을 직접 만들어 붙이는 방법도 있지만, 이는 `importlib` 내부 구조를 직접 흉내내야 하는 것이라 올바른 해결책이 아닙니다.

---

## 5. 에러 연쇄 구조 정리

세 에러를 나란히 놓고 보면 하나의 거대한 원인에서 파생된 도미노였습니다.

```
[근본 원인]
  Colab 런타임이 자동 업데이트 →
  PyTorch 2.8.0 Nightly + Python 3.12 설치
          │
          ▼
[에러 ①] RuntimeError: Could not load libtorchcodec
  → Nightly PyTorch의 torchcodec ↔ 구버전 FFmpeg 불일치
          │
          │ 우회 시도 (MagicMock)
          ▼
[에러 ②] AttributeError: torch._dynamo has no attribute 'utils'
  → Nightly 빌드의 내부 구조 변경으로 기존 경로 소멸
          │
          │ 우회 시도 (types.ModuleType)
          ▼
[에러 ③] ValueError: torchcodec.__spec__ is not set
  → Python 3.12의 강화된 모듈 검증 로직에 막힘
```

각각의 에러를 패치하는 것은 증상을 다스리는 것이었고, 실제 원인인 **런타임 버전 불일치**를 고치지 않는 한 계속 새로운 에러가 나타날 수밖에 없는 구조였습니다.

---

## 6. 진짜 원인: Colab 런타임이 조용히 업데이트됐다

Colab은 구글이 관리하는 클라우드 환경이기 때문에 런타임 이미지가 주기적으로 업데이트됩니다. 사용자에게 명시적으로 알리지 않아도 새 런타임 버전으로 교체될 수 있습니다.

### Colab 런타임 버전 이력

Google의 공식 런타임 FAQ 페이지에 공개된 내용을 보면, 각 런타임이 어떤 스택을 포함하는지 알 수 있습니다.

| 런타임 버전 | Python | PyTorch | 비고 |
| :--- | :---: | :---: | :--- |
| **2026-04 (현재)** | 3.12.13 | **2.10.0** | 안정 버전 |
| 이전 최신 | 3.12.12 | **2.9.0** | 안정 버전 |
| 그 이전 | 3.12.12 | **2.8.0** | ⚠️ Nightly 계열 포함 시기 |
| 구버전 | 3.11.13 | **2.6.0** | 안정 버전 |

개발을 진행하던 시점에 Colab 런타임이 PyTorch 2.8.0 계열로 업데이트됐고, 이 버전은 Nightly 빌드 기반이라 `torchcodec`, `torch._dynamo` 등의 라이브러리와 호환성 문제를 일으켰습니다.

### 내 런타임 버전 확인하는 방법

Colab 셀에서 아래를 실행하면 현재 환경을 즉시 파악할 수 있습니다.

```python
import torch
import sys
import platform

print(f"Python    : {sys.version}")
print(f"PyTorch   : {torch.__version__}")
print(f"CUDA      : {torch.version.cuda}")
print(f"OS        : {platform.platform()}")
```

문제가 발생하던 시점의 출력은 이런 형태였을 것입니다.

```
Python    : 3.12.12 (main, ...)
PyTorch   : 2.8.0.dev20250XXX+cu12X  ← dev 또는 nightly 빌드
CUDA      : 12.X
```

`torch.__version__`에 `.dev` 또는 `nightly`가 포함되어 있다면 Nightly 빌드입니다.

---

## 7. 해결: 런타임 버전 변경

### 변경 방법

Colab 상단 메뉴에서 런타임 버전을 직접 선택할 수 있습니다.

```
상단 메뉴 → 런타임(Runtime) → 런타임 유형 변경(Change runtime type)
→ "런타임 버전" 드롭다운에서 원하는 버전 선택
→ 저장(Save) → 런타임 재시작
```

![Colab 런타임 버전 변경 메뉴]

**2026-04** 버전(PyTorch 2.10.0, Python 3.12.13)으로 변경하자, 세 에러가 모두 사라지고 모델 로드가 정상적으로 완료됐습니다.



### 왜 2026-04 버전에서는 괜찮은가?

2026-04 런타임(PyTorch 2.10.0)은 안정 릴리즈 버전입니다. 이 버전에서는:

- `torchcodec`이 시스템 FFmpeg와 호환되는 버전으로 설치됩니다
- `torch._dynamo.utils` 경로가 안정적으로 존재합니다
- Unsloth가 테스트하고 지원 선언한 PyTorch 버전 범위에 들어옵니다

결국 Nightly 빌드는 기능 개발 중인 불안정한 버전이고, 서드파티 라이브러리들(Unsloth, torchcodec 등)이 아직 맞추지 못한 인터페이스 변경이 포함되어 있어서 발생한 문제였습니다.

---

## 8. 재발 방지: Colab에서 런타임 안정성 확보하기

같은 상황을 다시 만나지 않으려면 노트북 실행 초반에 환경 버전을 검사하는 습관이 도움이 됩니다.

### 런타임 버전 검사 코드

```python
import torch
import sys

def check_colab_runtime():
    """
    현재 Colab 런타임 환경을 출력하고
    Nightly 빌드 여부를 경고합니다.
    """
    torch_ver = torch.__version__
    python_ver = sys.version.split()[0]
    
    print("=" * 45)
    print(f"  Python  : {python_ver}")
    print(f"  PyTorch : {torch_ver}")
    print(f"  CUDA    : {torch.version.cuda}")
    print("=" * 45)
    
    # Nightly / dev 빌드 감지
    if "dev" in torch_ver or "nightly" in torch_ver.lower():
        print()
        print("⚠️  경고: Nightly(개발 중) 빌드가 감지되었습니다.")
        print("   Unsloth 등 서드파티 라이브러리와 호환되지 않을 수 있습니다.")
        print("   Colab 상단 메뉴 → 런타임 → 런타임 유형 변경에서")
        print("   안정 버전(예: 2026-04)으로 변경하는 것을 권장합니다.")
    else:
        print("✅ 안정(Stable) 버전의 PyTorch입니다.")

check_colab_runtime()
```

**안정 버전 출력 예시:**
```
=============================================
  Python  : 3.12.13
  PyTorch : 2.10.0+cu12X
  CUDA    : 12.X
=============================================
✅ 안정(Stable) 버전의 PyTorch입니다.
```

**Nightly 버전 출력 예시:**
```
=============================================
  Python  : 3.12.12
  PyTorch : 2.8.0.dev20250XXXXX+cu12X
  CUDA    : 12.X
=============================================

⚠️  경고: Nightly(개발 중) 빌드가 감지되었습니다.
   Unsloth 등 서드파티 라이브러리와 호환되지 않을 수 있습니다.
   Colab 상단 메뉴 → 런타임 → 런타임 유형 변경에서
   안정 버전(예: 2026-04)으로 변경하는 것을 권장합니다.
```

---

## 9. 정리

| 에러 | 직접 원인 | 근본 원인 |
| :--- | :--- | :--- |
| `RuntimeError: Could not load libtorchcodec` | Nightly PyTorch의 torchcodec ↔ 구버전 FFmpeg 버전 불일치 | Colab 런타임 자동 업데이트 |
| `AttributeError: torch._dynamo has no attribute 'utils'` | Nightly 빌드에서 내부 모듈 구조 변경 | Colab 런타임 자동 업데이트 |
| `ValueError: torchcodec.__spec__ is not set` | Python 3.12의 강화된 모듈 검증으로 Mock 패치 실패 | Colab 런타임 자동 업데이트 |
| **해결** | **Colab 런타임을 2026-04(PyTorch 2.10.0)으로 변경** | — |

핵심 교훈 두 가지입니다.

**1. 증상이 아닌 원인을 잡아라.**  
에러 메시지만 보고 각각의 에러를 패치하는 것은 두더지 잡기입니다. 세 에러가 연쇄적으로 나타났다면, 그 공통 뿌리를 먼저 찾아야 합니다. 이 경우 뿌리는 런타임 버전 불일치였습니다.

**2. Nightly 빌드는 안정 버전이 아니다.**  
PyTorch Nightly는 다음 릴리즈를 준비하는 개발 중 버전입니다. 내부 API가 예고 없이 바뀌고, 서드파티 생태계가 아직 따라잡지 못한 변경이 포함될 수 있습니다. 특별한 이유가 없다면 **서드파티 라이브러리(Unsloth, transformers 등)를 함께 쓰는 환경에서는 안정 버전을 사용**하는 것이 훨씬 안전합니다.
