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

#### 7. 구축 결과 보고서 작성 및 목적 확인
- 원본 작업 메모(`로드밸런싱 구축.txt`)를 근거로 사내 공유용 "구축 결과 보고서" 작성 → `build_report.md`
  - 구성: 개요 / 아키텍처 요약(구성도, VM 사양표, IP 체계표) / 사전 준비물 / STEP별 구축 절차 / 트러블슈팅 표 / 최종 검증 결과 / 향후 개선 제안
  - 원본 메모에 없는 내용은 추측 없이 "미확인·추가 검증 필요"로 표기 (예: web-01/web-02가 Full Clone된 이후 실제 CPU/RAM 값)
- 이번 구축 작업의 목적을 재확인
  - 회사 지시사항: 향후 인프라 교체를 전제로, 우선 로드밸런싱을 간단히 구축하고 그 과정을 가이드로 문서화
  - 확인 결과: 현재 VirtualBox 구성은 운영 인프라에 그대로 적용되는 것이 아니라, 실제 인프라 교체 설계 시 참고할 "1단계 개념검증(PoC) + 가이드" 역할로 회사 요구사항에 부합
  - 실제 교체 단계에서 별도 재검토가 필요한 항목(LB 자체 이중화, SSL 종단, 실 트래픽 규모에 맞는 사양, IaC 전환)을 보고서 "향후 개선 제안"에 명시해 둠

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
| 구축 결과 보고서 작성 (build_report.md) | 완료 |

**전체 구축 완료**

### 다음 예정 작업
- 상급자/회사에 `build_report.md` 공유 및 피드백 수렴
- 실제 인프라 교체 범위 확인 (대상 트래픽 규모, 실제 배포 환경이 클라우드/온프레미스 중 무엇인지)
- 필요 시 향후 개선 제안(LB 이중화, SSL, IaC 전환) 중 우선순위 항목 논의

---

## 2026-07-02

### 작업 목표
`로드밸런싱_구축_가이드.pdf` 최종 제출 후 리뷰 대비 준비 및 실제 동작 재검증, VMware Workstation 재구축 착수

### 수행 업무

#### 0. VMware Workstation 기반 재구축 착수
- 기존 VirtualBox 기반 구축(2026-07-01 완료)에서 **VMware Workstation 25.0.1**(build-25219725) 기반으로 전환 결정, 재구축 시작
- 호스트 환경 확인: `vmrun.exe`(VM 전원 제어), `vmnetcfg.exe`(가상 네트워크 편집기), `vnetlib64.exe`(네트워크 설정 조회) 등 CLI 도구 확인
- 가상 네트워크 어댑터 확인 (`Get-NetAdapter`/`Get-NetIPAddress` 기준): `VMnet8`(NAT, 호스트측 192.168.198.1/24), `VMnet1`(Host-only, 호스트측 192.168.164.1/24)
- Ubuntu 22.04.5 Server ISO 재사용 확인 (`C:\Users\ms.kang\Downloads\ubuntu-22.04.5-live-server-amd64.iso`)
- 미확인·추가 검증 필요: VMware 게스트 unattended install 방식, 계정 생성 방식, SSH 포트포워딩(VirtualBox처럼 개별 룰이 아니라 `vmnetnat.conf` 편집 방식으로 추정), 클론 후 정리 절차, guest NIC 명칭(enp0s3/enp0s8 유지 여부), haproxy-lb/web-01/web-02 실제 static IP(192.168.164.10~12 예정이나 게스트 설정 전이라 미확정)
- CLAUDE.md의 Build Status를 "진행 중"으로 갱신 — VirtualBox 기반 정보는 git 이력으로 보존

#### 1. 리뷰 준비 자료 작성 → `review_prep.md`
- 사용 오픈소스 목록 및 라이선스 정리 (HAProxy GPLv2, Nginx BSD-2-Clause, Ubuntu, VirtualBox 베이스 GPLv2/Extension Pack PUEL, socat GPLv2)
- 리뷰어가 지적할 가능성이 높은 지점 정리 (VM 사양 변경 미확인, Stats 페이지 기본 비밀번호, LB 이중화 미구축, IaC 미적용, raw 증적 미보존)
- 예상 Q&A 시나리오 및 리뷰 시연용 명령어 순서 정리

#### 2. HAProxy Stats 페이지(`:8404/stats`) 구조 재확인
- 헬스체크 결과는 `Check`가 아니라 `LastChk` 컬럼에 `L7OK/200 in Xms` 형태로 표시됨을 확인
- 페이지 자체는 한글화 불가(하드코딩된 고정 템플릿) — 필요 시 Prometheus+Grafana 연동으로 대체 가능하다고 정리

#### 3. haproxy-lb에서 장애 복구(Failover) 시연 명령어 재실행
- `for i in $(seq 1 4)` / `seq 1 6` 루프 및 `show servers state`로 재검증 진행
- 테스트 도중 일시적으로 `No server is available`(503) 및 예상과 다른 패턴(정지시킨 서버가 아니라 web-01만 연속 응답)이 관찰됨 → 어느 서버의 nginx를 멈췄는지 순서 확인이 필요한 상태로 남음(미확인)
- 최종 `show servers state` 조회 결과 web-01, web-02 모두 `srv_op_state=2`(UP)로 정상, 이후 `seq 1 6` 재실행 시 정상 교대 응답 확인 → 세션 종료 시점 기준 정상 상태

### 트러블슈팅 기록

| 문제 | 원인 | 해결 |
|---|---|---|
| Failover 재시연 중 `No server is available`(503) 및 예상과 다른 응답 패턴(web-01만 연속 응답) 발생 | 미확인 — nginx를 멈춘 서버와 명령 실행 순서 재확인 필요 | 최종적으로는 두 서버 모두 UP으로 정상 복귀 확인. 원인 규명은 다음 세션 과제로 이월 |

### 진행 현황

| 단계 | 상태 |
|---|---|
| 리뷰 준비 자료 작성 (review_prep.md) | 완료 |
| Stats 페이지 구조/헬스체크 컬럼 확인 | 완료 |
| Failover 시연 명령어 재검증 | 부분 완료 (결과는 정상, 중간 이상 패턴 원인 미확인) |
| VirtualBox Extension Pack 설치 여부 확인 | 미착수 |

### 다음 예정 작업
- Failover 재시연 중 발생한 503/이상 응답 패턴의 원인 재확인 (nginx 중단 순서 재점검) — 단, 이 결과는 구 VirtualBox 환경(192.168.56.10) 기준이므로 VMware 재구축 완료 후에는 재검증 필요
- VMware Workstation에서 haproxy-lb/web-01/web-02 VM 생성 및 Ubuntu 22.04 설치 진행
- VMware 환경에서 unattended install/SSH 포트포워딩(vmnetnat.conf) 방식 확인 후 setup_guide.md에 반영
- ⚠ `review_prep.md`(오픈소스 목록/Q&A/시연 명령어)와 `build_report.md`는 현재 VirtualBox(192.168.56.x) 기준으로 작성되어 있음 — VMware 재구축 완료 시 IP·환경 정보 전면 갱신 필요
- VirtualBox Extension Pack 설치 여부 확인 후 review_prep.md 오픈소스 라이선스 표 최종 확정 (VMware 전환으로 우선순위 하향 가능)
- 정리된 `review_prep.md`로 실제 리뷰 진행 및 피드백 수렴
