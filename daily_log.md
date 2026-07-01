# 프로젝트 일일 업무 일지

---

## 2026-06-30

### 작업 목표
가상화 환경 기반 오픈소스 로드밸런싱 시스템 구축 착수

### 수행 업무

#### 1. 기술 검토 및 문서화
- 오픈소스 로드밸런서 비교 분석 (HAProxy / Nginx / Traefik)
- **HAProxy 2.8 LTS** 채택 결정
  - 근거: OSS 버전에서 능동 헬스체크 기본 제공, Runtime API로 무중단 서버 교체 가능, VM 리소스 대비 메모리 효율 우수
- 기술 가이드 문서 작성 → `load.md`
- 실무 구축 절차 문서 작성 → `setup_guide.md`

#### 2. 환경 구성
- 가상화 도구: VirtualBox 7.2.10 (기 설치)
- Host-Only 네트워크 확인 (192.168.56.1/24 기존 어댑터 활용)
- Ubuntu 22.04.5 Server ISO 다운로드 완료

#### 3. VM 구성 설계 확정

| VM | 역할 | CPU | RAM | IP |
|---|---|---|---|---|
| haproxy-lb | 로드밸런서 | 2 | 2GB | 192.168.56.10 |
| web-01 | 백엔드 웹서버 | 1 | 1GB | 192.168.56.11 |
| web-02 | 백엔드 웹서버 | 1 | 1GB | 192.168.56.12 |

#### 4. VM 생성 및 Ubuntu 설치 착수
- haproxy-lb VM 생성 완료 (네트워크 어댑터 2개 설정: NAT + Host-Only)
- Ubuntu 22.04 Server 수동 설치 진행 중 (Unattended Installation 비활성화)
- 초기 설치 시도에서 자동 설치로 진행되는 문제 발생 → VM 삭제 후 재생성
  - 원인: VirtualBox 7.x에서 "Proceed with Unattended Installation" 옵션 기본 활성화
  - 조치: VM 생성 시 해당 옵션 체크 해제 후 재설치

### 진행 현황

| 단계 | 상태 |
|---|---|
| 기술 문서 작성 (load.md) | 완료 |
| 구축 가이드 작성 (setup_guide.md) | 완료 |
| Host-Only 네트워크 설정 | 완료 |
| haproxy-lb VM 생성 | 완료 |
| haproxy-lb Ubuntu 설치 | 완료 (vboxuser 자동설치) |
| web-01 VM 생성 및 설치 | 대기 |
| web-02 VM 생성 및 설치 | 대기 |
| 고정 IP 설정 (3대) | 대기 |
| HAProxy 설치 및 설정 | 대기 |
| Nginx 설치 (web-01, web-02) | 대기 |
| 부하 분산 동작 검증 | 대기 |

### 내일 예정 작업
- haproxy-lb enp0s8 고정 IP 설정 (192.168.56.10)
- web-01, web-02 VM 생성 및 Ubuntu 설치
- HAProxy 2.8 설치 및 haproxy.cfg 설정
- Nginx 설치 및 헬스체크 엔드포인트 구성
- 부하 분산 및 장애 복구 동작 검증

---

## 2026-07-01

### 작업 목표
VM 3대 네트워크 설정 완료 및 HAProxy 로드밸런싱 구축 완성

### 수행 업무

#### 1. haproxy-lb SSH 접속 설정
- OpenSSH가 자동설치에 포함되지 않아 수동 설치: `sudo apt install -y openssh-server`
- 포트포워딩 Host:2222 → Guest:22 설정 후 SSH 접속 성공

#### 2. haproxy-lb 고정 IP 설정
- `/etc/netplan/00-installer-config.yaml` 작성
- enp0s8에 192.168.56.10/24 할당 완료

#### 3. web-01, web-02 VM 구성
- Ubuntu 수동 설치 시 키보드 freeze 및 미러 오류 반복 발생
- haproxy-lb를 완전 복제(Full Clone)하는 방식으로 전환
- 복제 후 각 VM에서 hostname 및 netplan IP 변경:
  - web-01: hostname=web-01, IP=192.168.56.11
  - web-02: hostname=web-02, IP=192.168.56.12
- SSH 포트포워딩: web-01=2223, web-02=2224

#### 4. Nginx 설치 (web-01, web-02)
- nginx 설치 후 `/etc/nginx/sites-enabled/default` 삭제 필요 (default_server 충돌)
- `/etc/nginx/conf.d/health.conf` 작성으로 /health 엔드포인트 구성
- 헬스체크 응답 확인: curl http://192.168.56.11/health → OK

#### 5. HAProxy 2.8 설치 및 설정
- PPA 통해 HAProxy 2.8.25 설치
- `option httpchk GET /health HTTP/1.1` → HTTP/1.1은 Host 헤더 필수라 nginx 거부
- `option httpchk GET /health HTTP/1.0` 으로 변경 후 헬스체크 정상 동작

#### 6. 동작 검증 완료
- 부하 분산: web-01, web-02 라운드로빈 교대 응답 확인
- 장애 복구: 서버 1대 중단 시 나머지 서버로 자동 전환 확인
- Stats 페이지: http://192.168.56.10:8404/stats 접근 확인

### 트러블슈팅 기록

| 문제 | 원인 | 해결 |
|---|---|---|
| SSH 접속 불가 | 자동설치에 OpenSSH 미포함 | apt install openssh-server |
| Ubuntu 설치 키보드 freeze | VirtualBox Profile 화면 버그 | haproxy-lb 복제 방식으로 전환 |
| nginx /health 404 | sites-enabled/default 충돌 | default 삭제 후 health.conf 적용 |
| HAProxy 헬스체크 실패 | HTTP/1.1 Host 헤더 누락 | HTTP/1.0으로 변경 |

### 진행 현황

| 단계 | 상태 |
|---|---|
| 기술 문서 작성 (load.md) | 완료 |
| 구축 가이드 작성 (setup_guide.md) | 완료 |
| Host-Only 네트워크 설정 | 완료 |
| VM 3대 구성 및 IP 설정 | 완료 |
| Nginx 설치 및 헬스체크 구성 | 완료 |
| HAProxy 설치 및 설정 | 완료 |
| 부하 분산 동작 검증 | 완료 |
| 장애 복구 동작 검증 | 완료 |
| Stats 페이지 확인 | 완료 |

**전체 구축 완료**
