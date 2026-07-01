# 오픈소스 로드밸런싱 시스템 구축 실무 가이드
> VirtualBox + Ubuntu 22.04 + HAProxy 2.8 기반  
> Windows 환경에서 처음부터 끝까지 따라하는 순서

---

## 사전 준비

### 1. VirtualBox 설치
- 다운로드: https://www.virtualbox.org/wiki/Downloads
- 권장 버전: 7.2.10 이상
- 설치 시 기본 옵션 그대로 진행

### 2. Ubuntu 22.04 Server ISO 다운로드
- 다운로드: https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
- 파일명: `ubuntu-22.04.5-live-server-amd64.iso` (약 2.1GB)
- Desktop 버전 아닌 **Server** 버전 사용 (GUI 없음, 리소스 절약)

---

## 구성 개요

```
[Windows 호스트]
       │
       │ Host-Only 네트워크 (192.168.56.0/24)
       │
┌──────┴──────────────────────┐
│        HAProxy VM           │
│  IP: 192.168.56.10          │  ← 로드밸런서 (클라이언트 진입점)
│  NAT + Host-Only 어댑터 2개 │
└──────┬──────────────────────┘
       │ 트래픽 분산
    ┌──┴───┐
    ▼       ▼
┌────────┐ ┌────────┐
│ web-01 │ │ web-02 │
│.56.11  │ │.56.12  │  ← 백엔드 웹 서버
└────────┘ └────────┘
```

### VM 사양 요약

| VM 이름    | 역할         | CPU | RAM    | Disk | IP             |
|------------|--------------|-----|--------|------|----------------|
| haproxy-lb | 로드밸런서   | 2   | 2048MB | 20GB | 192.168.56.10  |
| web-01     | 백엔드 서버  | 1   | 1024MB | 20GB | 192.168.56.11  |
| web-02     | 백엔드 서버  | 1   | 1024MB | 20GB | 192.168.56.12  |

---

## STEP 1 — Host-Only 네트워크 확인

PowerShell을 **관리자 권한**으로 열고 실행:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" list hostonlyifs
```

- `IPAddress: 192.168.56.1` 항목이 있으면 그대로 사용
- 없으면 아래 명령어로 생성:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" hostonlyif create
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" hostonlyif ipconfig "VirtualBox Host-Only Ethernet Adapter" --ip 192.168.56.1 --netmask 255.255.255.0
```

---

## STEP 2 — VM 생성 (3대 공통 절차)

VirtualBox 메인 화면 → **새로 만들기**

### haproxy-lb

| 항목 | 값 |
|---|---|
| 이름 | haproxy-lb |
| ISO 이미지 | ubuntu-22.04.5-live-server-amd64.iso |
| 유형 | Linux / Ubuntu (64-bit) |
| 메모리 | 2048 MB |
| CPU | 2 |
| 디스크 | 20 GB |

> "자동 설치" 체크해제

**네트워크 설정** (VM 선택 → 설정 → 네트워크):
```
어댑터 1: NAT              ← 인터넷(패키지 설치)용
어댑터 2: 호스트 전용 어댑터 (VirtualBox Host-Only Ethernet Adapter)
```

### web-01

| 항목 | 값 |
|---|---|
| 이름 | web-01 |
| ISO 이미지 | ubuntu-22.04.5-live-server-amd64.iso |
| 유형 | Linux / Ubuntu (64-bit) |
| 메모리 | 1024 MB |
| CPU | 1 |
| 디스크 | 20 GB |

**네트워크 설정:**
```
어댑터 1: NAT
어댑터 2: 호스트 전용 어댑터 (VirtualBox Host-Only Ethernet Adapter)
```

### web-02

web-01과 동일한 사양, 이름만 `web-02`로 변경

---

## STEP 3 — Ubuntu 22.04 Server 설치 (VM 3대 공통)

VM 시작 후 설치 화면에서 순서대로 진행:

```
1. Try or Install Ubuntu Server   → Enter
2. 언어: English                  → Enter
3. Installer update: Continue without updating → Enter
4. Keyboard: English (US)         → Done
5. Install type: Ubuntu Server    → Done
6. Network: 기본값 확인           → Done
7. Proxy: 공백                    → Done
8. Mirror: 기본값                 → Done
9. Storage: Use an entire disk    → Done → Continue
10. Profile setup:
      Your name:    admin
      Server name:  haproxy-lb  (각 VM마다 다르게)
      Username:     admin
      Password:     (설정)
11. SSH: Install OpenSSH server 체크 → Done
12. Featured snaps: 선택 없이    → Done
13. 설치 완료 → Reboot Now
```

