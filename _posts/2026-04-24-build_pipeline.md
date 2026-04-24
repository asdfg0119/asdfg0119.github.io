---
layout: post
title: "Build Pipeline — 반복 작업을 파이프라인으로 자동화하기"
date: 2026-04-24 19:00:00 +0900
categories:
  - DevOps
  - CI/CD
tags: [BuildPipeline, CICD, GitLabCI, Jenkins, GitHubActions]
math: false
---

> {: .prompt-tip }
> **요약:** Build Pipeline은 코드 푸시 한 번으로 빌드 → 테스트 → 배포 전 과정을 자동으로 실행하는 자동화 파이프라인입니다. 이 글에서는 파이프라인이 무엇인지, 왜 필요한지, 어떤 도구를 선택해야 하는지, 그리고 GitLab CI/CD를 중심으로 실제 구성 방법까지 개념 중심으로 정리합니다.

---

이전 피드에서 Ansible과 Serverspec으로 서버 설정과 검증을 코드로 만들었습니다. 이제 더 이상 서버 설정 방법을 문서에 적어두지 않아도 됩니다. 코드가 곧 설명서입니다.

그런데 여전히 이런 반복 작업이 남아 있습니다.

```
1. 서버에 SSH 로그인
2. ansible-playbook -i development site.yml 실행
3. 결과 확인
4. rake spec 실행
5. 결과 확인
6. (문제없으면) 운영 서버에 배포
```

코드를 바꿀 때마다 이 과정을 반복합니다. 팀원이 여럿이라면 누가 마지막으로 실행했는지, 결과가 무엇이었는지 아무도 알 수 없습니다. 이 문제를 해결하는 것이 **Build Pipeline**입니다.

---

## 1. Build Pipeline이 해결하는 세 가지 문제

### 문제 1: 명령어 반복 작업

> "서버에 로그인해서 같은 명령어를 매번 입력하는 게 너무 귀찮다."

Build Pipeline은 **GUI(그래픽 인터페이스)** 또는 코드 파일 하나로 전체 자동화 흐름을 정의합니다. 이후에는 코드를 푸시하는 것만으로 파이프라인이 알아서 실행됩니다. 개발자가 직접 서버에 로그인할 필요가 없습니다.

### 문제 2: 사람의 실수

> "절차 중 한 단계를 깜빡했다. 테스트를 건너뛰었다."

수작업이 개입하면 실수가 생깁니다.

- 파라미터를 잘못 입력한다
- 절차 중 한 단계를 빠뜨린다
- 귀찮아서 테스트를 건너뛴다
- 잘 되겠지 하고 운영 배포를 눌러버린다

Build Pipeline은 정해진 순서를 **강제로** 실행합니다. 테스트를 건너뛸 수 없고, 이전 단계가 실패하면 다음 단계로 넘어가지 않습니다. 사람이 개입할 여지 자체를 없앱니다.

### 문제 3: 실행 이력 없음

> "지난주 배포 때 무엇이 바뀌었는지 아무도 모른다."

Git은 코드 변경 이력을 남기지만, "언제 어떤 명령어가 실행됐는지"는 기록하지 않습니다. Build Pipeline 도구는 **모든 실행 이력을 저장**합니다. 장애 발생 시 "마지막 배포가 언제였고, 무엇이 바뀌었는지"를 즉시 확인할 수 있습니다.

---

## 2. 파이프라인이란? — 개념부터 이해하기

### 파이프(Pipe)에서 온 이름

파이프라인이라는 이름은 Unix 쉘의 파이프 연산자(`|`)에서 유래합니다.

```bash
# Unix 파이프: 앞 명령의 출력이 뒤 명령의 입력이 된다
cat access.log | grep "ERROR" | sort | uniq -c
```

각 명령어가 앞 단계의 결과를 받아 처리하고 다음 단계로 넘겨줍니다. Build Pipeline도 동일한 구조입니다.

```
[코드 체크아웃] ──▶ [빌드] ──▶ [테스트] ──▶ [스테이징 배포] ──▶ [운영 배포]
```

