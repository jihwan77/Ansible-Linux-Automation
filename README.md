

# Ansible 기반 Linux 서버 초기 세팅 자동화 프로젝트

## 1. 프로젝트 개요

본 프로젝트는 신규 Linux 서버가 배포되었을 때 반복적으로 수행해야 하는 초기 설정 작업을 Ansible Playbook으로 자동화한 미니 프로젝트입니다.

수동으로 각 서버에 접속하여 사용자 생성, sudo 권한 설정, 기본 패키지 설치, 방화벽 설정, 시간 동기화, nginx 설치 등을 수행하는 대신, Ansible Control 서버에서 여러 관리 대상 서버에 동일한 설정을 일괄 적용하도록 구성했습니다.

이를 통해 Linux 서버 운영 초기 단계에서 필요한 기본 환경을 표준화하고, 반복 작업을 줄이는 것을 목표로 했습니다.

---

## 2. 프로젝트 목표

* Ansible을 이용한 Linux 서버 초기 구성 자동화
* SSH Key 기반 원격 서버 관리 구성
* Inventory 기반 다중 서버 제어
* Playbook과 Role 구조를 활용한 설정 분리
* 사용자, sudo, SSH, 패키지, 방화벽, 서비스 설정 자동화
* 동일 Playbook을 여러 번 실행해도 안정적으로 동작하는 멱등성 확보

---

## 3. 인프라 구성

본 프로젝트는 vSphere 환경에서 Linux VM을 생성하여 구성했습니다.

```text
vSphere
├── ansible-control   # Ansible 실행 서버
├── linux-node1       # 관리 대상 서버
└── linux-node2       # 관리 대상 서버
```

### VM 구성 예시

| VM 이름           | 역할                   | OS               | 설명               |
| --------------- | -------------------- | ---------------- | ---------------- |
| ansible-control | Ansible Control Node | Rocky-9.5-x86_64-minimal | Ansible 명령 실행 서버 |
| linux-node1     | Managed Node         | Rocky-9.5-x86_64-minimal | 초기 설정 자동화 대상 서버  |
| linux-node2     | Managed Node         | Rocky-9.5-x86_64-minimal | 초기 설정 자동화 대상 서버  |

---

## 4. 사용 기술

| 기술        | 설명                                 |
| --------- | ---------------------------------- |
| Ansible   | 서버 설정 자동화 도구                       |
| SSH       | Control Node와 Managed Node 간 원격 접속 |
| YAML      | Ansible Playbook 및 변수 파일 작성        |
| Linux     | Rocky Linux / RHEL 계열 서버 환경        |
| systemd   | 서비스 enable/start 관리                |
| firewalld | Linux 방화벽 서비스 관리                   |
| chrony    | 시간 동기화 서비스                         |
| nginx     | 웹 서버 설치 및 서비스 상태 검증                |
| SELinux   | 보안 정책 상태 점검                        |

---

## 5. 네트워크 구성

각 VM은 동일 네트워크 대역에 배치했습니다.

```text
ansible-control : 172.16.1.80
linux-node1     : 172.16.1.81
linux-node2     : 172.16.1.82
```
![alt text](picture/image-7.png)


각 서버에는 식별하기 쉬운 hostname을 설정했습니다.

```bash
sudo hostnamectl set-hostname ansible-control
sudo hostnamectl set-hostname linux-node1
sudo hostnamectl set-hostname linux-node2
```

Control 서버의 `/etc/hosts`에는 관리 대상 서버 정보를 등록했습니다.

```bash
sudo vi /etc/hosts
```

```text
172.16.1.80 ansible-control
172.16.1.81 linux-node1
172.16.1.82 linux-node2
```

---

## 6. Ansible Control 서버 구성

Ansible Control 서버에서 Ansible을 설치했습니다.

```bash
sudo dnf install -y epel-release
sudo dnf install -y ansible-core git vim
```

설치 확인:

```bash
ansible --version
```

프로젝트 디렉토리 생성:

```bash
mkdir -p ~/ansible-linux-init
cd ~/ansible-linux-init
```

---

## 7. SSH Key 기반 접속 구성

Control 서버에서 SSH Key를 생성한 뒤, 관리 대상 서버에 공개키를 복사했습니다.

```bash
ssh-keygen -t ed25519
```

