---
layout: post
title: "IaC — 인프라도 코드로 관리한다 (Vagrant · Ansible · Serverspec)"
date: 2026-04-23 12:00:00 +0900
categories:
  - DevOps
  - IaC
tags: [IaC, Vagrant, Ansible, Serverspec, DevOps]
math: false
---

> {: .prompt-tip }
> **요약:** IaC(Infrastructure as Code)는 서버 설정을 **문서가 아닌 실행 가능한 코드**로 관리하는 방법론입니다. Vagrant로 VM 환경을 코드화하고, Ansible로 서버 구성을 선언적으로 자동화하며, Serverspec으로 그 결과를 자동 검증하는 세 단계를 코드 예시와 함께 정리합니다.

---

새로운 팀원이 입사했습니다. 개발 환경을 세팅하는 데 이틀이 걸렸고, 그 과정에서 오래된 위키 문서 세 개와 Slack DM 수십 개가 오갔습니다. 운영 서버에 장애가 났는데, 담당자가 퇴사해 어떤 설정이 어디에 있는지 아무도 모릅니다. 보안 패치를 적용해야 하는데, 서버마다 설정이 조금씩 달라 한 번에 적용할 수가 없습니다.

이 문제들의 공통 원인은 하나입니다. **인프라를 코드가 아닌 사람의 기억과 문서에 의존**하고 있다는 것입니다. IaC는 이 문제를 해결하기 위해 등장했습니다.

---

## 1. IaC란 무엇인가?

**IaC(Infrastructure as Code)**는 서버, 네트워크, 데이터베이스 등 인프라 구성을 사람이 읽고 실행할 수 있는 코드 파일로 정의하고 관리하는 방법론입니다. 코드는 그 자체가 실행 가능한 설명서이기 때문에, 파일 하나만 공유하면 누구나 동일한 환경을 재현할 수 있습니다.

### IaC 이전 vs 이후

| 구분 | IaC 이전 | IaC 이후 |
| :--- | :--- | :--- |
| **환경 구축** | 문서 보며 수동 설치 (시간 소요, 실수 빈번) | 명령어 한 줄 실행, 자동 완료 |
| **환경 공유** | "내 PC에서는 됐는데..." | 누구나 동일한 환경 보장 |
| **환경 파악** | 담당자에게 물어봐야 함 | 코드를 읽으면 즉시 파악 |
| **팀 유지보수** | 담당자 퇴사 시 지식 소멸 | 코드로 누구나 재현 가능 |
| **변경 이력** | 없음 (언제, 누가, 왜 바꿨는지 불투명) | Git으로 모든 변경 추적 |
| **환경 차이(Drift)** | 서버마다 설정이 조금씩 달라짐 | 코드 기준으로 항상 동일 상태 유지 |

### IaC를 구성하는 세 가지 레이어

IaC는 보통 다음 세 단계로 나뉩니다. 이 글에서 다루는 도구가 각 단계를 담당합니다.

```
[1단계] 환경 코드화 ──── Vagrant
         VM을 어떻게 생성할지 정의

[2단계] 구성 코드화 ──── Ansible
         VM 안에서 무엇을 설치하고 설정할지 정의

[3단계] 검증 코드화 ──── Serverspec
         설정이 올바르게 적용됐는지 자동 테스트
```

---

## 2. Vagrant — VM 환경을 코드로

### Vagrant란?

**Vagrant**는 가상 머신(VM) 환경 구축을 코드로 정의하는 도구입니다. `Vagrantfile`이라는 Ruby 파일 하나에 "어떤 OS로 어떤 환경을 만들지"를 기술하면, `vagrant up` 명령어 하나로 VM이 자동으로 생성되고 설정까지 완료됩니다.

> {: .prompt-info }
> **참고:** 2026년 현재 Vagrant는 컨테이너(Docker)와 클라우드 개발 환경(GitHub Codespaces, Gitpod)에 상당 부분 자리를 내줬습니다. 그러나 네트워크 토폴로지 테스트, 멀티 VM 환경 재현, OS 수준의 완전한 격리가 필요한 상황에서는 여전히 유효한 도구입니다. 또한 IaC의 핵심 개념인 "환경을 코드로 정의한다"는 사상을 가장 직관적으로 이해하는 데 최적의 출발점입니다.

