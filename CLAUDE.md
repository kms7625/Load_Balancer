# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an infrastructure documentation repository for building a VirtualBox-based open-source load balancing system using HAProxy 2.8 + Ubuntu 22.04. There is no application code — the output is Markdown guides intended for internal company use.

## Files

- `load.md` — Technical reference: load balancer comparison, VM sizing, HAProxy config examples, IaC expansion tips
- `setup_guide.md` — Step-by-step hands-on guide: from VirtualBox install to verified load balancing; updated incrementally as each step is completed
- `daily_log.md` — Daily work log tracking progress, issues encountered, and next steps

## Environment Facts (confirmed during actual build)

- Host OS: Windows 11, VirtualBox 7.2.10
- VirtualBox 7.x does **not** have a GUI Network Manager — use `VBoxManage.exe` via PowerShell instead
- VirtualBox 7.x unattended install runs automatically unless "Proceed with Unattended Installation" is **unchecked** at VM creation; unattended installs create user `vboxuser` / `changeme`
- haproxy-lb: unattended install (vboxuser/changeme); web-01, web-02: cloned from haproxy-lb then hostname/IP changed
- SSH from Windows via NAT port forwarding (all confirmed working):
  - haproxy-lb: `ssh -p 2222 vboxuser@127.0.0.1`
  - web-01: `ssh -p 2223 vboxuser@127.0.0.1`
  - web-02: `ssh -p 2224 vboxuser@127.0.0.1`
- enp0s3 = NAT adapter (internet), enp0s8 = Host-Only adapter (inter-VM traffic); enp0s8 requires manual static IP configuration via netplan
- `netplan apply` produces harmless permission warnings — safe to ignore
- HAProxy health check must use `HTTP/1.0` not `HTTP/1.1` — nginx rejects HTTP/1.1 requests without a Host header
- After cloning, `/etc/nginx/sites-enabled/default` must be deleted to avoid `default_server` conflict with `health.conf`

## Build Status

**완료** — 전체 구축 및 동작 검증 완료 (2026-07-01)

## Known Issues

- Ubuntu 22.04 Server installer keyboard freeze: occurs on Profile Setup screen in VirtualBox; workaround is to clone haproxy-lb instead
- Ubuntu installer mirror check may time out on first attempt; select **Continue** to proceed
- Unattended install does **not** install OpenSSH server — must run `sudo apt install -y openssh-server` manually after first boot
- `sudo` command typed in Windows PowerShell will fail — all `sudo` commands must be run inside the Ubuntu VM via SSH or console
- `systemctl status` opens a pager — use `--no-pager` flag to avoid getting stuck

## IP Scheme

| Host | IP |
|---|---|
| Windows host | 192.168.56.1 |
| haproxy-lb | 192.168.56.10 |
| web-01 | 192.168.56.11 |
| web-02 | 192.168.56.12 |

## Conventions

- Shell blocks = bash (run inside Ubuntu VMs via SSH); PowerShell blocks = Windows host commands
- `setup_guide.md` reflects what was actually done, not what was planned — update it only after a step is confirmed working
- `daily_log.md` is updated at end of each session with completed items and next-day tasks