```bash
ssh-copy-id ansible@linux-node1
ssh-copy-id ansible@linux-node2
```

SSH 접속 확인:

```bash
ssh ansible@linux-node1
ssh ansible@linux-node2
```

이를 통해 Ansible이 비밀번호 입력 없이 관리 대상 서버에 접속할 수 있도록 구성했습니다.

---

## 8. Inventory 구성

`inventory.ini`

```ini
[linux_servers]
linux-node1 ansible_host=172.16.1.81
linux-node2 ansible_host=172.16.1.82

[linux_servers:vars]
ansible_user=ansible
ansible_become=true
ansible_become_method=sudo
```

Inventory 연결 테스트:

```bash
ansible -i inventory.ini linux_servers -m ping
```

정상 연결 시 다음과 같이 `pong` 응답을 확인할 수 있습니다.

![alt text](picture/image-3.png)



---

## 9. 프로젝트 디렉토리 구조

본 프로젝트는 Ansible Role 구조를 사용하여 기능별 설정을 분리했습니다.

```text
ansible-linux-init/
├── inventory.ini
├── site.yml
├── group_vars/
│   └── all.yml
├── roles/
│   ├── users/
│   │   └── tasks/
│   │       └── main.yml
│   ├── sudoers/
│   │   └── tasks/
│   │       └── main.yml
│   ├── packages/
│   │   └── tasks/
│   │       └── main.yml
│   ├── chrony/
│   │   └── tasks/
│   │       └── main.yml
│   ├── firewalld/
│   │   └── tasks/
│   │       └── main.yml
│   ├── nginx/
│   │   └── tasks/
│   │       └── main.yml
│   ├── ssh/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── handlers/
│   │       └── main.yml
│   └── selinux/
│       └── tasks/
│           └── main.yml
└── README.md
```

---

## 10. 자동화 항목

| 항목           | 설명                                            |
| ------------ | --------------------------------------------- |
| 사용자 생성       | `devops` 사용자 생성 및 wheel 그룹 추가                 |
| sudoers 설정   | wheel 그룹 sudo 권한 설정                           |
| 기본 패키지 설치    | vim, git, wget, curl, net-tools 등 기본 운영 도구 설치 |
| chrony 설정    | 시간 동기화 서비스 설치 및 활성화                           |
| firewalld 설정 | 방화벽 서비스 활성화 및 ssh/http 허용                     |
| nginx 설치     | nginx 설치 및 systemd 서비스 활성화                    |
| SSH 보안 설정    | root SSH 로그인 차단                               |
| SELinux 점검   | SELinux 상태 확인                                 |

---

## 11. 변수 파일 구성

`group_vars/all.yml`

```yaml
---
admin_users:
  - name: devops
    groups: wheel
    shell: /bin/bash

base_packages:
  - vim
  - git
  - wget
  - curl
  - net-tools
  - bind-utils
  - bash-completion
  - chrony
  - firewalld
  - nginx
  - audit

ssh_port: 22
permit_root_login: "no"
password_authentication: "yes"
```

SSH Key 기반 접속이 완전히 안정화되기 전까지는 `PasswordAuthentication`을 `yes`로 유지했습니다.
이후 운영 환경에서는 SSH Key 접속을 검증한 뒤 `no`로 변경할 수 있습니다.

---

## 12. 메인 Playbook

`site.yml`

```yaml
---
- name: Initial Linux server setup
  hosts: linux_servers
  become: true

  roles:
    - users
    - sudoers
    - packages
    - chrony
    - firewalld
    - nginx
    - ssh
    - selinux
```

실행 명령:

```bash
ansible-playbook -i inventory.ini site.yml
```

![alt text](picture/image-4.png)

---

## 13. Role별 주요 구성

### 13.1 users Role

관리 대상 서버에 관리자용 사용자를 생성하고 wheel 그룹에 추가합니다.

`roles/users/tasks/main.yml`

```yaml
---
- name: Create admin users
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    append: true
    shell: "{{ item.shell }}"
    state: present
  loop: "{{ admin_users }}"
```

---

### 13.2 sudoers Role

wheel 그룹에 sudo 권한을 부여합니다.

`roles/sudoers/tasks/main.yml`