각 단계는 이전 단계가 **성공해야만** 다음 단계로 진행됩니다. 어느 단계에서든 실패하면 파이프라인이 즉시 멈추고 담당자에게 알림을 보냅니다.

### 파이프라인의 세 가지 핵심 성질

**성질 1: 순차 실행 보장 (Sequential Guarantee)**

파이프라인은 단계의 순서를 보장합니다. 테스트가 끝나기 전에 배포가 실행되는 일은 없습니다. 이 순서는 코드 파일에 명시되어 있고, 아무도 이를 임의로 바꾸거나 건너뛸 수 없습니다.

**성질 2: 실패 시 즉각 중단 (Fail Fast)**

파이프라인의 중요한 철학 중 하나는 **"빨리 실패하라(Fail Fast)"**입니다. 문제를 빨리 발견할수록 수정 비용이 적습니다. 빌드 단계에서 실패하면 테스트와 배포는 실행조차 되지 않아, 잘못된 코드가 운영 환경에 도달하는 것을 원천 차단합니다.

```
[빌드] ──▶ ❌ 실패 → 즉시 중단, 알림 발송
            ↓ (이 이후는 실행되지 않음)
[테스트] (건너뜀)
[배포]   (건너뜀)
```

**성질 3: 이력과 재현성 (Auditability)**

모든 파이프라인 실행은 기록됩니다. 각 실행에 대해 다음 정보가 저장됩니다.

- 언제 실행됐는가 (타임스탬프)
- 누가 트리거했는가 (커밋한 개발자)
- 어떤 코드가 사용됐는가 (커밋 해시)
- 각 단계가 성공했는가, 실패했는가
- 실패했다면 로그가 무엇인가

이 이력이 있으면 장애 원인 분석이 훨씬 빨라집니다.

### CI/CD 파이프라인의 표준 구조

현대적인 CI/CD 파이프라인은 보통 다음 단계로 구성됩니다.

```
커밋/PR 발생
     │
     ▼
┌─────────────────────────────────────────┐
│  CI (Continuous Integration)            │
│                                         │
│  [소스 체크아웃]                          │
│       ↓                                 │
│  [의존성 설치]                            │
│       ↓                                 │
│  [정적 분석 / 린팅]  ← 코드 스타일 검사     │
│       ↓                                 │
│  [빌드 / 컴파일]                          │
│       ↓                                 │
│  [단위 테스트]       ← 함수·모듈 단위       │
│       ↓                                 │
│  [통합 테스트]       ← 모듈 간 연동        │
│       ↓                                 │
│  [아티팩트 생성]     ← Docker 이미지 등    │
└─────────────────────────────────────────┘
     │ 성공 시
     ▼
┌─────────────────────────────────────────┐
│  CD (Continuous Delivery/Deployment)    │
│                                         │
│  [스테이징 배포]                          │
│       ↓                                 │
│  [E2E 테스트 / 스모크 테스트]              │
│       ↓                                 │
│  [운영 배포] ← main 브랜치 또는 수동 승인   │
└─────────────────────────────────────────┘
```

> {: .prompt-info }
> **CI와 CD의 차이:** CI(Continuous Integration)는 코드를 통합하고 검증하는 과정, CD(Continuous Delivery)는 검증된 코드를 언제든 배포 가능한 상태로 만드는 과정입니다. Continuous Deployment는 한 발 더 나아가 운영 배포까지 자동화합니다.

---

## 3. 파이프라인 도구 선택하기

2026년 현재 CI/CD 도구 시장은 크게 세 가지 플레이어가 지배하고 있습니다. 각 도구의 철학과 적합한 상황이 다르기 때문에, "무엇이 제일 좋다"는 답은 없습니다. 팀의 상황에 맞게 선택하는 것이 중요합니다.

### 도구 비교표

| 도구 | 형태 | 파이프라인 정의 방식 | 2026 시장 점유 |
| :--- | :--- | :--- | :--- |
| **Jenkins** | 설치형(온프레미스) | Jenkinsfile (Groovy DSL) | Fortune 500의 80% |
| **GitLab CI** | 클라우드/설치형 | `.gitlab-ci.yml` (YAML) | 엔터프라이즈 YoY +34% |
| **GitHub Actions** | 클라우드 | `.github/workflows/*.yml` (YAML) | 오픈소스 68% 점유 |
| **CircleCI** | 클라우드 | `.circleci/config.yml` (YAML) | 빠른 빌드 특화 |