### 기본 Vagrantfile 구조

```ruby
# Vagrantfile

Vagrant.configure("2") do |config|

  # 1. 어떤 OS 이미지를 사용할지
  config.vm.box = "ubuntu/focal64"

  # 2. 호스트 ↔ 게스트 VM 간 포트 포워딩
  #    호스트의 8080 → VM의 80 (nginx 접근용)
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # 3. 공유 폴더 설정
  #    호스트의 ./app 디렉터리를 VM의 /var/www/html 로 마운트
  config.vm.synced_folder "./app", "/var/www/html"

  # 4. VM 리소스 설정 (VirtualBox 기준)
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"   # RAM 1GB
    vb.cpus = 2
  end

  # 5. VM 생성 후 자동으로 실행할 초기 설정 스크립트
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
  SHELL

end
```

### 핵심 명령어

```bash
# VM 생성 + 프로비저닝 (설정 스크립트 자동 실행)
vagrant up

# VM에 SSH 접속
vagrant ssh

# 프로비저닝만 다시 실행 (VM은 유지)
vagrant provision

# VM 일시 정지 / 재개
vagrant suspend
vagrant resume

# VM 완전 삭제
vagrant destroy

# VM 재생성 (destroy + up 한 번에)
vagrant destroy -f && vagrant up
```

이 파일 하나를 Git에 올리면 팀원 누구든 `git clone` 후 `vagrant up` 한 번으로 완전히 동일한 개발 환경을 얻습니다. **"내 PC에서는 됐는데"** 문제가 구조적으로 사라집니다.

### Vagrant의 한계 — 다음 단계가 필요한 이유

Vagrant를 실제로 사용하다 보면 세 가지 벽에 부딪힙니다.

**한계 1: Shell Script의 비일관성 (작성자 의존)**

`provision` 블록에 Shell Script를 직접 작성하면 작성자마다 표현 방식이 달라집니다. "맞고 틀리고"의 기준이 없어 팀 내 통일이 어렵습니다.

```bash
# 팀원 A 스타일
apt-get update && apt-get install -y nginx mysql-server

# 팀원 B 스타일 — 같은 결과지만 완전히 다른 코드
apt update
apt install nginx -y
apt install mysql-server -y
```

**한계 2: 이미 운영 중인 서버에는 적용 불가**

Vagrant는 VM을 **처음 만들 때**의 초기화에 특화돼 있습니다. 이미 실행 중인 운영 서버에 추가 설정을 적용하거나, 수십 대 서버에 동시에 패치를 배포하는 작업에는 적합하지 않습니다.

**한계 3: 실제 운영 환경으로의 이식성 부족**

`Vagrantfile`은 로컬 개발 환경에는 훌륭하지만, AWS EC2나 데이터센터 서버에 그대로 쓰기 어렵습니다. IP 주소, CPU 아키텍처(x86/ARM), 클라우드 API 등 환경마다 다른 요소들이 너무 많습니다.

이 세 가지 한계를 해결하기 위해 **구성 관리 도구(Configuration Management Tool)**가 등장합니다.

---

## 3. 구성 관리 도구의 5가지 핵심 특징

Ansible, Puppet, Chef 등 구성 관리 도구들이 공통적으로 가지는 철학을 이해하면 IaC의 본질이 보입니다. 이 다섯 가지 특징은 단순한 기능 설명이 아니라, Shell Script와 구성 관리 도구가 근본적으로 다른 이유를 설명합니다.

### 특징 1: 선언적(Declarative) — "어떻게"가 아니라 "무엇을"

Shell Script는 **절차적(Procedural)**입니다. 원하는 결과에 도달하는 방법을 순서대로 기술합니다.

```bash
# 절차적 방식: "이 순서대로 실행해라"
apt-get update
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
```

구성 관리 도구는 **선언적(Declarative)**입니다. 최종적으로 어떤 상태여야 하는지를 기술합니다. "어떻게 그 상태에 도달할지"는 도구가 알아서 결정합니다.

