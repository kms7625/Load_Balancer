# 가상화 환경 기반 오픈소스 로드밸런싱 시스템 구축 가이드

---

## 1. 오픈소스 로드밸런서 추천 및 비교

### 비교표

| 항목 | HAProxy | Nginx (OSS) | Traefik |
|---|---|---|---|
| L4/L7 지원 | L4 + L7 | L7 (L4는 stream 모듈) | L7 중심 |
| 처리 성능 | 최상 (이벤트 기반 단일 프로세스) | 상 | 중 |
| 헬스체크 | 능동/수동 모두 지원 | 수동 중심 (active는 Plus) | 능동 지원 |
| 세션 퍼시스턴스 | stick-table, cookie 등 다양 | ip_hash, sticky cookie | 제한적 |
| 실시간 통계 | 내장 Stats 페이지, Prometheus | 별도 구성 필요 | 내장 dashboard |
| 자동화 연계 | Runtime API, Dataplne API | reload 기반 | REST API |
| 학습 난이도 | 중 | 하 | 하 |
| VM 환경 적합성 | **최상** | 상 | 중 |

### 선정 결론: **HAProxy 2.8 LTS**

**선정 이유:**

- **성능:** 이벤트 루프 기반 아키텍처로 커넥션 수만 개를 단일 프로세스로 처리. VM 리소스 제약 환경에서 Nginx 대비 메모리 효율이 높음.
- **헬스체크:** 능동(active) 헬스체크가 OSS 버전에서 기본 제공됨. Nginx OSS는 수동(passive)만 지원하여 장애 감지 지연 발생.
- **자동화 연계:** Runtime API로 재시작 없이 백엔드 서버를 동적으로 추가/제거 가능. Ansible과 조합 시 무중단 운영 구현에 유리.
- **검증된 사례:** AWS ELB, GitHub, Cloudflare 등이 내부적으로 사용하는 업계 표준.

> **Nginx를 선택해야 하는 경우:** 웹 서버(정적 파일 서빙)와 리버스 프록시 기능을 하나의 프로세스에서 함께 처리해야 할 때. 순수 로드밸런서 역할만이라면 HAProxy가 우위.

---

## 2. VM 가상화 환경 세팅 가이드

### 2-1. 가상화 도구 선정

| 도구 | 권장 사용 환경 | 비고 |
|---|---|---|
| **KVM + libvirt** | Linux 호스트, 프로덕션 검증용 | 하이퍼바이저 오버헤드 최소, Terraform libvirt provider 지원 |
| **VirtualBox** | Windows/macOS 개발 로컬 테스트 | 무료, GUI 편의성 높음. 성능은 KVM 대비 열세 |
| **VMware Workstation/ESXi** | 엔터프라이즈 기존 인프라 | 라이선스 비용 발생, 안정성 검증됨 |

**실무 권장:** 개발/검증 단계는 VirtualBox + Vagrant, 스테이징/운영은 KVM + libvirt.

---

### 2-2. VM 구성 사양

#### 로드밸런서 VM (HAProxy)

| 항목 | 권장 사양 | 비고 |
|---|---|---|
| OS | Ubuntu 22.04 LTS | 장기 지원, HAProxy 2.8 패키지 공식 지원 |
| CPU | 2 vCPU | HAProxy는 단일 프로세스 → 고클럭 선호 |
| RAM | 2 GB | stats 페이지 + stick-table 고려 시 여유 확보 |
| Disk | 20 GB | 로그 로테이션 포함 |
| NIC | 2개 (외부망 + 내부망) | 역할 분리 필수 |

#### 백엔드 웹 서버 VM (Nginx/Apache) × 2~3대

| 항목 | 권장 사양 |
|---|---|
| OS | Ubuntu 22.04 LTS |
| CPU | 1~2 vCPU |
| RAM | 1~2 GB |
| Disk | 20 GB |
| NIC | 1개 (내부망만) |

---

### 2-3. 네트워크 토폴로지