```yaml
---
- name: Allow wheel group to use sudo
  ansible.builtin.lineinfile:
    path: /etc/sudoers
    regexp: '^%wheel'
    line: '%wheel ALL=(ALL) ALL'
    validate: '/usr/sbin/visudo -cf %s'
```

`validate` 옵션을 사용하여 sudoers 파일 수정 전 문법 검사를 수행하도록 구성했습니다.
이를 통해 잘못된 sudoers 설정으로 인해 sudo 사용이 불가능해지는 문제를 방지했습니다.

---

### 13.3 packages Role

운영에 필요한 기본 패키지를 설치합니다.

`roles/packages/tasks/main.yml`

```yaml
---
- name: Install base packages
  ansible.builtin.dnf:
    name: "{{ base_packages }}"
    state: present
```

---

### 13.4 chrony Role

시간 동기화를 위해 chrony를 설치하고 서비스를 활성화합니다.

`roles/chrony/tasks/main.yml`

```yaml
---
- name: Install chrony
  ansible.builtin.dnf:
    name: chrony
    state: present

- name: Enable and start chronyd
  ansible.builtin.systemd:
    name: chronyd
    enabled: true
    state: started
```

검증 명령:

```bash
ansible -i inventory.ini linux_servers -m command -a "chronyc tracking" -b
```

![alt text](picture/image-5.png)


---

### 13.5 firewalld Role

방화벽 서비스를 활성화하고 SSH, HTTP 서비스를 허용합니다.

`roles/firewalld/tasks/main.yml`

```yaml
---
- name: Install firewalld
  ansible.builtin.dnf:
    name: firewalld
    state: present

- name: Enable and start firewalld
  ansible.builtin.systemd:
    name: firewalld
    enabled: true
    state: started

- name: Allow SSH service
  ansible.posix.firewalld:
    service: ssh
    permanent: true
    immediate: true
    state: enabled

- name: Allow HTTP service
  ansible.posix.firewalld:
    service: http
    permanent: true
    immediate: true
    state: enabled
```

필요한 collection 설치:

```bash
ansible-galaxy collection install ansible.posix
```

검증 명령:

```bash
ansible -i inventory.ini linux_servers -m command -a "firewall-cmd --list-services" -b
```

---

### 13.6 nginx Role

nginx를 설치하고 systemd를 통해 자동 시작되도록 구성합니다.

`roles/nginx/tasks/main.yml`

```yaml
---
- name: Install nginx
  ansible.builtin.dnf:
    name: nginx
    state: present

- name: Enable and start nginx
  ansible.builtin.systemd:
    name: nginx
    enabled: true
    state: started
```

검증 명령:

```bash
ansible -i inventory.ini linux_servers -m command -a "systemctl is-active nginx" -b
ansible -i inventory.ini linux_servers -m command -a "systemctl is-enabled nginx" -b
```

브라우저 또는 curl로 nginx 기본 페이지 접속을 확인할 수 있습니다.

```bash
curl http://172.16.1.81
curl http://172.16.1.82
```

---

### 13.7 SSH Role

SSH 설정에서 root 계정의 직접 로그인을 차단합니다.

`roles/ssh/tasks/main.yml`

```yaml
---
- name: Disable root SSH login
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: "PermitRootLogin {{ permit_root_login }}"
    backup: true
  notify: Restart sshd

- name: Configure SSH password authentication
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: "PasswordAuthentication {{ password_authentication }}"
    backup: true
  notify: Restart sshd
```

`roles/ssh/handlers/main.yml`

```yaml
---
- name: Restart sshd
  ansible.builtin.systemd:
    name: sshd
    state: restarted
```

SSH 설정 파일 변경 시 handler를 통해 sshd 서비스를 재시작하도록 구성했습니다.

---

### 13.8 SELinux Role

SELinux는 보안상 중요한 기능이므로 비활성화하지 않고 현재 상태를 점검하는 방식으로 구성했습니다.

`roles/selinux/tasks/main.yml`

```yaml
---
- name: Check SELinux status
  ansible.builtin.command: getenforce
  register: selinux_status
  changed_when: false

- name: Print SELinux status
  ansible.builtin.debug:
    msg: "SELinux status is {{ selinux_status.stdout }}"
```

검증 명령:

```bash
ansible -i inventory.ini linux_servers -m command -a "getenforce" -b
```

