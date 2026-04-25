---
layout: post
title: "12 Factor App — 클라우드 시대의 앱 설계 원칙"
date: 2026-04-25 15:00:00 +0900
categories:
  - DevOps
  - Architecture
tags: [12FactorApp, CloudNative, DevOps, Heroku, Kubernetes, Docker]
math: false
---

> {: .prompt-tip }
> **요약:** 12 Factor App은 Heroku 개발자들이 수천 개의 앱을 운영하며 발견한 **"배포하기 좋은 앱의 공통점"** 12가지입니다. 특정 언어나 프레임워크에 종속되지 않으며, Docker·Kubernetes·클라우드 환경에서도 그대로 적용되는 시대를 초월한 원칙입니다.

---

CI/CD 파이프라인을 갖추고, IaC로 인프라를 코드화하고, 빌드 파이프라인을 자동화했습니다. 이제 배포는 버튼 하나로 끝납니다. 그런데 막상 배포를 해보면 이런 일이 생깁니다.

> "로컬에서는 됐는데 운영 서버에서는 왜 안 돼요?"  
> "DB 연결 정보가 코드에 박혀 있어서, 환경을 바꾸려면 빌드를 새로 해야 해요."  
> "서버를 2대로 늘렸더니 세션이 날아가요."

이 문제들은 파이프라인이나 인프라의 문제가 아닙니다. **앱 자체가 배포하기 어렵게 설계된 것**이 문제입니다. 아무리 좋은 파이프라인을 갖춰도, 앱 설계가 잘못됐다면 배포는 항상 불안합니다.

이 문제를 해결하는 설계 원칙이 **12 Factor App**입니다.

---

## 12 Factor App이란?