### Jenkins

2004년에 등장한 가장 오래된 CI/CD 도구입니다. 오픈소스이며, 현재도 대기업 인프라의 핵심으로 자리 잡고 있습니다.

**어떻게 동작하는가?**

Jenkins는 **컨트롤러-에이전트(Controller-Agent)** 아키텍처를 사용합니다. 중앙의 컨트롤러(마스터)가 작업을 스케줄링하고 관리하며, 에이전트(워커)가 실제 빌드와 테스트를 실행합니다.

```
[Jenkins Controller]
  - 파이프라인 스케줄링
  - 웹 UI 제공
  - 플러그인 관리
        │
        ├── [Agent 1] → Java 빌드 전용
        ├── [Agent 2] → Python 테스트 전용
        └── [Agent 3] → Docker 빌드 전용
```

파이프라인은 **Jenkinsfile**이라는 파일에 Groovy 언어로 작성합니다.

```groovy
// Jenkinsfile
pipeline {
    agent any  // 어떤 에이전트에서든 실행

    stages {
        stage('Build') {
            steps {
                sh 'ansible-playbook -i development site.yml'
            }
        }
        stage('Test') {
            steps {
                sh 'cd /serverspec && rake spec'
            }
        }
        stage('Deploy') {
            when {
                branch 'main'  // main 브랜치일 때만 실행
            }
            steps {
                sh 'ansible-playbook -i production site.yml'
            }
        }
    }

    post {
        failure {
            // 실패 시 Slack 알림
            slackSend channel: '#alerts',
                      message: "빌드 실패: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        success {
            slackSend channel: '#deploys',
                      message: "배포 완료: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

**장점:**
- 1,800개 이상의 플러그인으로 거의 모든 시스템과 연동 가능
- 사내 네트워크에 완전히 격리된 환경에서 운영 가능 (에어갭 환경)
- 복잡한 멀티 브랜치, 멀티 스테이지 파이프라인에 강함

**단점:**
- 직접 서버에 설치하고 관리해야 함 (운영 비용 발생)
- Groovy DSL의 학습 곡선이 있음
- 2025년 기준 1,800개 플러그인에서 127개의 CVE(보안 취약점)가 발견됨

> {: .prompt-warning }
> **Jenkins를 선택해야 하는 상황:** 코드를 외부 서버에 올릴 수 없는 보안 규정이 있거나, 이미 Jenkins 파이프라인이 수백 개 구성된 레거시 환경이라면 Jenkins가 현실적인 선택입니다.

### GitHub Actions

2018년 GitHub이 출시한 클라우드 네이티브 CI/CD 플랫폼입니다. 현재 GitHub 오픈소스 프로젝트의 68%가 Actions를 사용합니다.

**어떻게 동작하는가?**

저장소의 `.github/workflows/` 디렉터리에 YAML 파일을 올리면 자동으로 파이프라인이 활성화됩니다. 파이프라인을 실행하는 **Runner**는 GitHub이 클라우드에서 자동으로 제공합니다.

파이프라인의 핵심 개념은 **Action**입니다. Action은 재사용 가능한 자동화 단위로, GitHub Marketplace에서 수만 개의 공개 Action을 그대로 가져다 쓸 수 있습니다.

```yaml
# .github/workflows/ci.yml

name: CI/CD Pipeline

# 어떤 이벤트에 반응할지
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest  # GitHub이 제공하는 Runner

    steps:
      # 1. 코드 체크아웃 (Marketplace Action 사용)
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. Python 환경 설정 (Marketplace Action 사용)
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      # 3. 의존성 설치
      - name: Install dependencies
        run: pip install -r requirements.txt

      # 4. 린팅
      - name: Lint
        run: ruff check .

      # 5. 테스트
      - name: Run tests
        run: pytest tests/ --cov=src --cov-fail-under=80

  deploy:
    needs: build-and-test  # 위 job이 성공해야 실행
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'  # main 브랜치일 때만

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to production
        run: ansible-playbook -i production site.yml
        env:
          ANSIBLE_SSH_KEY: ${{ secrets.ANSIBLE_SSH_KEY }}
          # secrets는 GitHub 저장소 설정에서 암호화하여 저장