```yaml
# 선언적 방식: "이런 상태여야 한다"
- name: nginx가 설치되고 실행 중이어야 한다
  service:
    name: nginx
    state: started   # "실행 중 상태"여야 한다
    enabled: yes     # "부팅 시 자동 시작 상태"여야 한다
```

선언적 코드는 **의도가 명확**합니다. 절차적 코드는 명령의 나열이라 "이게 왜 있는 코드지?"라고 묻게 되지만, 선언적 코드는 "nginx가 실행 중이어야 한다"는 의도를 바로 읽을 수 있습니다.

### 특징 2: 추상화(Abstraction) — OS 차이를 숨긴다

Ubuntu는 `apt`, CentOS/RHEL은 `yum`/`dnf`, Amazon Linux 2023은 `dnf`로 패키지를 설치합니다. Shell Script로는 OS마다 다른 분기를 직접 작성해야 합니다.

```bash
# Shell Script: OS를 직접 판별해 분기해야 함
if [ -f /etc/debian_version ]; then
    apt-get install -y nginx
elif [ -f /etc/redhat-release ]; then
    yum install -y nginx
fi
```

Ansible은 이 OS별 차이를 **추상화 레이어** 뒤에 숨깁니다.

```yaml
# Ansible: OS 종류에 상관없이 동일한 코드
- name: install nginx
  package:         # OS를 자동 감지해 적절한 패키지 관리자 사용
    name: nginx
    state: present
```

`package` 모듈은 Ubuntu라면 `apt`, CentOS라면 `yum`을 자동으로 선택합니다. 서버 운영자가 OS 세부 사항을 몰라도 됩니다. 이 추상화가 수십 종의 이기종 서버를 단일 플레이북으로 관리할 수 있게 해줍니다.

### 특징 3: 수렴화(Convergence) — 현재 상태와 무관하게 목표 상태로

"현재 상태가 어떻든 간에, 반복 실행하면 결국 목표 상태에 도달한다"는 개념입니다. Shell Script는 이 보장을 주지 않습니다.

```bash
# Shell Script: 이전 상태를 알아야 안전하게 실행 가능
sed -i 's/worker_processes 1/worker_processes 4/' /etc/nginx/nginx.conf
# 이미 4로 설정돼 있다면? → 의도치 않은 결과 가능
# 값이 아예 없다면? → 아무 변경 없음, 원하는 결과를 못 얻음
```

Ansible의 `lineinfile` 모듈은 파일의 현재 상태를 직접 확인하고 필요한 경우에만 변경합니다.

```yaml
# Ansible: 현재 상태를 직접 확인하고 필요한 경우에만 변경
- name: nginx worker_processes를 4로 설정
  lineinfile:
    path: /etc/nginx/nginx.conf
    regexp: '^worker_processes'
    line: 'worker_processes 4;'
```

이 작업을 10번 실행해도 결과는 항상 `worker_processes 4;`입니다. **현재 상태가 0이든, 1이든, 이미 4든 상관없이** 목표 상태로 수렴합니다.

### 특징 4: 멱등성(Idempotency) — 몇 번을 실행해도 결과는 같다

수학적으로 표현하면 `f(f(x)) = f(x)`입니다. **같은 작업을 반복 적용해도 결과가 달라지지 않는 성질**입니다.

일상의 비유로 설명하면: 에어컨 리모컨으로 22도를 설정합니다. 이미 22도인 상태에서 버튼을 다시 눌러도 여전히 22도입니다. 온도를 바꾸는 게 아니라 **22도라는 상태를 보장**하는 것입니다.

```yaml
# 이 플레이북을 10번 실행해도 결과는 항상 동일합니다
- name: nginx 패키지 설치
  apt:
    name: nginx
    state: present  # "설치된 상태"여야 한다
    # → 이미 설치돼 있으면? 아무것도 하지 않음 (changed: false)
    # → 설치 안 돼 있으면? 설치 진행 (changed: true)
```

