# 로드밸런싱 구축

> 작성일: 2026-07-02
> 대상 문서: `로드밸런싱_구축_가이드.pdf`

---

## 1. 오픈소스 목록 및 선정 이유

| 구성요소 | 버전 | 라이선스 | 선정 이유 |
|---|---|---|---|
| **HAProxy** | 2.8 LTS (실제 설치 2.8.25, PPA `vbernat/haproxy-2.8`) | GPLv2 (OpenSSL 링킹 예외 포함) | Nginx OSS·Traefik 대비 능동 헬스체크가 OSS 버전에 기본 포함, Runtime API로 무중단 서버 제어 가능, 이벤트 루프 기반이라 VM 리소스 제약 환경에서 메모리 효율 우수 (load.md 비교표 근거) |
| **Nginx** | Ubuntu 22.04 기본 저장소 버전 | BSD-2-Clause | 백엔드 웹서버 역할만 분리 담당. "웹서버+리버스프록시를 한 프로세스에서" 처리해야 하는 상황이 아니므로 로드밸런서 역할은 HAProxy에 위임 |
| **Ubuntu Server** | 22.04.5 LTS | 다수 OSS 라이선스 조합(GPL 등), Canonical 배포 | 장기 지원(LTS), HAProxy 2.8 패키지 공식 지원 |
| **VirtualBox** | 7.2.10 | 베이스는 GPLv2, **단 Extension Pack은 PUEL(상용 라이선스)** — 이번 구축에서 Extension Pack 설치 여부는 미확인·추가 확인 필요 | 개발/검증 단계 로컬 테스트용으로 무료+GUI 편의성 선택 (실 운영 전환 시 KVM+libvirt 권장이라고 load.md에 명시) |
| **socat** | Ubuntu 기본 저장소 버전 | GPLv2 | HAProxy Runtime API 소켓(`/run/haproxy/admin.sock`) 조회용 보조 도구 |

**미확인·추가 확인 필요**: VirtualBox Extension Pack(USB 2.0/3.0, RDP 등) 설치 여부. build_report.md·daily_log.md 어디에도 Extension Pack 설치 기록이 없음. 설치하지 않았다면 "베이스 VirtualBox만 사용, GPLv2"로 명확히 답변 가능하나, 리뷰 전 실제 VM 설정에서 재확인 필요.

---

## 2. 



| 포인트 | 근거 |
|---|---|
| web-01/web-02 CPU/RAM이 계획(1vCPU/1GB) 대비 변경(2vCPU/2GB)된 채 최종값 미확인 | build_report.md 2절, Full Clone 전환 때문(트러블슈팅 #2) |
| Stats 페이지 인증정보(`admin`/`changeme123`)가 예시값 그대로 | build_report.md 3-3, 7절 개선 제안 1번 |
| HAProxy 자체가 SPOF, keepalived 이중화 미구축 | build_report.md 7절 개선 제안 2번 |
| 전 과정 수동 구성, Ansible/Terraform IaC 미적용 | build_report.md 7절 개선 제안 3번, load.md 4절에 향후 전환 로드맵은 이미 설계됨 |
| curl 응답 원문 등 raw 증적 미보존, daily_log 서술에 의존 | build_report.md 6절 하단 명시 |

---


## 4. 시연 명령어

haproxy-lb SSH 접속: `ssh -p 2222 vboxuser@127.0.0.1`

```bash
# 1. 부하 분산 확인 (web-01/web-02 교대 응답)
for i in $(seq 1 6); do curl -s http://192.168.56.10/; done

# 2. 장애 복구 확인
#   (a) web-01에서 실행: ssh -p 2223 vboxuser@127.0.0.1
sudo systemctl stop nginx
#   (b) haproxy-lb에서 6초(fall 3 x inter 2s) 후 실행 — web-02만 응답해야 함
for i in $(seq 1 4); do curl -s http://192.168.56.10/; done
#   (c) web-01 복구
sudo systemctl start nginx
sleep 5
for i in $(seq 1 6); do curl -s http://192.168.56.10/; done   # 다시 교대 확인

# 3. 헬스체크 상태 직접 조회
echo "show servers state" | sudo socat stdio /run/haproxy/admin.sock
```

```
Stats 페이지 (Windows 브라우저)
http://192.168.56.10:8404/stats
ID: admin / PW: changeme123
```

  접속하면 웹 브라우저에서 실시간으로 확인할 수 있는 것들:                                                 
  - Status: web-01, web-02 각 백엔드 서버가 UP인지 DOWN인지                                                
  - Sessions: 현재 연결된 세션 수                                                                          
  - Bytes in/out: 서버별 트래픽 통계                                                                       
  - Check: 헬스체크 결과 및 응답 시간                                                                      
  - 서버를 수동으로 drain/maintenance 상태로 전환하는 관리 기능도 포함 