---

## 14. 실행 방법

### 1단계. Inventory 연결 확인

```bash
ansible -i inventory.ini linux_servers -m ping
```

### 2단계. Playbook 실행

```bash
ansible-playbook -i inventory.ini site.yml
```

### 3단계. 적용 결과 확인

```bash
ansible -i inventory.ini linux_servers -m command -a "id devops" -b
ansible -i inventory.ini linux_servers -m command -a "systemctl is-active nginx" -b
ansible -i inventory.ini linux_servers -m command -a "systemctl is-enabled nginx" -b
ansible -i inventory.ini linux_servers -m command -a "firewall-cmd --list-services" -b
ansible -i inventory.ini linux_servers -m command -a "getenforce" -b
```

---

## 15. 검증 결과

Playbook 실행 후 다음 항목을 확인했습니다.

| 검증 항목       | 확인 방법                          | 기대 결과                      |
| ----------- | ------------------------------ | -------------------------- |
| Ansible 연결  | `ansible -m ping`              | pong 응답                    |
| 사용자 생성      | `id devops`                    | devops 사용자 확인              |
| sudo 권한     | wheel 그룹 확인                    | devops 사용자가 wheel 그룹에 포함   |
| nginx 상태    | `systemctl is-active nginx`    | active                     |
| nginx 자동 시작 | `systemctl is-enabled nginx`   | enabled                    |
| 방화벽 허용 서비스  | `firewall-cmd --list-services` | ssh, http 포함               |
| chrony 상태   | `chronyc tracking`             | 시간 동기화 상태 출력               |
| SELinux 상태  | `getenforce`                   | Enforcing 또는 Permissive 출력 |

---

## 16. 프로젝트에서 학습한 내용

### 16.1 Ansible Inventory 기반 서버 관리

관리 대상 서버의 IP, 접속 사용자, become 설정을 Inventory에 정의하여 여러 서버를 일괄 관리하는 방법을 학습했습니다.

### 16.2 Playbook과 Role 분리

하나의 긴 Playbook에 모든 작업을 작성하지 않고, users, sudoers, packages, chrony, firewalld, nginx, ssh, selinux 역할로 분리했습니다.
이를 통해 설정 파일의 가독성과 유지보수성을 높였습니다.

### 16.3 멱등성

Ansible의 주요 특징인 멱등성을 확인했습니다.
동일한 Playbook을 여러 번 실행해도 이미 원하는 상태라면 중복 변경이 발생하지 않고, 시스템 상태가 안정적으로 유지됩니다.

### 16.4 systemd 서비스 관리

nginx, chronyd, firewalld와 같은 서비스를 Ansible의 systemd 모듈로 관리했습니다.
서비스를 단순히 실행하는 것뿐 아니라 재부팅 후에도 자동 시작되도록 `enabled: true` 설정을 적용했습니다.

### 16.5 sudoers 설정 안정성

sudoers 파일은 잘못 수정하면 sudo 사용이 불가능해질 수 있기 때문에 `visudo -cf` 검증을 추가했습니다.
이를 통해 설정 자동화 과정에서도 안전성을 확보했습니다.

### 16.6 SELinux 보안 접근

SELinux를 단순히 비활성화하지 않고 현재 상태를 점검하는 방식으로 구성했습니다.
보안 기능을 무조건 끄는 방식이 아니라, 운영 환경에서 현재 보안 상태를 확인하는 방향으로 접근했습니다.

---

## 17. 트러블슈팅



### 17.1 root SSH 로그인 차단 설정 미적용 문제

SSH 보안 설정 Role에서 `/etc/ssh/sshd_config` 파일의 `PermitRootLogin` 값을 `no`로 변경했지만, 실제로는 MobaXterm에서 root 계정으로 SSH 접속이 가능한 문제가 발생했습니다.


즉, 설정 파일에는 `PermitRootLogin no`가 보였지만 실제 SSH 데몬이 적용 중인 값은 `yes`였습니다.

---

#### 원인

Rocky Linux / RHEL 계열의 OpenSSH 설정 파일에는 일반적으로 다음과 같은 Include 구문이 포함되어 있습니다.

```text
Include /etc/ssh/sshd_config.d/*.conf
```