[https://12factor.net/ko](https://12factor.net/ko)

2011년, 클라우드 플랫폼 **Heroku**의 공동 창업자 Adam Wiggins가 수천 개의 앱을 운영하며 발견한 공통점을 정리해 발표한 문서입니다. **"배포하기 좋은 앱들은 어떤 특징을 공유하는가?"** 라는 질문에 대한 답이 12가지로 정리되어 있습니다.

특정 언어나 프레임워크에 종속되지 않습니다. Python, Java, Node.js, Go — 어떤 언어로 작성된 앱이든 적용할 수 있습니다. 2011년에 작성된 문서지만, Docker·Kubernetes·서버리스 환경이 주류가 된 2026년에도 **그대로 유효한 원칙**입니다. 오히려 컨테이너 환경은 12 Factor의 원칙을 더 자연스럽게 따르도록 설계되어 있습니다.

12가지를 이해하기 쉽게 4개의 그룹으로 묶어 설명합니다.

---

## 그룹 1: 코드 관리 (Factor 1 ~ 3)

### Factor 1 — 코드베이스(Codebase): 하나의 앱, 하나의 저장소

> "하나의 애플리케이션은 하나의 버전 관리 저장소에서 관리한다."

```
❌ 잘못된 예
  repo-frontend/   # 프론트엔드 코드
  repo-backend/    # 백엔드 코드
  repo-infra/      # 인프라 설정
  → 이 세 개가 하나의 서비스라면, 하나의 저장소에서 함께 관리해야 한다

✅ 올바른 예
  my-service/
  ├── frontend/
  ├── backend/
  └── infra/       ← IaC 파일도 같은 저장소에 포함
```

중요한 것은 **환경(개발/스테이징/운영)의 차이를 코드로 구분하지 않는다**는 점입니다. `if ENV == 'production'` 같은 분기가 코드 안에 있다면, 환경마다 코드가 달라지는 것이고 이는 곧 신뢰할 수 없는 배포로 이어집니다. 환경의 차이는 코드가 아니라 **설정(환경 변수)**으로 관리합니다.

코드베이스는 하나지만 배포는 여러 환경으로 이루어집니다.

```
하나의 코드베이스 (Git 저장소)
       │
       ├──▶ 개발 환경 배포  (환경 변수: DEV DB URL)
       ├──▶ 스테이징 배포   (환경 변수: STAGING DB URL)
       └──▶ 운영 배포       (환경 변수: PROD DB URL)
```

### Factor 2 — 의존성(Dependencies): 명시적으로 선언하기

> "모든 라이브러리 의존성을 Manifest 파일에 세밀하게 기술하고, 격리해서 실행한다."

| 언어 | 의존성 선언 파일 | 격리 실행 도구 |
| :--- | :--- | :--- |
| Python | `requirements.txt`, `pyproject.toml` | `venv`, `poetry` |
| Node.js | `package.json` | `npm`, `yarn` |
| Ruby | `Gemfile` | `bundler` |
| Java | `pom.xml`, `build.gradle` | Maven, Gradle |
| Go | `go.mod` | Go Modules |

"그냥 서버에 설치된 라이브러리 쓰면 되는 거 아닌가요?"라고 생각할 수 있습니다. 이 방식의 문제는 **암묵적 의존성**입니다. 코드를 읽어서는 어떤 버전의 라이브러리가 필요한지 알 수 없고, 다른 서버에서는 버전이 달라 작동하지 않을 수 있습니다.

```python
# ❌ 암묵적 의존성 — 어떤 버전인지 알 수 없음
# 그냥 서버에 설치된 라이브러리를 씀
import flask
import sqlalchemy

# ✅ 명시적 의존성 — requirements.txt
# flask==3.1.0
# sqlalchemy==2.0.30
# 버전이 고정되어 있어 어떤 환경에서도 동일하게 재현 가능
```

Docker 컨테이너를 사용하면 이 원칙이 자연스럽게 지켜집니다. `Dockerfile`에 모든 의존성 설치 과정이 명시되고, 이미지 자체가 격리된 실행 환경이 됩니다.

```dockerfile
# Dockerfile — 의존성이 명시적으로 선언됨
FROM python:3.12-slim

WORKDIR /app

# 의존성 파일을 먼저 복사 (Docker 레이어 캐싱 활용)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Factor 3 — 설정(Config): 환경 변수로 분리하기

> "코드에 리소스 정보(DB 연결, API 키 등)를 포함하지 않는다. 설정은 환경 변수에 저장한다."

이 원칙을 지키는지 확인하는 간단한 기준이 있습니다.

> **"코드를 지금 당장 GitHub에 공개해도 보안에 문제가 없는가?"**

만약 코드에 DB 비밀번호, API 키, IP 주소가 하드코딩되어 있다면, 이 질문의 답은 "아니오"입니다.

```python
# ❌ 잘못된 예 — 설정이 코드에 하드코딩
DATABASE_URL = "postgresql://admin:password123@192.168.1.10:5432/mydb"
STRIPE_API_KEY = "sk_live_abc123..."
REDIS_HOST = "192.168.1.20"

# ✅ 올바른 예 — 설정을 환경 변수에서 읽음
import os

DATABASE_URL = os.environ["DATABASE_URL"]
STRIPE_API_KEY = os.environ["STRIPE_API_KEY"]
REDIS_HOST = os.environ.get("REDIS_HOST", "localhost")  # 기본값 지정 가능
```

각 환경에서는 동일한 코드가 실행되지만, 환경 변수만 다르게 주입됩니다.

```bash
# 개발 환경 실행
DATABASE_URL="postgresql://localhost/mydb_dev" python app.py

# 운영 환경 실행 (코드 변경 없음)
DATABASE_URL="postgresql://rds.amazonaws.com/mydb_prod" python app.py
```

> {: .prompt-warning }
> **주의:** 설정을 환경 변수로 관리하되, 절대로 `.env` 파일을 Git에 커밋하지 마세요. `.gitignore`에 `.env`를 추가하고, `.env.example` 파일에 필요한 변수 목록(값 없이)만 공유하는 것이 표준 방식입니다.

---

## 그룹 2: 실행 환경 (Factor 4 ~ 6)

### Factor 4 — 백엔드 서비스(Backing Services): 교체 가능하게

> "DB, 메시지 큐, 캐시 등 네트워크를 통해 사용하는 모든 서비스를 '교체 가능한 부착 자원'으로 취급한다."

백엔드 서비스(Backing Service)란 앱이 정상적으로 동작하기 위해 네트워크를 통해 사용하는 모든 것입니다.

| 종류 | 예시 |
| :--- | :--- |
| 데이터베이스 | PostgreSQL, MySQL, MongoDB |
| 메시지 큐 | RabbitMQ, Apache Kafka, AWS SQS |
| 캐시 | Redis, Memcached |
| 이메일 서비스 | SendGrid, AWS SES |
| 스토리지 | AWS S3, Google Cloud Storage |

핵심은 **로컬에서 직접 운영하는 서비스와 클라우드가 제공하는 서비스를 코드 입장에서 동일하게 취급**하는 것입니다.

```python
# ✅ 올바른 예 — 환경 변수 하나만 바꾸면 DB가 교체됨
DATABASE_URL = os.environ["DATABASE_URL"]

# 개발: DATABASE_URL="postgresql://localhost/mydb"   (로컬 PostgreSQL)
# 운영: DATABASE_URL="postgresql://rds.amazonaws.com/mydb"  (AWS RDS)
# 코드는 동일. 환경 변수만 다름.
```

이 원칙이 지켜지면 로컬 MySQL을 AWS RDS로, 로컬 Redis를 ElastiCache로 전환할 때 **코드를 한 줄도 수정하지 않아도** 됩니다.

### Factor 5 — 빌드, 릴리즈, 실행(Build/Release/Run): 세 단계를 엄격히 분리

> "코드를 배포하는 과정을 빌드(Build) → 릴리즈(Release) → 실행(Run) 세 단계로 명확히 나눈다."

| 단계 | 정의 | 결과물 | 불변 여부 |
| :--- | :--- | :--- | :--- |
| **Build** | 코드 + 의존성을 컴파일해 실행 가능한 형태로 만듦 | Docker 이미지, 바이너리 | 불변(Immutable) |
| **Release** | Build 결과물 + 환경별 설정을 결합 | 배포 준비 완료 상태 | 불변(Immutable) |
| **Run** | Release를 실제 환경에서 실행 | 실행 중인 프로세스 | — |

```
❌ 잘못된 예
  운영 서버에서 git pull 후 직접 실행
  → Build/Release/Run이 뒤섞임
  → 어떤 코드가 실행 중인지 추적 불가
  → 롤백 시 다시 git pull해야 함

✅ 올바른 예 (GitHub Actions 파이프라인)
  # Build: Docker 이미지 생성
  docker build -t myapp:v1.2.3 .
  docker push registry.company.com/myapp:v1.2.3

  # Release: 이미지 + 설정 결합
  # (Kubernetes ConfigMap, 환경 변수 주입)

  # Run: 이미지를 배포
  kubectl set image deployment/myapp app=myapp:v1.2.3
```

**릴리즈는 불변(Immutable)**입니다. 한 번 만들어진 릴리즈는 절대 수정하지 않습니다. 문제가 생기면 이전 릴리즈를 그대로 다시 실행하면 됩니다. 이것이 **즉각적인 롤백**을 가능하게 하는 기반입니다.

```bash
# 문제 발생 시 이전 릴리즈로 즉시 롤백
kubectl rollout undo deployment/myapp
```

### Factor 6 — 프로세스(Processes): Stateless하게 설계

> "앱은 하나 이상의 Stateless(무상태) 프로세스로 실행한다. 어떤 것도 프로세스 내에 저장하지 않는다."

가장 많은 문제를 일으키는 원칙입니다. 이 원칙을 위반한 흔한 패턴이 **세션을 서버 메모리에 저장**하는 것입니다.

```
❌ Stateful 설계 (서버 메모리에 세션 저장)

사용자 → [서버 A] 로그인 → 세션이 서버 A 메모리에 저장
사용자 → [서버 B] 요청 → "세션이 없습니다. 로그인해주세요." ← 로그아웃됨!

서버를 2대로 늘리는 순간 이 문제가 터짐
```

```
✅ Stateless 설계 (외부 저장소에 세션 저장)

사용자 → [서버 A] 로그인 → 세션을 Redis에 저장
사용자 → [서버 B] 요청 → Redis에서 세션 조회 → 정상 처리

서버가 10대가 돼도 문제없음. 서버 A가 죽어도 서버 B가 처리함.
```

```python
# ✅ 올바른 예 — Flask + Redis로 Stateless 세션 구현
from flask import Flask
from flask_session import Session
import redis

app = Flask(__name__)

# 세션을 서버 메모리가 아닌 Redis에 저장
app.config["SESSION_TYPE"] = "redis"
app.config["SESSION_REDIS"] = redis.from_url(os.environ["REDIS_URL"])
Session(app)

@app.route("/cart/add")
def add_to_cart():
    # 어떤 서버 인스턴스가 이 요청을 처리하든
    # Redis에서 동일한 세션 데이터를 읽음
    session["cart"] = session.get("cart", []) + [request.json["item"]]
    return {"status": "added"}
```

Stateless 설계는 단순히 세션 문제에 그치지 않습니다. **프로세스(서버 인스턴스)가 언제든 죽고 새로 생겨도 서비스에 영향이 없어야 한다**는 원칙입니다. 업로드한 파일을 로컬 디스크에 저장하는 것도 이 원칙 위반입니다. 파일은 S3 같은 외부 스토리지에 저장해야 합니다.

---

## 그룹 3: 확장성과 운영 (Factor 7 ~ 9)

### Factor 7 — 포트 바인딩(Port Binding): 앱이 직접 포트를 노출

> "앱은 외부 웹 서버 컨테이너에 의존하지 않고, 직접 포트에 바인딩해 HTTP 서비스를 제공한다."

전통적인 Java 앱은 Tomcat이나 JBoss 같은 외부 WAS(Web Application Server)에 WAR 파일을 올려 배포했습니다. 이 방식은 배포 환경에 외부 컨테이너가 반드시 설치되어 있어야 한다는 의존성을 만듭니다.

12 Factor 앱은 웹 서버를 라이브러리로 취급합니다. 앱 자체에 포함하고, 앱이 직접 포트를 엽니다.

```python
# ✅ Flask가 직접 포트에 바인딩
from flask import Flask
app = Flask(__name__)

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port)  # 앱이 직접 5000번 포트를 엶
```

```javascript
// ✅ Express가 직접 포트에 바인딩
const express = require("express");
const app = express();

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`서버가 포트 ${PORT}에서 실행 중`);
});
```

이렇게 하면 어떤 환경에서도 동일한 방식으로 앱을 실행할 수 있습니다. Tomcat이 설치됐는지, Apache가 있는지 신경 쓸 필요가 없습니다. `docker run -p 80:5000 myapp` 한 줄로 배포 끝입니다.

### Factor 8 — 병행성(Concurrency): 프로세스 모델로 스케일 아웃

> "워크로드 유형별로 프로세스 타입을 나누고, 각 타입을 독립적으로 수평 확장한다."

규모를 키우는 방법에는 두 가지가 있습니다. **스케일 업(Scale Up)**은 서버 자체를 더 강력하게 만드는 것이고, **스케일 아웃(Scale Out)**은 동일한 서버를 더 많이 실행하는 것입니다. 12 Factor는 스케일 아웃을 권장합니다.

앱의 역할을 프로세스 타입으로 분류합니다.

```
web     프로세스: HTTP 요청 처리
worker  프로세스: 백그라운드 작업 (이메일 발송, 이미지 리사이징, 배치 처리)
clock   프로세스: 정기 실행 작업 (스케줄러, 크론 잡)
```

```yaml
# Docker Compose — 프로세스 타입별 정의
services:
  web:
    image: myapp
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    ports:
      - "8000:8000"

  worker:
    image: myapp
    command: celery -A tasks worker --loglevel=info

  clock:
    image: myapp
    command: celery -A tasks beat --loglevel=info