```

**장점:**
- 별도 서버 설치 없이 바로 사용 가능 (제로 설정)
- GitHub 저장소와 완벽 통합 (PR, 커밋 상태 표시 등)
- Marketplace의 수만 개 Action 재활용 가능
- 공개 저장소 무료, 월 2,000분 무료 제공

**단점:**
- GitHub에 코드가 있어야만 사용 가능 (플랫폼 종속)
- 복잡한 파이프라인에서는 YAML 파일이 매우 길어짐
- 대규모 팀에서는 Runner 분당 비용이 빠르게 증가

### GitLab CI/CD

GitLab이 초기부터 CI/CD를 플랫폼의 핵심으로 설계한 도구입니다. 코드 저장소, 이슈 트래커, CI/CD, 보안 스캔, 컨테이너 레지스트리가 모두 하나의 플랫폼에 통합되어 있습니다.

**어떻게 동작하는가?**

프로젝트 루트의 `.gitlab-ci.yml` 파일이 파이프라인을 정의합니다. 실제 실행은 **GitLab Runner**가 담당합니다. Runner는 GitLab SaaS가 제공하는 공유 Runner를 사용할 수도 있고, 직접 서버에 설치한 전용 Runner를 사용할 수도 있습니다.

**핵심 개념:**

| 개념 | 설명 | 비유 |
| :--- | :--- | :--- |
| **Pipeline** | 전체 자동화 흐름 | 공장의 전체 생산 라인 |
| **Stage** | 파이프라인을 나누는 단계 | 조립 → 검사 → 포장 |
| **Job** | 각 Stage에서 실행하는 구체적 작업 | 나사 조이기, 페인트 칠하기 |
| **Runner** | Job을 실제로 실행하는 에이전트 | 작업자(사람 또는 로봇) |
| **Artifact** | Job이 생성한 결과물 | 테스트 리포트, Docker 이미지 |

같은 Stage의 Job은 **병렬로 실행**되고, Stage는 **순서대로** 실행됩니다.

```
Stage:   build          test                    deploy
         ┌────────┐    ┌────────┐ ┌──────────┐  ┌────────┐
Job:     │ build  │ →  │ unit   │ │ lint     │→ │ deploy │
         └────────┘    │ test   │ │          │  └────────┘
                       └────────┘ └──────────┘
                       (병렬 실행)
```

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

# ── Stage: build ──────────────────────────────────────────
build-job:
  stage: build
  image: ubuntu:22.04        # Docker 이미지 지정
  script:
    - apt-get update -q
    - apt-get install -y ansible
    - ansible-playbook -i inventory/development site.yml
  artifacts:
    paths:
      - build/               # 다음 Stage에서 사용할 결과물 보존
    expire_in: 1 hour

# ── Stage: test (2개 Job이 병렬 실행됨) ───────────────────
unit-test-job:
  stage: test
  image: ruby:3.1
  script:
    - gem install serverspec
    - cd serverspec && rake spec
  artifacts:
    reports:
      junit: serverspec/reports/rspec.xml   # 테스트 리포트 자동 파싱

lint-job:
  stage: test
  image: python:3.11
  script:
    - pip install ruff
    - ruff check .

# ── Stage: deploy ─────────────────────────────────────────
deploy-staging:
  stage: deploy
  script:
    - ansible-playbook -i inventory/staging site.yml
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop     # develop 브랜치에서만 스테이징 배포

deploy-production:
  stage: deploy
  script:
    - ansible-playbook -i inventory/production site.yml
  environment:
    name: production
    url: https://example.com
  when: manual   # 수동 승인 후에만 운영 배포 실행
  only:
    - main        # main 브랜치에서만
```

**장점:**
- 설정 파일(`.gitlab-ci.yml`)이 코드 저장소 안에 있어 파이프라인도 버전 관리됨
- 보안 스캔(SAST, DAST), 의존성 검사가 파이프라인에 기본 내장
- 클라우드와 설치형 모두 지원 → 보안 규정이 있는 환경에서도 사용 가능
- `when: manual`로 수동 승인 게이트를 쉽게 설정 가능