이로 인해 `/etc/ssh/sshd_config`뿐만 아니라 `/etc/ssh/sshd_config.d/` 디렉토리 아래의 `.conf` 파일들도 함께 읽힙니다.

문제 원인을 찾기 위해 다음 명령으로 전체 SSH 설정 파일을 검색했습니다.

```bash
grep -Rni 'PermitRootLogin' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

확인 결과 다음과 같이 별도 drop-in 설정 파일에서 root 로그인을 허용하고 있었습니다.

```text
/etc/ssh/sshd_config.d/01-permitrootlogin.conf:3:PermitRootLogin yes
/etc/ssh/sshd_config:40:PermitRootLogin no
```

즉, 메인 설정 파일에서는 root 로그인을 차단하고 있었지만, `/etc/ssh/sshd_config.d/01-permitrootlogin.conf` 파일에서 다시 root 로그인을 허용하고 있었기 때문에 실제 적용값이 `yes`로 유지되었습니다.

```text
/etc/ssh/sshd_config.d/01-permitrootlogin.conf
  PermitRootLogin yes

/etc/ssh/sshd_config
  PermitRootLogin no

실제 적용값
  permitrootlogin yes
```

이 문제를 통해 SSH 설정은 단순히 `/etc/ssh/sshd_config`만 확인하는 것이 아니라, `sshd -T` 명령으로 실제 적용값을 검증해야 한다는 점을 확인했습니다.

---

#### 해결 방법



기존 SSH Role은 `/etc/ssh/sshd_config` 파일만 수정하도록 구성되어 있었습니다.

```yaml
- name: Disable root SSH login
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: "PermitRootLogin {{ permit_root_login }}"
    backup: true
  notify: Restart sshd
```

하지만 실제 문제는 `/etc/ssh/sshd_config.d/01-permitrootlogin.conf` 파일에서 발생했기 때문에, 향후에는 충돌 가능성이 있는 drop-in 파일도 함께 관리하는 방식이 더 적절하다고 판단했습니다.

개선된 방식은 다음과 같습니다.

```yaml
- name: Remove conflicting root login config
  ansible.builtin.file:
    path: /etc/ssh/sshd_config.d/01-permitrootlogin.conf
    state: absent
  notify: Restart sshd

- name: Configure SSH hardening drop-in
  ansible.builtin.copy:
    dest: /etc/ssh/sshd_config.d/00-hardening.conf
    owner: root
    group: root
    mode: "0644"
    content: |
      PermitRootLogin no
      PasswordAuthentication yes
  notify: Restart sshd
```

handler는 기존과 동일하게 구성합니다.

```yaml
- name: Restart sshd
  ansible.builtin.systemd:
    name: sshd
    state: restarted
