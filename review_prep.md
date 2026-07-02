# 로드밸런싱 구축 리뷰 준비 자료

> 작성일: 2026-07-02
> 근거 자료: `build_report.md`, `load.md`, `daily_log.md`, `CLAUDE.md`
> 대상 문서: `로드밸런싱_구축_가이드.pdf` (build_report.md 기반 최종 제출본)

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

## 2. 리뷰어가 파고들 가능성이 높은 지점

daily_log.md 2026-07-01 기록에 따르면 이번 구축은 **"회사 지시사항: 향후 인프라 교체를 전제로 한 1단계 PoC + 가이드 문서화"**라는 목적이 명확하며, 아래 지적 포인트는 모두 이 프레임으로 대응 가능하다.

| 지적 포인트 | 근거 |
|---|---|
| web-01/web-02 CPU/RAM이 계획(1vCPU/1GB) 대비 변경(2vCPU/2GB)된 채 최종값 미확인 | build_report.md 2절, Full Clone 전환 때문(트러블슈팅 #2) |
| Stats 페이지 인증정보(`admin`/`changeme123`)가 예시값 그대로 | build_report.md 3-3, 7절 개선 제안 1번 |
| HAProxy 자체가 SPOF, keepalived 이중화 미구축 | build_report.md 7절 개선 제안 2번 |
| 전 과정 수동 구성, Ansible/Terraform IaC 미적용 | build_report.md 7절 개선 제안 3번, load.md 4절에 향후 전환 로드맵은 이미 설계됨 |
| curl 응답 원문 등 raw 증적 미보존, daily_log 서술에 의존 | build_report.md 6절 하단 명시 |

---

## 3. Q&A 시나리오

**Q. 오픈소스만으로 구축한 이유는? 상용 로드밸런서는 왜 검토 안 했나?**
A. 상용 로드밸런서 도입 전 검증 단계로 진행한 PoC. 회사 지시사항이 "향후 인프라 교체 전 오픈소스 기반으로 우선 구축 및 문서화"였음 (daily_log 2026-07-01).

**Q. web-01/02 사양이 계획과 다른데 왜 재조정 안 했나?**
A. Ubuntu 신규 설치 중 Profile Setup 화면 키보드 freeze가 반복 발생해 haproxy-lb를 Full Clone하는 방식으로 급선회했고, 클론 후 사양 축소 조정 기록이 원본 메모에 없어 "추가 검증 필요"로 명시했다. 실제 운영 전환 시에는 트래픽 규모에 맞춘 사양 재산정이 선행 과제.

**Q. Stats 페이지 비밀번호가 노출된 예시값인데 보안 문제 아닌가?**
A. 로컬 PoC 환경이라 노출 위험이 없는 격리망에서 진행. 보고서 7절에 운영 전환 시 인증서 적용 및 접근 IP 제한(또는 비밀번호 변경) 필요 사항으로 명시해 두었다.

**Q. LB가 단일 장애점인데 괜찮나?**
A. 이번 범위는 로드밸런싱 기본 동작(분산·헬스체크·장애복구) 검증까지였고, LB 자체 이중화(keepalived/VRRP)는 구축 범위 밖이며 향후 개선 제안에 별도 명시했다.

**Q. 왜 다 수동으로 했나, Ansible/Terraform은?**
A. 1단계는 수동 구성으로 개념 검증, 이후 Ansible(설정 관리)→Terraform(프로비저닝)→CI/CD 순으로 전환하는 로드맵을 load.md 4절에 이미 설계해뒀다. 2단계를 건너뛰고 바로 IaC 도입 시 설정 드리프트 위험이 있다는 점도 문서화함.

---

## 4. 리뷰 자리 시연 명령어

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