**단점:**
- GitLab에 코드가 있어야만 사용 가능 (플랫폼 종속)
- 기능이 많은 만큼 처음 익히는 학습 비용이 있음

### 도구 선택 결정 트리

```
코드 저장소가 어디에 있나?
├── GitHub → GitHub Actions (가장 자연스러운 선택)
├── GitLab → GitLab CI/CD
└── 사내 Git (Bitbucket, 자체 설치) ─────────────────┐
                                                      │
                                    외부 서버 사용 가능?
                                    ├── Yes → CircleCI / GitLab CI
                                    └── No  → Jenkins (설치형)
```

> {: .prompt-tip }
> **2026년 신규 프로젝트라면?** 코드가 GitHub에 있다면 GitHub Actions, GitLab에 있다면 GitLab CI/CD를 선택하는 것이 가장 빠르고 자연스럽습니다. Jenkins는 레거시 인프라와 연동이 필요하거나 에어갭 환경일 때 선택합니다.

---

## 4. 실습: GitLab Runner + Ansible 환경 구성

2편에서 만든 Ansible + Serverspec 파이프라인을 GitLab CI/CD와 연결합니다.

### 전체 디렉터리 구조

```
project/
├── Vagrantfile                  # VM 환경 정의
├── .gitlab-ci.yml               # CI/CD 파이프라인 정의
├── playbook/
│   ├── inventory/
│   │   ├── development          # 개발 인벤토리
│   │   ├── staging              # 스테이징 인벤토리
│   │   └── production           # 운영 인벤토리
│   ├── site.yml                 # 메인 플레이북
│   └── roles/
│       ├── common/              # 공통 설정
│       ├── nginx/               # nginx 설치 및 설정
│       ├── docker/              # Docker 설치 (Runner 전용)
│       ├── gitlab-runner/       # GitLab Runner 설치
│       └── serverspec/          # Serverspec 설치
└── serverspec/
    ├── Rakefile
    └── spec/
        └── localhost/
            └── nginx_spec.rb
```

### site.yml — Runner 서버 프로비저닝

```yaml
# playbook/site.yml
---
# Runner 서버: Docker와 GitLab Runner 설치
- hosts: runners
  become: yes
  connection: local
  roles:
    - common           # 기본 패키지, 보안 설정
    - docker           # Runner가 Docker 위에서 동작하므로 먼저 설치
    - gitlab-runner    # GitLab Runner 설치 및 등록
    - serverspec       # 인프라 테스트 도구 설치

# 웹 서버: nginx 설치
- hosts: webservers
  become: yes
  roles:
    - common
    - nginx
```

### Docker 설치 Role

```yaml
# playbook/roles/docker/tasks/main.yml
---
- name: Docker GPG 키 추가
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Docker 공식 저장소 추가
  apt_repository:
    repo: deb https://download.docker.com/linux/ubuntu focal stable
    state: present

- name: Docker 패키지 설치
  apt:
    name: "{{ item }}"   # {{ item }}은 with_items의 각 요소를 순서대로 참조
    state: latest
    update_cache: yes
  with_items:
    - docker-ce
    - docker-ce-cli
    - docker-compose-plugin
    - containerd.io

- name: docker 서비스 시작 및 자동 시작 등록
  service:
    name: docker
    state: started
    enabled: yes
```

> {: .prompt-info }
> **`{{ item }}`과 `with_items`:** `with_items`는 Ansible의 반복(loop) 문법입니다. 리스트의 각 요소를 순서대로 `{{ item }}`에 대입해 같은 Task를 반복 실행합니다. 패키지를 여러 개 설치할 때 자주 씁니다.

### GitLab Runner 등록 Role

