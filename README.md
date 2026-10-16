# Load_Balancer — HAProxy 오픈소스 로드밸런싱 구축 기록

상용 로드밸런서 도입 전 검증 단계로, 로컬 가상화 환경에 **HAProxy 2.8 LTS + Nginx** 기반 로드밸런싱을 직접 구축하고 절차·시행착오를 문서로 남긴 저장소입니다. 애플리케이션 코드는 없고 Markdown 가이드가 결과물입니다.

## 구성

```
              [Windows 호스트]
                    │ Host-Only 네트워크
          ┌─────────┴─────────┐
          │  haproxy-lb (:80)  │  ← 로드밸런서 · Stats :8404
          └─────────┬─────────┘
            라운드로빈 분산
          ┌─────────┴─────────┐
          ▼                   ▼
     web-01 (Nginx)      web-02 (Nginx)
```

Ubuntu 22.04 Server VM 3대(로드밸런서 1 · 백엔드 웹서버 2), 각 VM은 NAT(인터넷) + Host-Only(VM 간 통신) 어댑터 사용.

## 검증 결과 (VirtualBox 구축, 2026-07-01)

| 항목 | 결과 |
|---|---|
| 부하 분산(Round Robin) | web-01·web-02 교대 응답 확인 |
| 장애 복구(Failover) | web-01 Nginx 중단 → 약 6초(fall 3 × inter 2s) 후 web-02로만 응답, 복구 후 다시 분산 |
| 헬스체크 | 두 서버 `srv_op_state = 2`(UP) |
| Stats 페이지 | 접속·로그인 확인 |

응답 원문 로그는 보존하지 않아 작업 기록(`daily_log.md`) 기준으로 적었습니다.

**주요 트러블슈팅**: 헬스체크를 `HTTP/1.1`로 보내면 Host 헤더가 없어 Nginx가 거부 → `option httpchk GET /health HTTP/1.0`으로 변경 후 UP 전환. 설치 중 키보드 멈춤으로 웹서버 VM은 로드밸런서 VM을 Full Clone해 생성.

## 문서

| 파일 | 내용 |
|---|---|
| `load.md` | 로드밸런서 비교(HAProxy·Nginx·Traefik)와 선정 이유, VM 사양, HAProxy 설정 예시 |
| `setup_guide.md` | VM 생성부터 부하 분산 검증까지 단계별 실습 가이드 |
| `build_report.md` | 구축 결과 보고서 — 구성도·VM/IP 표·트러블슈팅·검증 결과 |
| `daily_log.md` | 일자별 진행·이슈 기록 |

## 진행 상태

VirtualBox 기반 구축은 2026-07-01 완료, 2026-07-02부터 VMware Workstation으로 재구축 진행 중입니다. 로드밸런서 이중화(keepalived 등)는 범위 밖입니다.