```
[클라이언트]
     │
     │  (Public Network / NAT or Bridged)
     ▼
┌─────────────────────┐
│   HAProxy VM        │  eth0: 192.168.56.10  ← 클라이언트 접점
│   (Load Balancer)   │  eth1: 10.0.0.10      ← 백엔드 통신 전용
└────────┬────────────┘
         │  (Internal Host-Only Network: 10.0.0.0/24)
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌────────┐
│ Web-1  │ │ Web-2  │
│10.0.0.11│ │10.0.0.12│
└────────┘ └────────┘
```

#### VirtualBox 네트워크 모드 설정

```
HAProxy VM:
  - 어댑터 1: NAT 또는 브리지 (외부 접근용, 192.168.56.10)
  - 어댑터 2: Host-Only (내부망, 10.0.0.10)

Web-1, Web-2 VM:
  - 어댑터 1: Host-Only (내부망만, 10.0.0.11 / 10.0.0.12)
```

> **Host-Only 선택 이유:** 백엔드 서버를 외부 네트워크와 완전히 격리. 모든 외부 트래픽이 반드시 HAProxy를 경유하도록 강제.

#### KVM 사용 시 네트워크 구성

```bash
# 내부 전용 브리지 생성
sudo virsh net-define internal-net.xml
sudo virsh net-start internal-net
sudo virsh net-autostart internal-net
```

```xml
<!-- internal-net.xml -->
<network>
  <name>internal</name>
  <bridge name='virbr1'/>
  <ip address='10.0.0.1' netmask='255.255.255.0'/>
</network>
```

---

### 2-4. 각 VM 초기 설정 (Ubuntu 22.04 기준)

```bash
# 호스트명 설정 (각 VM에서 실행)
sudo hostnamectl set-hostname haproxy-lb   # 로드밸런서
sudo hostnamectl set-hostname web-01       # 백엔드 1
sudo hostnamectl set-hostname web-02       # 백엔드 2

# /etc/hosts 등록 (HAProxy VM)
sudo tee -a /etc/hosts <<EOF
10.0.0.11 web-01
10.0.0.12 web-02
EOF

# 패키지 최신화
sudo apt update && sudo apt upgrade -y

# 방화벽 설정 (HAProxy VM)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8404/tcp   # HAProxy Stats 포트
sudo ufw enable
```

---

## 3. 로드밸런싱 구축 및 설정 절차

### 3-1. HAProxy 설치

```bash
# HAProxy 공식 PPA 사용 (Ubuntu 기본 repo는 구버전 포함 위험)
sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:vbernat/haproxy-2.8
sudo apt update
sudo apt install -y haproxy=2.8.*

# 버전 확인
haproxy -v
# HAProxy version 2.8.x ...
```

### 3-2. 백엔드 웹 서버 설치 (Web-01, Web-02)

```bash
sudo apt install -y nginx

# 식별용 index 페이지 생성 (테스트 시 분산 확인용)
echo "<h1>Backend Server: $(hostname)</h1>" | sudo tee /var/www/html/index.html

sudo systemctl enable nginx
sudo systemctl start nginx
```

---

### 3-3. HAProxy 핵심 설정 파일

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo tee /etc/haproxy/haproxy.cfg <<'EOF'
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # 최대 커넥션 수 (RAM 2GB 기준 현실적인 값)
    maxconn 20000

    # TLS 보안 설정 (HTTPS 구성 시)
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

#---------------------------------------------------------------------
# Default settings (모든 frontend/backend에 상속)
#---------------------------------------------------------------------
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect  5s      # 백엔드 연결 타임아웃
    timeout client   30s     # 클라이언트 비활성 타임아웃
    timeout server   30s     # 백엔드 응답 타임아웃
    timeout http-request 10s # HTTP 요청 수신 완료 타임아웃 (SlowLoris 방어)
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 503 /etc/haproxy/errors/503.http