> Server name은 VM마다 다르게 설정:
> - haproxy-lb VM → `haproxy-lb`
> - web-01 VM → `web-01`
> - web-02 VM → `web-02`

---

## STEP 4 — 고정 IP 설정 (VM 3대 각각)

재부팅 후 로그인 → 아래 명령어 실행

### haproxy-lb (192.168.56.10)

```bash
# 네트워크 인터페이스 이름 확인
ip a
# enp0s3 (NAT), enp0s8 (Host-Only) 확인

sudo nano /etc/netplan/00-installer-config.yaml
```

아래 내용으로 교체:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true          # NAT - 인터넷용 (자동)
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.56.10/24  # Host-Only 고정 IP
```

```bash
sudo netplan apply
ip a  # 192.168.56.10 할당 확인
```

### web-01 (192.168.56.11)

동일하게 진행, enp0s8 주소만 변경:
```yaml
      addresses:
        - 192.168.56.11/24
```

### web-02 (192.168.56.12)

```yaml
      addresses:
        - 192.168.56.12/24
```

---

## STEP 5 — 백엔드 웹 서버 설치 (web-01, web-02)

```bash
sudo apt update && sudo apt install -y nginx

# 어느 서버가 응답하는지 식별용 페이지 생성
echo "<h1>Backend: $(hostname)</h1>" | sudo tee /var/www/html/index.html

# 헬스체크 엔드포인트 생성
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

sudo nginx -t
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

## STEP 6 — HAProxy 설치 및 설정 (haproxy-lb)

### 설치

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:vbernat/haproxy-2.8
sudo apt update
sudo apt install -y haproxy=2.8.*

# 버전 확인
haproxy -v
```

### 설정 파일 작성

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo tee /etc/haproxy/haproxy.cfg <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 20000

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect  5s
    timeout client   30s
    timeout server   30s
    timeout http-request 10s

frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:changeme123

frontend http_front
    bind *:80
    default_backend web_servers
    option forwardfor

backend web_servers
    balance roundrobin
    option httpchk GET /health HTTP/1.0
    http-check expect status 200
    server web-01 192.168.56.11:80 check inter 2s rise 2 fall 3
    server web-02 192.168.56.12:80 check inter 2s rise 2 fall 3
EOF
```

### 기동

```bash
# 설정 파일 문법 검사
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# 서비스 시작
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```

---

## STEP 7 — 동작 검증

### 부하 분산 확인

Windows PowerShell에서 실행:

```powershell
# 6회 요청 → web-01, web-02 교대 응답 확인
for ($i=1; $i -le 6; $i++) {
    Invoke-WebRequest -Uri "http://192.168.56.10" -UseBasicParsing | Select-Object -ExpandProperty Content
}
```

또는 haproxy-lb VM에서:

```bash
for i in $(seq 1 6); do curl -s http://192.168.56.10/ ; done
```

예상 출력:
```
<h1>Backend: web-01</h1>
<h1>Backend: web-02</h1>
<h1>Backend: web-01</h1>
...
```

### Stats 페이지 확인

Windows 브라우저에서:
```
http://192.168.56.10:8404/stats
ID: admin / PW: changeme123
```

### 장애 복구 테스트

```bash
# web-01에서 Nginx 중단
sudo systemctl stop nginx

# haproxy-lb에서 확인 (약 6초 후 web-01 DOWN 처리)
echo "show servers state" | sudo socat stdio /run/haproxy/admin.sock

# 요청이 web-02로만 가는지 확인
for i in $(seq 1 4); do curl -s http://192.168.56.10/; done

# web-01 복구
sudo systemctl start nginx
```

---

## 완료 체크리스트

- [x] VirtualBox 설치 및 Host-Only 네트워크 확인
- [x] VM 3대 생성 (haproxy-lb, web-01, web-02)
- [x] Ubuntu 22.04 Server 설치 완료 (3대)
- [x] 고정 IP 설정 완료 (3대)
- [x] Nginx 설치 및 헬스체크 엔드포인트 구성 (web-01, web-02)
- [x] HAProxy 설치 및 설정 완료 (haproxy-lb)
- [x] 부하 분산 동작 확인
- [x] 장애 복구 동작 확인
- [x] Stats 페이지 접근 확인