```

이 방식은 SSH 보안 설정을 별도 drop-in 파일로 관리할 수 있어 설정 의도가 명확하고, Ansible이 관리하는 파일과 OS 기본 설정 파일을 분리할 수 있다는 장점이 있습니다.

다만 다른 `.conf` 파일에서 동일 옵션을 다시 설정하면 충돌이 발생할 수 있으므로, 최종적으로는 반드시 다음 명령으로 실제 적용값을 확인해야 합니다.

```bash
sshd -T | grep -i permitrootlogin
```

---

### 17.2 vSphere 사용자 지정 스크립트 미실행 문제

vSphere Template과 사용자 지정 규격을 이용해 VM을 배포하면서 `postcustomization` 스크립트를 통해 `ansible` 계정, SSH 공개키, sudoers 설정을 자동 생성하려고 했습니다.

하지만 템플릿에서 VM을 복제한 뒤에도 관리 대상 서버 내부에 `ansible` 사용자가 생성되지 않는 문제가 발생했습니다.


---

#### 원인

Rocky Linux에서 vSphere 사용자 지정 스크립트를 실행하려면 `open-vm-tools`의 custom script 실행 옵션이 활성화되어 있어야 합니다.

템플릿 원본 VM에서 다음 명령으로 설정값을 확인했습니다.

```bash
vmware-toolbox-cmd config get deployPkg enable-custom-scripts
```

문제가 있는 경우 다음과 같이 출력될 수 있습니다.

```text
[deployPkg] enable-custom-scripts = false
```

이 값이 `false`이면 vSphere 사용자 지정 규격에 스크립트를 입력하더라도 실제 게스트 OS 내부에서 스크립트가 실행되지 않습니다.

---

#### 해결 방법

템플릿 원본 VM에서 custom script 실행 옵션을 활성화했습니다.

```bash
vmware-toolbox-cmd config set deployPkg enable-custom-scripts true
systemctl restart vmtoolsd
```

설정값을 다시 확인했습니다.

```bash
vmware-toolbox-cmd config get deployPkg enable-custom-scripts
```

정상적으로 설정되면 다음과 같이 출력됩니다.

```text
[deployPkg] enable-custom-scripts = true
```

이후 해당 VM을 다시 템플릿으로 변환하고, 사용자 지정 규격을 적용하여 VM을 재배포했습니다.

---

#### 템플릿 VM 필수 패키지 확인

vSphere Guest Customization이 정상 동작하려면 템플릿 VM에 `open-vm-tools`와 관련 패키지가 설치되어 있어야 합니다.

Rocky Linux 템플릿 원본 VM에서 다음 패키지를 설치했습니다.

```bash
dnf install -y open-vm-tools perl sudo openssh-server
```

그리고 `vmtoolsd`와 `sshd` 서비스를 활성화했습니다.

```bash
systemctl enable --now vmtoolsd
systemctl enable --now sshd
```

서비스 상태는 다음 명령으로 확인했습니다.

```bash
systemctl status vmtoolsd --no-pager
systemctl status sshd --no-pager
```

`vmtoolsd`가 정상적으로 실행 중이어야 vSphere의 Guest Customization과 사용자 지정 스크립트가 정상적으로 동작합니다.


---


## 18. 확장 설계: vSphere Template 기반 Ansible 접속 자동화

현재 1차 버전에서는 관리 대상 서버에 `ansible` 계정을 만들고, Ansible Control 서버에서 `ssh-copy-id`를 통해 SSH 공개키를 복사하는 방식으로 구성했습니다.

```bash
ssh-copy-id ansible@linux-node1
ssh-copy-id ansible@linux-node2
```

이 방식은 소규모 실습 환경에서는 충분히 단순하고 직관적입니다.
하지만 관리 대상 서버가 20대, 50대 이상으로 늘어나면 각 서버마다 수동으로 `ssh-copy-id`를 반복해야 하므로 운영 효율성이 떨어집니다.

따라서 실제 운영 환경을 고려하면 다음과 같은 방식으로 확장할 수 있습니다.

```text
1차 버전:
  VM 생성 후 ansible 계정 생성
  ssh-copy-id로 각 서버에 공개키 복사
  Ansible Playbook 실행

확장 버전:
  vSphere Template 또는 사용자 지정 스크립트에서
  ansible 계정, SSH 공개키, sudoers 설정을 자동 구성
  VM 배포 직후부터 Ansible 접속 가능
```

---

### 18.1 vSphere Template에 미리 포함할 수 있는 항목

vSphere Template을 사용할 경우 템플릿 VM에 다음 항목을 미리 구성해 둘 수 있습니다.

```text
- ansible 계정 생성
- /home/ansible/.ssh/authorized_keys 기본 구성
- /etc/sudoers.d/ansible 파일 생성
- sshd_config의 Match User ansible 정책 설정
```

이렇게 구성한 뒤 템플릿으로 VM을 배포하면, 배포된 VM은 처음부터 Ansible Control 서버에서 접속 가능한 상태가 됩니다.

다만 `authorized_keys`에 `from="컨트롤서버IP"` 제한을 넣는 경우에는 주의가 필요합니다.
Ansible Control 서버의 IP가 고정되어 있다면 템플릿에 미리 넣어도 되지만, 환경마다 Control 서버 IP가 달라질 수 있다면 배포 시점의 사용자 지정 스크립트에서 동적으로 적용하는 방식이 더 적합합니다.

---

### 18.2 사용자 지정 스크립트를 사용하는 방식

vSphere의 VM 사용자 지정 규격에서는 사용자 지정 전/후 스크립트를 사용할 수 있습니다.

기본 구조는 다음과 같습니다.

```sh
#!/bin/sh

if [ "x$1" = x"precustomization" ]; then
  echo "사용자 지정 전 작업 수행"