#---------------------------------------------------------------------
# Stats 페이지 (모니터링용 — 프로덕션에서는 IP 제한 필수)
#---------------------------------------------------------------------
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:changeme123      # 반드시 변경
    stats admin if TRUE               # Runtime으로 서버 조작 허용

#---------------------------------------------------------------------
# Frontend: 클라이언트 진입점
#---------------------------------------------------------------------
frontend http_front
    bind *:80
    default_backend web_servers

    # X-Forwarded-For 헤더 추가 (백엔드가 실제 클라이언트 IP 인식)
    option forwardfor
    http-request set-header X-Forwarded-Proto http

#---------------------------------------------------------------------
# Backend: 웹 서버 풀
#---------------------------------------------------------------------
backend web_servers
    balance roundrobin        # 라운드로빈 알고리즘

    # 능동 헬스체크: 2초마다 /health 엔드포인트 확인
    # 2회 연속 성공 시 복구(rise), 3회 연속 실패 시 제거(fall)
    option httpchk GET /health HTTP/1.1\r\nHost:\ localhost
    http-check expect status 200

    # 쿠키 기반 세션 퍼시스턴스 (stateful 앱 필요 시 활성화)
    # cookie SERVERID insert indirect nocache

    server web-01 10.0.0.11:80 check inter 2s rise 2 fall 3 weight 1
    server web-02 10.0.0.12:80 check inter 2s rise 2 fall 3 weight 1

    # Backup 서버: 모든 active 서버 다운 시에만 사용
    # server web-backup 10.0.0.13:80 check backup

    # 세션 퍼시스턴스 사용 시 아래 주석 해제
    # server web-01 10.0.0.11:80 check inter 2s rise 2 fall 3 cookie web01
    # server web-02 10.0.0.12:80 check inter 2s rise 2 fall 3 cookie web02
EOF
```

---

### 3-4. 헬스체크 엔드포인트 구성 (백엔드 서버)

```bash
# Web-01, Web-02 모두 실행
sudo tee /var/www/html/health <<'EOF'
OK
EOF

# Nginx location 블록 추가 (별도 상태 파일로 분리)
sudo tee /etc/nginx/conf.d/health.conf <<'EOF'
server {
    listen 80 default_server;
    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
    location / {
        root /var/www/html;
        index index.html;
    }
}
EOF

sudo nginx -t && sudo systemctl reload nginx
```

---

### 3-5. HAProxy 기동 및 검증

```bash
# 설정 파일 문법 검사 (기동 전 반드시 실행)
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# 서비스 시작
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```

---

### 3-6. 부하 분산 동작 검증

#### 기본 분산 확인

```bash
# HAProxy IP로 반복 요청 → 응답이 web-01, web-02 교대로 반환되는지 확인
for i in $(seq 1 6); do
  curl -s http://192.168.56.10/ | grep -o "Backend Server:.*"
done

# 예상 출력:
# Backend Server: web-01
# Backend Server: web-02
# Backend Server: web-01
# Backend Server: web-02
# Backend Server: web-01
# Backend Server: web-02
```

#### 헬스체크 및 장애 복구 확인

```bash
# Web-01에서 Nginx 중단 (장애 시뮬레이션)
# Web-01 VM에서 실행:
sudo systemctl stop nginx

# HAProxy VM에서 상태 확인 (약 6초 후 web-01이 DOWN으로 전환됨)
echo "show servers state" | sudo socat stdio /run/haproxy/admin.sock

# 트래픽이 web-02로만 전달되는지 확인
for i in $(seq 1 4); do curl -s http://192.168.56.10/ | grep "Backend Server"; done

# Web-01 복구
sudo systemctl start nginx

# rise 2 조건 충족 후 web-01 자동 복귀 확인
sleep 5
echo "show servers state" | sudo socat stdio /run/haproxy/admin.sock
```

#### 부하 테스트 (ab 또는 wrk 사용)

```bash
# ab (Apache Bench) 설치
sudo apt install -y apache2-utils

