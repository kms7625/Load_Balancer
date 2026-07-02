# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an infrastructure documentation repository for building a VMware Workstation-based open-source load balancing system using HAProxy 2.8 + Ubuntu 22.04. There is no application code — the output is Markdown guides intended for internal company use.

> **참고**: 이 프로젝트는 원래 VirtualBox 7.2.10 기준으로 구축되어 2026-07-01에 완료되었으나, 2026-07-02부터 VMware Workstation으로 다시 구축 중. VirtualBox 관련 세부 정보는 필요 시 git 이력(`git log`)에서 확인 가능.

## Files

- `load.md` — Technical reference: load balancer comparison, VM sizing, HAProxy config examples, IaC expansion tips
- `setup_guide.md` — Step-by-step hands-on guide: from VMware Workstation VM setup to verified load balancing; updated incrementally as each step is completed
- `daily_log.md` — Daily work log tracking progress, issues encountered, and next steps
- `build_report.md` — Internal-facing build report (architecture diagram, VM/IP tables, troubleshooting table, verification results); generated from `로드밸런싱 구축.txt` + `daily_log.md` + `setup_guide.md` + `CLAUDE.md`
- `로드밸런싱 구축.txt` — Original raw work notes; source of truth that `build_report.md` is checked against
- `로드밸런싱_구축_가이드.pdf` — PDF export of the build guide for sharing outside the repo

## Environment Facts (confirmed during actual build)

- Host OS: Windows 11, VMware Workstation 25.0.1 (build-25219725), installed at `C:\Program Files (x86)\VMware\VMware Workstation`
- VMware Workstation CLI tools live alongside the GUI: `vmrun.exe` (VM power control), `vmnetcfg.exe` (Virtual Network Editor), `vnetlib64.exe` (network config queries) — use these via PowerShell for anything not worth doing in the GUI
- Virtual network adapters on this host (confirmed via `Get-NetAdapter` / `Get-NetIPAddress`):
  - `VMnet8` = NAT adapter, host-side IP `192.168.198.1/24` (internet access for VMs)
  - `VMnet1` = Host-only adapter, host-side IP `192.168.164.1/24` (inter-VM traffic)
- Ubuntu 22.04.5 Server ISO already downloaded: `C:\Users\ms.kang\Downloads\ubuntu-22.04.5-live-server-amd64.iso`
- 미확인·추가 검증 필요: VMware 게스트에서 unattended install 방식, 계정 생성 방식, SSH 포트포워딩 설정 방법(NAT는 VirtualBox처럼 개별 포트포워딩 룰이 아니라 `vmnetnat.conf` 편집 방식), 클론 후 정리 필요 항목, guest NIC 명칭(enp0s3/enp0s8 유지 여부)

## Build Status

**진행 중** — VMware Workstation으로 재구축 시작 (2026-07-02). 이전 VirtualBox 기반 구축은 2026-07-01에 완료됨(git 이력 참고).

## Known Issues

- 미확인·추가 검증 필요: 아래는 VirtualBox 빌드에서 확인된 이슈로, VMware에서도 동일하게 재현되는지 아직 검증되지 않음
  - Ubuntu 22.04 Server installer keyboard freeze (Profile Setup 화면)
  - Ubuntu installer mirror check 타임아웃 시 **Continue**로 진행
  - Unattended install은 OpenSSH server를 설치하지 않음 — 최초 부팅 후 `sudo apt install -y openssh-server` 수동 실행 필요
- `sudo` 명령은 Windows PowerShell에서 실행 불가 — 반드시 Ubuntu VM 내부(SSH 또는 콘솔)에서 실행
- `systemctl status`는 pager를 열므로 `--no-pager` 플래그 사용
- HAProxy 헬스체크는 `HTTP/1.0` 사용 필요(`HTTP/1.1`은 Host 헤더 없으면 nginx가 거부) — VirtualBox 빌드에서 확인, VMware에서도 동일할 것으로 예상되나 재검증 필요

## IP Scheme

| Host | IP | 비고 |
|---|---|---|
| Windows host (VMnet1) | 192.168.164.1 | Host-only, 인터-VM 트래픽 |
| Windows host (VMnet8) | 192.168.198.1 | NAT, 인터넷 접근용 |
| haproxy-lb | 192.168.164.10 | 미확인·추가 검증 필요 (게스트 static IP 설정 전) |
| web-01 | 192.168.164.11 | 미확인·추가 검증 필요 |
| web-02 | 192.168.164.12 | 미확인·추가 검증 필요 |

## Conventions

- Shell blocks = bash (run inside Ubuntu VMs via SSH); PowerShell blocks = Windows host commands
- `setup_guide.md` reflects what was actually done, not what was planned — update it only after a step is confirmed working
- `daily_log.md` is updated at end of each session with completed items and next-day tasks
- `build_report.md` must only state what's backed by the raw notes/logs; anything not confirmed in source material is labeled "미확인·추가 검증 필요" rather than inferred or guessed