멱등성이 있으면:
- CI/CD 파이프라인에서 **매 배포마다 플레이북을 실행**해도 안전합니다
- 실수로 두 번 실행했을 때도 **시스템이 망가지지 않습니다**
- 중간에 실패한 플레이북을 **다시 실행**해도 완료된 단계를 건너뜁니다

> {: .prompt-warning }
> **주의:** Ansible의 `command`, `shell` 모듈은 기본적으로 멱등성이 **없습니다**. 이 모듈을 쓸 때는 `creates`, `removes`, `when` 조건을 반드시 함께 사용해 멱등성을 직접 구현해야 합니다.

### 특징 5: 간소화(Simplicity) — 코드가 곧 문서

선언적으로 작성된 코드는 짧고, 의도가 명확하며, 추가적인 문서 없이 스스로를 설명합니다.

```yaml
# 이 코드는 별도 설명 없이도 읽힙니다
- name: MySQL이 설치되고 실행 중이며, 부팅 시 자동 시작해야 한다
  service:
    name: mysql
    state: started
    enabled: yes
```

여기서 파생되는 추가 이점들:

- **Portability**: YAML 텍스트 파일이라 Git에 올리고, 공유하고, PR로 리뷰받기 쉽습니다
- **Diff**: `git diff`로 "무엇이 어떻게 바뀌었는지"를 즉시 파악할 수 있습니다
- **Rollback**: 문제 발생 시 이전 커밋으로 되돌려 이전 상태를 재현할 수 있습니다
- **Reuse**: Ansible Galaxy 등 커뮤니티가 공유한 Role(역할 묶음)을 그대로 가져다 쓸 수 있습니다

---

## 4. Ansible — YAML로 쓰는 서버 설정

### Ansible이란?

**Ansible**은 Red Hat이 개발한 오픈소스 구성 관리 도구입니다. 다른 도구들과 달리 **에이전트가 필요 없습니다(agentless)**. 대상 서버에 별도 소프트웨어를 설치하지 않고 SSH만 연결되면 작동하기 때문에, 도입 장벽이 낮고 기존 서버에 바로 적용할 수 있습니다.

2026년 기준 엔터프라이즈 환경의 50% 이상이 Ansible을 사용하고 있으며, Terraform과 함께 가장 널리 쓰이는 IaC 도구입니다.

### Ansible의 3가지 핵심 구성 요소

**1. 인벤토리(Inventory) — "어떤 서버에 적용할까?"**

관리 대상 서버를 목록으로 정의합니다. 서버를 그룹으로 묶어 역할별로 관리할 수 있습니다.

```ini
# inventory/development

# 웹 서버 그룹
[webservers]
192.168.33.10
192.168.33.11

# DB 서버 그룹
[dbservers]
192.168.33.20

# 공통 변수 설정
[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

동적 인벤토리(Dynamic Inventory)를 사용하면 AWS EC2, GCP 같은 클라우드에서 실행 중인 인스턴스를 자동으로 가져올 수도 있습니다.

**2. Role — "어떤 역할을 부여할까?"**

서버의 역할(웹 서버, DB 서버 등)을 묶음으로 정의합니다. Role은 재사용 가능한 단위로, Ansible Galaxy에서 커뮤니티가 만든 수천 개의 Role을 검색해 그대로 사용할 수 있습니다.

```
roles/
├── common/          # 모든 서버에 공통 적용 (보안 패치, 기본 패키지)
│   └── tasks/
│       └── main.yml
├── nginx/           # nginx 웹 서버 설치 및 설정
│   ├── tasks/
│   │   └── main.yml
│   ├── handlers/
│   │   └── main.yml
│   ├── templates/
│   │   └── nginx.conf.j2   # Jinja2 템플릿
│   └── defaults/
│       └── main.yml        # 기본 변수값
└── mysql/           # MySQL DB 설치 및 설정
    └── tasks/
        └── main.yml
```

**3. Playbook — "어떤 서버에 어떤 Role을 적용할까?"**

인벤토리의 서버 그룹과 Role을 연결하는 최상위 파일입니다.

```yaml
# site.yml — 전체 인프라의 설정 진입점

