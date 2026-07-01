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
- All 3 VMs (haproxy-lb, web-01, web-02) installed via unattended install: username `vboxuser` / `changeme`
- SSH from Windows via NAT port forwarding:
  - haproxy-lb: `ssh -p 2222 vboxuser@127.0.0.1`
  - web-01: `ssh -p 2223 vboxuser@127.0.0.1` (port forwarding to be configured)
  - web-02: `ssh -p 2224 vboxuser@127.0.0.1` (port forwarding to be configured)
- enp0s3 = NAT adapter (internet), enp0s8 = Host-Only adapter (inter-VM traffic); enp0s8 requires manual static IP configuration via netplan
- haproxy-lb enp0s8 static IP confirmed working: `192.168.56.10/24` set via `/etc/netplan/00-installer-config.yaml`
- `netplan apply` produces harmless permission warnings — safe to ignore

## Known Issues

- Ubuntu 22.04 Server installer keyboard freeze: occurs on Profile Setup screen in VirtualBox; workaround is unattended install
- Ubuntu installer mirror check may time out on first attempt; select **Continue** to proceed
- Unattended install does **not** install OpenSSH server — must run `sudo apt install -y openssh-server` manually after first boot
- `sudo` command typed in Windows PowerShell will fail — all `sudo` commands must be run inside the Ubuntu VM via SSH or console

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