# 총 1000 요청, 동시 10 커넥션
ab -n 1000 -c 10 http://192.168.56.10/

# 결과에서 확인할 항목:
# - Requests per second (처리량)
# - Failed requests: 0 (분산 중 오류 없어야 함)
# - Time per request (응답 지연)

# wrk 사용 시 (더 정밀한 측정)
# sudo apt install -y wrk
# wrk -t4 -c100 -d30s http://192.168.56.10/
```

#### Stats 페이지 접근

```
브라우저: http://192.168.56.10:8404/stats
ID: admin / PW: changeme123

확인 항목:
- Status: UP/DOWN
- Sessions: 현재 세션 수
- Bytes in/out: 트래픽 통계
- Check: 헬스체크 결과 및 응답시간
```

#### Runtime API로 서버 동적 제어

```bash
# web-01을 무중단으로 드레이닝 (배포 시 활용)
echo "set server web_servers/web-01 state drain" | \
  sudo socat stdio /run/haproxy/admin.sock

# 완전 비활성화
echo "set server web_servers/web-01 state maint" | \
  sudo socat stdio /run/haproxy/admin.sock

# 복구
echo "set server web_servers/web-01 state ready" | \
  sudo socat stdio /run/haproxy/admin.sock
```

---

## 4. 자동화 전환을 위한 향후 확장 팁

### 4-1. Ansible 연계 핵심 포인트

#### 디렉토리 구조 권장안

```
ansible/
├── inventory/
│   ├── hosts.ini          # 정적 인벤토리 (초기 단계)
│   └── hosts.yml          # 동적 인벤토리로 전환 예정
├── roles/
│   ├── haproxy/
│   │   ├── tasks/main.yml
│   │   ├── templates/haproxy.cfg.j2   # Jinja2 템플릿
│   │   └── handlers/main.yml          # reload 핸들러
│   └── webserver/
│       ├── tasks/main.yml
│       └── templates/nginx.conf.j2
├── playbooks/
│   ├── site.yml
│   ├── deploy_lb.yml
│   └── rolling_update.yml
└── group_vars/
    ├── all.yml
    └── lb.yml
```

#### haproxy.cfg Jinja2 템플릿화 핵심

```jinja2
# roles/haproxy/templates/haproxy.cfg.j2
backend web_servers
    balance {{ haproxy_balance_algorithm | default('roundrobin') }}
    option httpchk GET {{ haproxy_health_check_uri | default('/health') }}

{% for server in groups['webservers'] %}
    server {{ hostvars[server]['inventory_hostname'] }} \
      {{ hostvars[server]['ansible_host'] }}:{{ webserver_port | default(80) }} \
      check inter {{ haproxy_check_interval | default('2s') }} \
      rise {{ haproxy_rise | default(2) }} \
      fall {{ haproxy_fall | default(3) }}
{% endfor %}
```

#### 주의사항

- **HAProxy reload는 재시작이 아님:** `systemctl reload haproxy`는 무중단. Ansible handler에서 `notify: reload haproxy` 사용 시 `service` 모듈의 `state: reloaded` 사용.
- **설정 검증 선행:** Ansible task에서 `haproxy -c -f` 검증 후 reload하도록 순서 보장.
- **Runtime API vs reload:** 단순 서버 추가/제거는 Runtime API(`community.general.haproxy` 모듈)로 처리하면 reload 자체를 생략 가능. 설정 구조 변경 시에만 reload.

```yaml
# Rolling update 예시 (Ansible)
- name: Drain backend before deploy
  community.general.haproxy:
    state: drain
    host: "{{ inventory_hostname }}"
    socket: /run/haproxy/admin.sock
    backend: web_servers
  delegate_to: haproxy-lb

- name: Deploy application
  # ... 배포 작업 ...

- name: Re-enable backend after deploy
  community.general.haproxy:
    state: enabled
    host: "{{ inventory_hostname }}"
    socket: /run/haproxy/admin.sock
    backend: web_servers
  delegate_to: haproxy-lb