---
# 웹 서버 그룹에 적용
- hosts: webservers
  become: yes          # sudo 권한으로 실행
  vars:
    nginx_worker_processes: 4
    nginx_port: 80
  roles:
    - common           # 공통 기본 설정
    - nginx            # nginx 설치 및 구성
    - serverspec       # 테스트 도구 설치

# DB 서버 그룹에 적용
- hosts: dbservers
  become: yes
  roles:
    - common
    - mysql
```

### 실전 Task 코드 — nginx Role

```yaml
# roles/nginx/tasks/main.yml

---
# 패키지 설치 (멱등: 이미 설치돼 있으면 건너뜀)
- name: nginx 패키지 설치
  apt:
    name: nginx
    state: present
    update_cache: yes

# 설정 파일 배포 (Jinja2 템플릿으로 변수 치환)
- name: nginx 설정 파일 배포
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
  notify: restart nginx    # 파일이 변경됐을 때만 핸들러 실행

# 서비스 상태 관리
- name: nginx 서비스 시작 및 자동 시작 등록
  service:
    name: nginx
    state: started         # "실행 중 상태"여야 한다
    enabled: yes           # "부팅 시 자동 시작 상태"여야 한다

# 방화벽 설정 (ufw 모듈 - OS 종류 무관)
- name: HTTP 포트 허용
  ufw:
    rule: allow
    port: '80'
    proto: tcp
```

```yaml
# roles/nginx/handlers/main.yml

---
# notify에서 호출될 때만 실행되는 핸들러
# 여러 태스크가 같은 핸들러를 notify해도 플레이북 마지막에 한 번만 실행됨
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

```jinja2
{# roles/nginx/templates/nginx.conf.j2 #}
{# {{ }} 안의 변수는 Playbook의 vars에서 정의된 값으로 치환됨 #}

worker_processes {{ nginx_worker_processes }};

events {
    worker_connections 1024;
}

http {
    server {
        listen {{ nginx_port }};
        root /var/www/html;
        index index.html;
    }
}
```

### Ansible 실행 명령어

```bash
# 기본 실행: 개발 인벤토리에 전체 플레이북 적용
ansible-playbook -i inventory/development site.yml

# --diff: 파일의 변경 전/후를 직접 보여줌 (적용 전 검토에 유용)
ansible-playbook -i inventory/development site.yml --diff

# --check: 실제 변경 없이 어떤 변경이 일어날지 시뮬레이션 (dry-run)
ansible-playbook -i inventory/development site.yml --check

# 특정 태그가 붙은 태스크만 실행
ansible-playbook -i inventory/development site.yml --tags "nginx"

# 특정 호스트만 대상으로 실행
ansible-playbook -i inventory/development site.yml --limit "192.168.33.10"

# 연결 테스트 (ping 모듈)
ansible -i inventory/development all -m ping
```

`--check`와 `--diff`를 함께 사용하면 실제로 아무것도 바꾸지 않으면서 어떤 변경이 일어날지 미리 확인할 수 있습니다. **프로덕션 서버에 플레이북을 처음 적용하기 전, 반드시 이 조합으로 사전 검토하는 것이 권장됩니다.**

### Ansible 멱등성 확인 방법

플레이북을 두 번 실행했을 때 두 번째 실행에서 `changed: 0`이 나오면 멱등성이 올바르게 구현된 것입니다.

```
# 첫 번째 실행 (변경 발생)
PLAY RECAP ************************************************************
192.168.33.10  : ok=5  changed=3  unreachable=0  failed=0

# 두 번째 실행 (동일 플레이북 재실행)
PLAY RECAP ************************************************************
192.168.33.10  : ok=5  changed=0  unreachable=0  failed=0
#                             ↑
#               changed=0 이면 멱등성 달성!
```

---

## 5. Serverspec — 인프라에도 테스트를

### 왜 인프라 테스트가 필요한가?

소프트웨어 개발에는 유닛 테스트가 있습니다. "이 함수는 이 입력에 대해 이 결과를 반환해야 한다"는 기대값을 코드로 작성하고 자동으로 검증합니다.