```yaml
# playbook/roles/gitlab-runner/tasks/main.yml
---
- name: GitLab Runner 공식 저장소 추가
  shell: >
    curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash
  args:
    creates: /etc/apt/sources.list.d/runner_gitlab-runner.list
    # creates: 이 파일이 이미 존재하면 이 Task를 건너뜀 (멱등성 확보)

- name: GitLab Runner 설치
  apt:
    name: gitlab-runner
    state: present
    update_cache: yes

- name: GitLab Runner Docker executor로 등록
  command: >
    gitlab-runner register
    --non-interactive
    --url "{{ gitlab_url }}"
    --registration-token "{{ gitlab_token }}"
    --executor "docker"
    --docker-image "ubuntu:22.04"
    --description "ansible-runner"
  # gitlab_url, gitlab_token은 Ansible Vault로 암호화된 변수
```

### .gitlab-ci.yml — 완성된 파이프라인

```yaml
# .gitlab-ci.yml

stages:
  - provision   # 서버 환경 구성
  - test        # 인프라 검증
  - deploy      # 운영 배포

variables:
  ANSIBLE_HOST_KEY_CHECKING: "False"   # 첫 SSH 접속 시 호스트 키 확인 비활성화

# ── Stage 1: 서버 환경 구성 ───────────────────────────────
provision-job:
  stage: provision
  image: cytopia/ansible:latest   # Ansible이 설치된 Docker 이미지
  script:
    - echo "인벤토리 확인..."
    - ansible -i playbook/inventory/development all -m ping
    - echo "플레이북 실행..."
    - ansible-playbook -i playbook/inventory/development playbook/site.yml --diff
  only:
    - develop
    - main

# ── Stage 2: 인프라 상태 검증 ─────────────────────────────
serverspec-job:
  stage: test
  image: ruby:3.2
  script:
    - cd serverspec
    - gem install serverspec rake
    - rake spec
  artifacts:
    reports:
      junit: serverspec/reports/rspec.xml
    when: always   # 실패해도 리포트는 항상 보존
  needs: ["provision-job"]   # provision-job이 끝난 후에 실행

# ── Stage 3: 운영 배포 (main 브랜치 + 수동 승인) ──────────
deploy-production:
  stage: deploy
  image: cytopia/ansible:latest
  script:
    - ansible-playbook -i playbook/inventory/production playbook/site.yml
  environment:
    name: production
    url: https://example.com
  when: manual     # 담당자가 GitLab UI에서 직접 버튼을 눌러야 실행됨
  only:
    - main
  needs: ["serverspec-job"]
```

### 전체 실행 흐름

```
개발자가 develop 브랜치에 코드 푸시
           │
           ▼
GitLab이 .gitlab-ci.yml 감지 → 파이프라인 자동 시작
           │
           ▼
┌─────────────────────────────────────────────────┐
│  Stage 1: provision                             │
│  ansible-playbook -i development site.yml       │
│  → nginx 설치, 서비스 시작, 설정 파일 배포          │
└─────────────────────────────────────────────────┘
           │ 실패 → 알림 발송, 파이프라인 중단
           │ 성공
           ▼
┌─────────────────────────────────────────────────┐
│  Stage 2: test                                  │
│  rake spec                                      │
│  → 포트 80 열림 ✓, nginx 실행 중 ✓                │
└─────────────────────────────────────────────────┘
           │ 실패 → 알림 발송, 파이프라인 중단
           │ 성공
           ▼
┌─────────────────────────────────────────────────┐
│  Stage 3: deploy (⏸ 수동 승인 대기)              │
│  GitLab UI에서 담당자가 버튼을 눌러야 실행됨         │
│  ansible-playbook -i production site.yml        │
└─────────────────────────────────────────────────┘
           │
           ▼
    배포 완료 → Slack/이메일 알림

이 전 과정에서 개발자가 할 일: 코드 푸시 한 번
```

---

## 5. 파이프라인 실행 이력: 장애 대응의 핵심

Build Pipeline 도구는 모든 실행 이력을 저장합니다. GitLab 파이프라인 페이지에서 이력을 보면 다음과 같은 형태입니다.

```
Pipeline #1247   ✅ passed   2026-04-24 14:32   main ← feat/payment (커밋 a3f9c12)
Pipeline #1246   ❌ failed   2026-04-24 11:15   main (커밋 b8e2d47) — test 단계 실패
Pipeline #1245   ✅ passed   2026-04-23 17:08   main ← feat/user-auth (커밋 c1a4f78)
Pipeline #1244   ✅ passed   2026-04-23 09:40   main ← fix/db-connection (커밋 d5b3e90)
```