```

```yaml
# Kubernetes — 각 프로세스 타입을 독립적으로 스케일 조정
kubectl scale deployment web    --replicas=10  # 트래픽 급증 → web만 10대로
kubectl scale deployment worker --replicas=3   # 작업 처리 지연 → worker만 3대로
```

트래픽이 늘면 `web` 프로세스만 늘리고, 백그라운드 작업이 밀리면 `worker`만 늘립니다. **각 타입을 필요한 만큼, 필요한 시점에 독립적으로 확장**할 수 있습니다.

### Factor 9 — 폐기 용이성(Disposability): 빠르게 켜고, 안전하게 끄기

> "프로세스는 빠르게 시작하고, SIGTERM 신호를 받으면 진행 중인 요청만 마무리하고 안전하게 종료한다."

Kubernetes나 Docker에서 배포가 일어날 때의 과정을 이해하면 이 원칙이 왜 중요한지 알 수 있습니다.

```
[배포 발생 시 Kubernetes의 행동]

1. 새 버전 Pod 시작 (새 프로세스 켜기)
2. 기존 Pod에 SIGTERM 신호 전송 ("이제 종료해도 됩니다")
3. 기존 Pod는 신규 요청 수신 중단 + 처리 중인 요청 마무리
4. 정상 종료 (process.exit(0))
5. 30초 후에도 살아있으면 강제 종료 (SIGKILL)
```

앱이 SIGTERM을 무시하고 즉시 종료하면 처리 중이던 HTTP 요청이 강제로 끊깁니다. 사용자 입장에서는 갑작스러운 에러가 발생합니다. **Graceful Shutdown(우아한 종료)**은 이것을 방지합니다.

```python
# ✅ Python — Graceful Shutdown 구현
import signal
import sys
from fastapi import FastAPI
import uvicorn