그런데 Ansible로 nginx를 설치한 후, 실제로 포트 80이 열려 있는지, 서비스가 정말 실행 중인지, 설정 파일이 올바른 위치에 있는지는 어떻게 확인할까요? 지금까지는 직접 서버에 접속해 `curl localhost`, `systemctl status nginx`, `ls /etc/nginx/` 같은 명령어를 수동으로 입력해야 했습니다.

**Serverspec**은 이 수동 검증 과정을 **테스트 코드**로 자동화합니다.

### Serverspec의 특징

| 항목 | 설명 |
| :--- | :--- |
| **언어** | Ruby (RSpec 문법) |
| **방식** | SSH 또는 로컬 실행으로 서버 상태 검증 |
| **출력** | 사람이 읽기 쉬운 테스트 리포트 |
| **관리** | 텍스트 파일이므로 Git으로 버전 관리 가능 |

Serverspec 코드는 자연어에 가까운 문법을 사용합니다. "모호함을 없앤 자연어"라고 이해하면 됩니다. `should be_listening`은 "포트가 열려 있어야 한다"는 의미를 가지지만, 이를 정확히 "TCP 포트가 LISTEN 상태"로 정의합니다.

### 설치 및 초기화

```bash
# Serverspec 설치 (Ruby gem)
gem install serverspec

# 새 디렉터리 생성 후 초기화
mkdir ~/serverspec && cd ~/serverspec
serverspec-init
```

대화형 초기화 진행:

```
Select OS type:
  1) UN*X
  2) Windows
Select number: 1   ← Linux/macOS는 UN*X 계열

Select a backend type:
  1) SSH
  2) Exec (local)
Select number: 2   ← 로컬에서 테스트할 경우 (Vagrant VM 내부에서 실행)
```

초기화 후 생성되는 파일 구조:

```
serverspec/
├── Rakefile           # rake spec 실행 진입점
├── .rspec             # RSpec 설정
└── spec/
    ├── spec_helper.rb # 공통 설정
    └── localhost/
        └── sample_spec.rb  # 테스트 코드 작성 위치
```

### 테스트 코드 작성 — nginx 검증

```ruby
# spec/localhost/nginx_spec.rb

require 'spec_helper'

# ──────────────────────────────────────────
# 패키지 설치 확인
# ──────────────────────────────────────────
describe package('nginx') do
  it { should be_installed }
end

# ──────────────────────────────────────────
# 서비스 상태 확인
# ──────────────────────────────────────────
describe service('nginx') do
  it { should be_enabled }    # 부팅 시 자동 시작 등록 여부
  it { should be_running }    # 현재 실행 중 여부
end

# ──────────────────────────────────────────
# 포트 상태 확인
# ──────────────────────────────────────────
describe port(80) do
  it { should be_listening }
end

# ──────────────────────────────────────────
# 설정 파일 존재 및 권한 확인
# ──────────────────────────────────────────
describe file('/etc/nginx/nginx.conf') do
  it { should exist }
  it { should be_file }
  it { should be_owned_by 'root' }
  its(:content) { should match /worker_processes 4/ }  # 설정값 검증
end

# ──────────────────────────────────────────
# 프로세스 확인
# ──────────────────────────────────────────
describe process('nginx') do
  it { should be_running }
  its(:user) { should eq 'root' }   # nginx master는 root로 실행
end

# ──────────────────────────────────────────
# 실제 HTTP 응답 확인
# ──────────────────────────────────────────
describe command('curl -s -o /dev/null -w "%{http_code}" http://localhost') do
  its(:stdout) { should eq '200' }
end
```

`describe`와 `it { should ... }` 구문을 자연어처럼 읽으면 의도가 명확히 드러납니다. 이 코드가 **인프라의 명세서(Specification)**이자 **자동 검증 도구** 역할을 동시에 합니다.

### 테스트 실행 및 결과

```bash
rake spec
```

**성공 시 출력:**

```
Package "nginx"
  should be installed

Service "nginx"
  should be enabled
  should be running

Port "80"
  should be listening

File "/etc/nginx/nginx.conf"
  should exist
  should be file
  should be owned by "root"
  content
    should match /worker_processes 4/

Process "nginx"
  should be running
  user
    should eq "root"

Command "curl -s -o /dev/null -w "%{http_code}" http://localhost"
  stdout
    should eq "200"

Finished in 0.43 seconds (files took 1.2 seconds to load)
10 examples, 0 failures
```