장애가 발생했을 때 대응 순서는 명확해집니다.

```
1. "마지막으로 성공한 배포는 언제?"
   → Pipeline #1245 (2026-04-23 17:08)

2. "그 이후 무엇이 바뀌었나?"
   → 커밋 a3f9c12 확인: payment 모듈 변경

3. 원인 파악 → 롤백 또는 핫픽스 결정
   → 이전 배포(#1245)로 즉시 롤백 가능
```

이력이 없으면 이 분석 자체가 불가능합니다.

---

## 6. 파이프라인을 더 잘 만드는 법: 알아두면 좋은 개념들

파이프라인을 처음 구성할 때는 기본적인 build → test → deploy 구조로 시작합니다. 팀이 성숙해지면서 다음 개념들을 점진적으로 적용해나갑니다.

### Shift-Left Security — 보안 검사를 앞당긴다

"Shift-Left"는 보안 검사를 파이프라인의 **왼쪽(앞쪽) 단계로 옮기는** 것을 의미합니다. 운영 배포 직전에 발견한 보안 취약점보다 커밋 직후에 발견한 취약점이 훨씬 저렴하게 수정됩니다.

```yaml
security-scan:
  stage: test   # test 단계에서 보안 검사 실행 (배포 전)
  script:
    - pip install bandit
    - bandit -r src/   # Python 코드의 보안 취약점 정적 분석
```

### 파이프라인 병렬화 — 속도를 높인다

같은 Stage의 Job은 병렬로 실행됩니다. 독립적인 테스트를 여러 Job으로 나누면 전체 파이프라인 시간을 단축할 수 있습니다.

```yaml
# 3개의 테스트 Job이 동시에 실행됨 (병렬)
unit-tests:
  stage: test
  script: pytest tests/unit/

integration-tests:
  stage: test
  script: pytest tests/integration/

security-scan:
  stage: test
  script: bandit -r src/
```

### 브랜치 전략과 파이프라인 — 언제 무엇을 실행할까

모든 브랜치에서 모든 파이프라인을 실행할 필요는 없습니다. 브랜치 전략과 파이프라인을 연계하면 자원을 절약하면서 안전한 배포가 가능합니다.

```yaml
deploy-staging:
  only:
    - develop    # develop 브랜치 → 스테이징 자동 배포

deploy-production:
  only:
    - main       # main 브랜치 → 운영 배포 (수동 승인)
  when: manual
```

```
feature/* 브랜치 → [CI만 실행] 빌드 + 테스트
develop 브랜치   → [CI + CD] 빌드 + 테스트 + 스테이징 자동 배포
main 브랜치      → [CI + CD] 빌드 + 테스트 + 운영 배포(수동 승인)
```

---

## 7. 정리

| 개념 | 핵심 내용 |
| :--- | :--- |
| **Build Pipeline** | 빌드 → 테스트 → 배포를 자동으로 연결하는 파이프라인 |
| **Fail Fast** | 문제를 빨리 발견할수록 수정 비용이 적다 |
| **Jenkins** | 설치형, 1,800+ 플러그인, 에어갭 환경 필수일 때 |
| **GitHub Actions** | GitHub 저장소 연동, 클라우드, 제로 설정 |
| **GitLab CI/CD** | 올인원 DevOps 플랫폼, Stage/Job/Runner 구조 |
| **`.gitlab-ci.yml`** | 저장소 안에 파이프라인 정의 → 파이프라인도 버전 관리 |
| **실행 이력** | 장애 발생 시 원인 분석과 롤백의 핵심 |

파이프라인의 본질은 **반복을 자동화**하고, **실수를 구조적으로 제거**하는 것입니다. "코드 푸시 한 번으로 끝"이라는 단순함 뒤에는, 정의된 순서를 반드시 따르고, 어떤 단계도 건너뛰지 않으며, 모든 실행을 기록하는 엄격한 시스템이 있습니다. 그 시스템이 바로 Build Pipeline입니다.