```

---

### 4-2. Terraform 연계 핵심 포인트

#### KVM 환경 (libvirt provider)

```hcl
# main.tf
terraform {
  required_providers {
    libvirt = {
      source  = "dmacvicar/libvirt"
      version = "~> 0.7"
    }
  }
}

provider "libvirt" {
  uri = "qemu:///system"
}

resource "libvirt_volume" "haproxy_disk" {
  name           = "haproxy-lb.qcow2"
  pool           = "default"
  source         = "https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img"
  format         = "qcow2"
}

resource "libvirt_domain" "haproxy" {
  name   = "haproxy-lb"
  memory = 2048
  vcpu   = 2

  network_interface {
    network_name   = "default"
    wait_for_lease = true
  }

  network_interface {
    network_name = "internal"
  }

  disk {
    volume_id = libvirt_volume.haproxy_disk.id
  }

  cloudinit = libvirt_cloudinit_disk.haproxy_init.id
}

resource "libvirt_domain" "webserver" {
  count  = var.webserver_count   # 변수로 대수 조정
  name   = "web-0${count.index + 1}"
  memory = 1024
  vcpu   = 1
  # ...
}
```

#### 주의사항

- **State 파일 관리:** `terraform.tfstate`에 VM 정보가 담기므로 S3/GCS 원격 backend 설정 필수. 로컬 state로 팀 협업 시 충돌 발생.
- **Cloud-init 활용:** VM 초기 사용자/SSH키/패키지 설치는 cloud-init으로 처리. Terraform은 인프라 프로비저닝만, 설정 관리는 Ansible에 위임하는 역할 분리 권장.
- **HAProxy 설정과 Terraform 분리:** Terraform은 VM/네트워크 생성에만 집중. `haproxy.cfg` 내용을 Terraform의 `templatefile()`로 생성하면 Terraform과 Ansible 책임 경계가 흐려짐. Terraform output(IP 주소 등)을 Ansible 동적 인벤토리가 읽는 구조로 분리.

```hcl
# outputs.tf — Ansible 동적 인벤토리 연동용
output "webserver_ips" {
  value = libvirt_domain.webserver[*].network_interface[0].addresses[0]
}

output "haproxy_ip" {
  value = libvirt_domain.haproxy.network_interface[0].addresses[0]
}
```

---

### 4-3. 모니터링 확장 (Prometheus + Grafana)

HAProxy 2.8은 Prometheus 메트릭 엔드포인트를 기본 내장.

```haproxy
# haproxy.cfg에 추가
frontend prometheus
    bind *:8405
    http-request use-service prometheus-exporter if { path /metrics }
    # 프로덕션에서는 반드시 IP 제한 추가
    # acl prometheus_src src 10.0.0.0/24
    # http-request deny unless prometheus_src
```

```yaml
# prometheus.yml scrape 설정
scrape_configs:
  - job_name: 'haproxy'
    static_configs:
      - targets: ['192.168.56.10:8405']
```

---

### 4-4. 전환 로드맵 (현실적인 단계)

```
1단계 (현재): 수동 구성
   VM 생성 → HAProxy 설치 → 설정 파일 직접 작성

2단계: Ansible 도입
   haproxy.cfg → Jinja2 템플릿
   Playbook으로 설치/설정/reload 자동화
   Rolling update playbook 작성

3단계: Terraform 도입
   VM 프로비저닝 코드화
   Terraform output → Ansible 동적 인벤토리 연동

4단계: CI/CD 통합
   Git push → GitHub Actions/Jenkins → Terraform plan/apply
   → Ansible playbook 실행 → 배포 완료
```

> **주의:** 2단계를 건너뛰고 1단계에서 바로 3단계로 가는 경우, HAProxy 설정 관리 자동화가 누락되어 인프라 코드와 실제 설정 간 드리프트가 발생하기 쉬움. Ansible 없이 Terraform만으로 설정 관리를 시도하면 책임 경계가 불명확해짐.