elif [ "x$1" = x"postcustomization" ]; then
  echo "사용자 지정 후 작업 수행"

fi
```

여기서 Ansible 접속 계정 생성, SSH 공개키 등록, sudoers 설정은 `postcustomization` 단계에 넣는 것이 적절합니다.

```text
precustomization:
  hostname, IP, DNS 등 사용자 지정 전 단계

postcustomization:
  hostname/IP 설정 이후 단계
  계정 생성, SSH 키 등록, sudoers 설정에 적합
```

즉, VM의 hostname과 IP가 적용된 이후에 Ansible 접속 정책을 구성하는 흐름입니다.

---



### 18.3 사용자 지정 스크립트 예시

아래는 Rocky Linux 기준으로 사용할 수 있는 예시입니다.
`CONTROL_IP`와 `PUBKEY` 값은 환경에 맞게 변경해야 합니다.

```sh
#!/bin/sh

if [ "x$1" = x"postcustomization" ]; then
  CONTROL_IP="172.16.1.80"
  PUBKEY='ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIxxxxxxxxxxxxxxxxxxxxxxxx ansible-control'

  groupadd -f ansible
  id ansible >/dev/null 2>&1 || useradd -m -g ansible -s /bin/bash ansible

  mkdir -p /home/ansible/.ssh
  echo "from=\"${CONTROL_IP}\" ${PUBKEY}" > /home/ansible/.ssh/authorized_keys

  chown -R ansible:ansible /home/ansible/.ssh
  chmod 700 /home/ansible/.ssh
  chmod 600 /home/ansible/.ssh/authorized_keys

  echo '%ansible ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/ansible
  chmod 440 /etc/sudoers.d/ansible
  visudo -cf /etc/sudoers.d/ansible || exit 1

  grep -q 'Match User ansible' /etc/ssh/sshd_config || cat >> /etc/ssh/sshd_config <<'EOF'
# BEGIN ANSIBLE SERVICE USER POLICY
Match User ansible
    PasswordAuthentication no
    PubkeyAuthentication yes
# END ANSIBLE SERVICE USER POLICY
EOF

  sshd -t && systemctl restart sshd
fi
```

이 스크립트가 수행하는 작업은 다음과 같습니다.

| 작업                 | 설명                                    |
| ------------------ | ------------------------------------- |
| ansible 그룹 생성      | Ansible 전용 그룹 생성                      |
| ansible 계정 생성      | 관리 자동화 전용 서비스 계정 생성                   |
| authorized_keys 생성 | Control 서버의 공개키 등록                    |
| from 제한 적용         | 지정된 Control 서버 IP에서만 SSH 키 사용 가능      |
| sudoers 설정         | ansible 그룹에 NOPASSWD sudo 권한 부여       |
| SSH 정책 설정          | ansible 계정은 비밀번호 로그인을 차단하고 공개키 인증만 허용 |
| sshd 설정 검증         | `sshd -t`로 설정 문법 확인 후 재시작             |

---

### 18.4 authorized_keys의 from 제한

중요한 부분은 다음 줄입니다.

```sh
echo "from=\"${CONTROL_IP}\" ${PUBKEY}" > /home/ansible/.ssh/authorized_keys
```

관리 대상 서버의 `/home/ansible/.ssh/authorized_keys`에는 최종적으로 다음과 같은 형식으로 저장됩니다.

```text
from="172.16.1.80" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ansible-control
```

이 설정을 적용하면 SSH Key를 가지고 있더라도 지정된 Control 서버 IP에서만 `ansible` 계정으로 접속할 수 있습니다.

```text
Ansible Control Server: 172.16.1.80
  └── Ansible 전용 private key 보유

Managed Nodes:
  └── ansible 계정의 authorized_keys에 Control 서버 public key 등록
  └── from="172.16.1.80" 제한으로 Control 서버에서만 접속 허용