app = FastAPI()
is_shutting_down = False

def handle_sigterm(signum, frame):
    global is_shutting_down
    print("SIGTERM 수신. Graceful Shutdown 시작...")
    is_shutting_down = True
    # 현재 처리 중인 요청이 완료되면 서버가 종료됨

signal.signal(signal.SIGTERM, handle_sigterm)
signal.signal(signal.SIGINT, handle_sigterm)

@app.get("/health")
def health():
    # 종료 중이면 503 반환 → 로드밸런서가 트래픽 라우팅 중단
    if is_shutting_down:
        return Response(status_code=503)
    return {"status": "ok"}
```

```javascript
// ✅ Node.js — Graceful Shutdown 구현
const server = app.listen(3000);

async function shutdown(signal) {
  console.log(`${signal} 수신. Graceful Shutdown 시작...`);

  // 새 연결 수락 중단
  server.close(async () => {
    console.log("HTTP 서버 종료. 리소스 정리 중...");

    // DB 연결, Redis 연결 등 리소스 정리
    await database.close();
    await redisClient.quit();

    console.log("정리 완료. 프로세스 종료.");
    process.exit(0);
  });
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT",  () => shutdown("SIGINT"));
```

> {: .prompt-info }
> **빠른 시작도 중요합니다.** 앱의 시작 시간이 수분씩 걸린다면, 장애 복구가 느려지고 오토스케일링의 반응성이 떨어집니다. 일반적으로 **수 초 이내에 첫 요청을 받을 수 있는 상태**가 되어야 한다고 봅니다.

---

## 그룹 4: 관찰 가능성 (Factor 10 ~ 12)

### Factor 10 — 개발/운영 환경 일치(Dev/Prod Parity): "내 컴퓨터에선 됐는데" 없애기

> "개발, 스테이징, 운영 환경을 최대한 동일하게 유지한다."

환경 차이가 클수록 **환경 특이적 버그(Environment-specific Bug)**가 많아집니다. 개발 환경에서는 SQLite, 운영에서는 PostgreSQL을 쓴다면, SQLite에서는 잘 동작하지만 PostgreSQL에서는 실패하는 쿼리를 개발 단계에서 발견하지 못합니다.

| 구분 | ❌ 환경 불일치 | ✅ 환경 일치 |
| :--- | :--- | :--- |
| 데이터베이스 | 개발: SQLite / 운영: PostgreSQL | 개발/운영 모두 PostgreSQL |
| 캐시 | 개발: 없음 / 운영: Redis | 개발/운영 모두 Redis |
| OS | 개발: macOS / 운영: Linux | Docker로 동일한 환경 |
| 서비스 버전 | 개발: PostgreSQL 14 / 운영: PostgreSQL 16 | 동일 버전 |

**Docker Compose**를 사용하면 이 원칙을 쉽게 지킬 수 있습니다. 개발 환경에서도 운영 환경과 동일한 버전의 DB와 Redis를 컨테이너로 실행합니다.

```yaml
# docker-compose.yml — 개발 환경도 운영과 동일한 서비스 스택
services:
  db:
    image: postgres:16          # 운영과 동일한 PostgreSQL 버전
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_PASSWORD: devpassword

  cache:
    image: redis:7              # 운영과 동일한 Redis 버전

  app:
    build: .
    environment:
      DATABASE_URL: postgresql://postgres:devpassword@db:5432/myapp_dev
      REDIS_URL: redis://cache:6379
    depends_on:
      - db
      - cache
```

```bash
# 개발 환경 전체를 한 번에 실행
docker compose up
# → PostgreSQL 16, Redis 7, 앱 모두 동일한 환경으로 실행됨
```

이전 시리즈에서 다룬 IaC(Vagrant + Ansible)도 이 원칙을 지원하는 도구입니다. 인프라 구성을 코드로 관리하면 환경 간 차이가 코드로 가시화되고, 동일한 구성을 재현할 수 있습니다.

### Factor 11 — 로그(Logs): 파일이 아닌 스트림으로

> "앱은 로그 파일을 직접 관리하지 않는다. 로그를 stdout(표준 출력)으로 내보내고, 수집·저장은 외부 시스템에 맡긴다."

앱이 로그 파일을 직접 관리하면 다음 문제가 생깁니다.

- 서버가 10대라면 로그 파일이 10곳에 흩어짐
- 디스크 용량이 부족하면 앱이 멈출 수 있음
- 로그 시스템을 바꾸려면 코드를 수정해야 함

```python
# ❌ 잘못된 예 — 파일에 직접 로그 쓰기
import logging
logging.basicConfig(filename="/var/log/myapp.log", level=logging.INFO)
logger.info("사용자 로그인: user_id=123")
# → 컨테이너가 재시작되면 로그 파일이 사라짐

# ✅ 올바른 예 — stdout으로 출력
import logging
import sys

logging.basicConfig(
    stream=sys.stdout,          # stdout으로 출력
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)
logger = logging.getLogger(__name__)
logger.info("사용자 로그인: user_id=123")
# → Docker/Kubernetes가 이 출력을 수집해 외부 시스템으로 전달
```

앱은 로그를 **어디에 저장할지 관여하지 않습니다**. stdout으로 출력하면 실행 환경이 알아서 수집합니다.

```
앱 (stdout 출력)
    │
    ▼
[로그 수집 에이전트]      Fluentd, Logstash, Promtail
    │
    ▼
[로그 저장소]             Elasticsearch, AWS CloudWatch, Loki
    │
    ▼
[로그 대시보드]           Kibana, Grafana
```

이 구조에서는 앱 코드를 한 줄도 바꾸지 않고 로그 시스템 전체를 교체할 수 있습니다.

### Factor 12 — 관리 프로세스(Admin Processes): 일회성 작업도 코드로 관리

> "DB 마이그레이션, 데이터 수정 등 일회성 관리 작업도 동일한 코드베이스에서 관리하고, 동일한 환경에서 실행한다."

긴급 상황에서 흔히 발생하는 위험한 패턴이 있습니다.

```bash
# ❌ 위험한 패턴 — 운영 DB에 직접 SQL 실행
ssh admin@prod-server
mysql -u root -p mydb

> UPDATE users SET role='admin' WHERE email='someone@company.com';
# → 누가, 언제, 왜 실행했는지 기록이 없음
# → 실수로 WHERE 절을 빠뜨리면 전체 데이터 손상
# → 롤백 방법이 없음
```

관리 작업도 코드화하면 버전 관리, 코드 리뷰, 실행 이력이 모두 남습니다.

```python
# ✅ Django 마이그레이션 — 코드로 관리되는 DB 변경
# migrations/0005_add_admin_role.py

from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [("users", "0004_user_profile")]

    operations = [
        migrations.AddField(
            model_name="user",
            name="role",
            field=models.CharField(
                max_length=20,
                choices=[("user", "일반 사용자"), ("admin", "관리자")],
                default="user",
            ),
        ),
    ]
```

```bash
# CI/CD 파이프라인 또는 Kubernetes Job으로 실행
python manage.py migrate         # Django
flask db upgrade                 # Flask-Migrate
kubectl exec deploy/myapp -- python manage.py migrate   # Kubernetes
```

마이그레이션 코드는 저장소에 포함되어 코드 리뷰를 거치고, 실행은 파이프라인을 통해 이루어지므로 누가 언제 실행했는지 이력이 남습니다.

---

## 12 Factor 셀프 체크리스트

현재 프로젝트가 얼마나 12 Factor를 따르는지 점검해봅니다. 한 번에 다 갖출 필요는 없습니다. **지금 가장 문제가 되는 항목부터** 하나씩 개선해나가면 됩니다.

```
[코드 관리]
□ 하나의 앱이 하나의 Git 저장소에서 관리된다
□ 모든 의존성이 requirements.txt / package.json 등에 버전과 함께 명시되어 있다
□ DB 비밀번호, API 키, 연결 주소가 코드에 하드코딩되어 있지 않다
□ 코드를 GitHub에 공개해도 보안에 문제가 없다

[실행 환경]
□ 환경 변수 하나만 바꿔서 DB 또는 외부 서비스를 교체할 수 있다
□ 운영 서버에서 git pull 후 직접 실행하지 않는다 (Build/Release/Run 분리)
□ 서버 메모리에 세션을 저장하지 않는다 (Redis 등 외부 저장소 사용)
□ 로컬 파일 시스템에 업로드 파일을 저장하지 않는다

[확장성과 운영]
□ 앱이 직접 포트에 바인딩한다 (Apache/Tomcat 불필요)
□ 웹 요청과 백그라운드 작업이 분리된 프로세스 타입으로 실행된다
□ 앱이 수 초 내에 시작된다
□ SIGTERM 신호를 받으면 처리 중인 요청을 마무리하고 안전하게 종료된다

[관찰 가능성]
□ 개발 환경의 DB 종류와 버전이 운영 환경과 동일하다
□ 로그를 파일이 아닌 stdout으로 출력한다
□ DB 마이그레이션 코드가 저장소에 포함되어 버전 관리된다
□ 일회성 관리 작업이 파이프라인 또는 코드로 실행된다
```

---

## 12 Factor와 현대 기술 스택의 관계

12 Factor는 2011년에 작성됐지만, 오히려 현대 기술 스택과 **완벽하게 맞아떨어집니다.** Kubernetes와 Docker가 이 원칙들을 구현하는 방향으로 설계되어 있기 때문입니다.

| 12 Factor 원칙 | Docker/Kubernetes의 구현 |
| :--- | :--- |
| **의존성 격리** | Dockerfile로 의존성 격리, 이미지 자급자족 |
| **설정 분리** | ConfigMap, Secret, 환경 변수로 설정 주입 |
| **Build/Release/Run 분리** | CI/CD 파이프라인 + 이미지 레지스트리 |
| **Stateless 프로세스** | Pod는 언제든 재시작 가능하도록 설계됨 |
| **포트 바인딩** | Container Port → Service로 노출 |
| **스케일 아웃** | `kubectl scale --replicas=N` |
| **폐기 용이성** | Rolling Update, Graceful Shutdown, SIGTERM |
| **로그 스트림** | Pod의 stdout을 자동 수집 |

12 Factor를 따르는 앱은 **Kubernetes 위에서 자연스럽게 잘 동작**합니다. 반대로 이 원칙을 위반한 앱은 컨테이너화하더라도 운영 중에 예기치 않은 문제를 일으킵니다.

---

## 정리

| Factor | 핵심 원칙 | 위반 시 증상 |
| :--- | :--- | :--- |
| **1. 코드베이스** | 하나의 앱, 하나의 저장소 | 환경마다 코드가 달라짐 |
| **2. 의존성** | Manifest 파일에 명시 | "내 서버엔 설치됐는데…" |
| **3. 설정** | 환경 변수로 분리 | 비밀번호 GitHub 유출, 재빌드 필요 |
| **4. 백엔드 서비스** | 교체 가능한 부착 자원 | DB 교체 시 코드 수정 필요 |
| **5. Build/Release/Run** | 3단계 엄격히 분리 | 롤백 불가, 배포 이력 없음 |
| **6. 프로세스** | Stateless | 서버 증설 시 세션 날아감 |
| **7. 포트 바인딩** | 앱이 직접 포트 노출 | 외부 WAS 의존성 발생 |
| **8. 병행성** | 프로세스 타입별 스케일 | 불필요한 자원 낭비 |
| **9. 폐기 용이성** | 빠른 시작·안전한 종료 | 배포 시 요청 강제 종료 |
| **10. 개발/운영 일치** | 환경 차이 최소화 | "내 PC에서는 됐는데…" |
| **11. 로그** | stdout으로 출력 | 로그 파일 관리 지옥 |
| **12. 관리 프로세스** | 관리 작업도 코드화 | 직접 DB 접속, 이력 없음 |


좋은 파이프라인이 있어도, 앱이 Stateful하거나 설정이 하드코딩되어 있으면 배포는 여전히 불안합니다. 12 Factor는 파이프라인이 제대로 동작하기 위한 **앱 설계의 전제 조건**입니다.