**실패 시 출력 (예: nginx가 시작되지 않은 경우):**

```
Service "nginx"
  should be running (FAILED - 1)

Port "80"
  should be listening (FAILED - 2)

Failures:
  1) Service "nginx" should be running
     expected Service "nginx" to be running

  2) Port "80" should be listening
     expected Port "80" to be listening

Finished in 0.38 seconds
10 examples, 2 failures
```

실패한 항목과 기대값이 명확히 출력됩니다. Ansible 플레이북 실행 후 Serverspec을 자동으로 돌리도록 CI/CD 파이프라인에 연결하면, **"설정 적용 → 자동 검증"** 흐름이 완전히 자동화됩니다.

---

## 6. 전체 IaC 흐름 정리

세 도구가 어떻게 연결되는지 전체 그림을 봅니다.

```
┌─────────────────────────────────────────────────────┐
│  [1단계] 환경 코드화 — Vagrant                        │
│                                                       │
│  Vagrantfile                                          │
│    └─ config.vm.box = "ubuntu/focal64"               │
│    └─ config.vm.network ...                          │
│    └─ config.vm.provision ...                        │
│                   ↓                                   │
│           vagrant up                                  │
│                   ↓                                   │
│         VM 생성 완료                                  │
└─────────────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│  [2단계] 구성 코드화 — Ansible                        │
│                                                       │
│  inventory/   →  어떤 서버에                          │
│  roles/       →  어떤 설정을                          │
│  site.yml     →  어떻게 연결할지                      │
│                   ↓                                   │
│  ansible-playbook -i inventory/dev site.yml          │
│                   ↓                                   │
│  nginx 설치, 서비스 시작, 설정 파일 배포 완료          │
└─────────────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│  [3단계] 검증 코드화 — Serverspec                     │
│                                                       │
│  spec/localhost/nginx_spec.rb                        │
│    └─ package('nginx') → be_installed               │
│    └─ service('nginx') → be_running                 │
│    └─ port(80)         → be_listening               │
│                   ↓                                   │
│               rake spec                               │
│                   ↓                                   │
│  10 examples, 0 failures ✓                           │
└─────────────────────────────────────────────────────┘
```

이 세 단계가 모두 코드로 관리되면:

- **누구든** `vagrant up` → `ansible-playbook` → `rake spec` 세 명령어로 동일한 환경을 재현하고 검증할 수 있습니다
- **Git으로** 모든 변경 이력을 추적하고, 문제 발생 시 이전 버전으로 롤백할 수 있습니다
- **CI/CD 파이프라인에** 연결하면 인프라 변경도 소프트웨어 배포처럼 자동화됩니다

---

## 7. 정리

| 도구 | 역할 | 핵심 개념 |
| :--- | :--- | :--- |
| **Vagrant** | VM 환경 코드화 | `Vagrantfile` 하나로 환경 재현 |
| **Ansible** | 서버 구성 자동화 | 선언적 · 멱등적 플레이북 |
| **Serverspec** | 인프라 상태 검증 | 테스트 코드로 설정 자동 확인 |

그리고 이 세 도구를 관통하는 IaC의 핵심 철학 5가지:

| 특징 | 의미 |
| :--- | :--- |
| **선언적** | "어떻게"가 아니라 "무엇을(어떤 상태)" 기술 |
| **추상화** | OS, 환경 차이를 숨겨 동일한 코드로 처리 |
| **수렴화** | 현재 상태와 무관하게 목표 상태로 귀결 |
| **멱등성** | 몇 번을 실행해도 항상 동일한 결과 |
| **간소화** | 코드 자체가 문서이자 실행 파일 |

IaC의 본질은 인프라를 소프트웨어처럼 다루는 것입니다. 코드로 작성하고, Git으로 관리하고, 자동으로 검증합니다. "서버 담당자만 아는 지식"을 누구나 읽고 실행할 수 있는 코드로 만드는 것이 IaC가 해결하는 핵심 문제입니다.