```

이를 통해 단순히 SSH Key를 등록하는 것보다 접속 경로를 더 제한할 수 있습니다.

---



### 18.5 템플릿 VM에 사전 준비할 항목

사용자 지정 스크립트 방식이 정상적으로 동작하려면 템플릿 VM에 최소한 다음 항목이 준비되어 있어야 합니다.

```text
1. open-vm-tools 설치
2. sshd 활성화
3. sudo 설치
4. NetworkManager 정상 동작
5. VM을 템플릿으로 변환하기 전 machine-id 정리
```

Rocky Linux 템플릿에서 미리 준비할 수 있는 명령은 다음과 같습니다.

```bash
sudo dnf install -y open-vm-tools perl sudo openssh-server
sudo systemctl enable --now vmtoolsd
sudo systemctl enable --now sshd
```

템플릿으로 변환하기 전에는 machine-id를 정리하여, 템플릿에서 복제된 VM들이 동일한 machine-id를 공유하지 않도록 합니다.

```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
```

필요하다면 로그도 정리할 수 있습니다.

```bash
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s
sudo rm -f /var/log/*.log
```

이후 VM을 종료하고 vSphere Template으로 변환하면 됩니다.

---

### 18.6 배포 후 확인 방법

템플릿에서 VM을 배포한 뒤 Ansible Control 서버에서 SSH 접속을 확인합니다.

```bash
ssh -i ~/.ssh/id_ed25519_ansible ansible@새VM_IP
```

sudo 권한도 확인합니다.

```bash
sudo whoami
```

기대 결과는 다음과 같습니다.

```text
root
```

Ansible 연결 테스트는 다음과 같이 수행합니다.

```bash
ansible -i inventory.ini all -m ping
```

become 권한까지 확인하려면 다음 명령을 사용할 수 있습니다.

```bash
ansible -i inventory.ini all -b -m command -a "whoami"
```

결과가 `root`로 출력되면 Ansible 접속 계정, SSH Key 인증, sudo 권한이 모두 정상적으로 구성된 것입니다.

---

### 18.7 방식 비교

| 방식                     | 설명                                 | 장점                     | 단점                        |
| ---------------------- | ---------------------------------- | ---------------------- | ------------------------- |
| 수동 ssh-copy-id         | VM 생성 후 각 서버에 공개키 복사               | 단순하고 이해하기 쉬움           | 서버 수가 늘어나면 반복 작업 증가       |
| Bootstrap Playbook     | 초기 접속 계정으로 ansible 계정/키/sudoers 배포 | 기존 서버에도 적용 가능          | 최초 접속 가능한 계정 필요           |
| vSphere Template 사전 구성 | 템플릿 안에 ansible 계정/키/sudoers 포함     | VM 배포 즉시 Ansible 접속 가능 | Control 서버 IP가 바뀌면 관리 어려움 |
| vSphere 사용자 지정 스크립트    | VM 배포 시점에 계정/키/sudoers 생성          | 배포 환경에 따라 동적 적용 가능     | 스크립트 관리 필요                |

현재 1차 버전에서는 `ssh-copy-id`를 사용했지만, 서버 규모가 커질수록 Bootstrap Playbook이나 vSphere 사용자 지정 스크립트 방식이 더 적합합니다.

최종적으로 다음과 같은 구조입니다.

```text
vSphere Template
  └── 공통 OS 설정, open-vm-tools, sshd, sudo, NetworkManager 준비

VM Customization Spec
  └── hostname/IP 적용
  └── postcustomization 스크립트로 ansible 계정/키/sudoers 설정

Ansible
  └── 배포된 VM에 접속
  └── 사용자, 패키지, 방화벽, 서비스, 보안 설정 자동화
```

이 구조를 사용하면 VM을 새로 배포하는 순간부터 Ansible이 접속 가능한 상태가 되며, 이후 서버 초기 설정은 Playbook으로 일관되게 관리할 수 있습니다.



## 19. 프로젝트 의의

이 프로젝트는 단순히 Linux 명령어를 수동으로 실행하는 수준을 넘어, 서버 초기 설정 작업을 Ansible로 표준화하고 자동화했다는 점에 의미가 있습니다.

특히 RHCSA/Linux 학습에서 다루는 사용자 관리, sudo 권한, 패키지 설치, systemd 서비스 관리, 방화벽, SELinux, SSH 설정 등을 실제 서버 자동화 흐름으로 연결했습니다.

이를 통해 Linux 서버 운영 환경에서 반복적인 초기 설정 작업을 자동화하고, 여러 서버에 일관된 설정을 적용하는 기본적인 Infrastructure Automation 역량을 학습할 수 있었습니다.

---

