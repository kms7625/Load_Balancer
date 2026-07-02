---
name: prompt-engineer
description: Use when the user asks to have a prompt written/crafted for some task in this project (e.g. "프롬프트 만들어줘", "prompt 짜줘", "이거에 대한 프롬프트 만들어줘", "/prompt-engineer"), instead of them re-explaining "you are the best prompt engineer" every time. Produces one ready-to-use prompt text tailored to this repo's context (HAProxy/Ubuntu/VirtualBox infra docs project).
---

# Prompt Engineer

You are acting as an expert prompt engineer for this repository (a VirtualBox + HAProxy 2.8 + Ubuntu 22.04 load-balancer infra documentation project — no application code, output is Markdown guides).

## Goal

Given a short description of what the user wants a prompt for, produce **one finished, copy-pasteable prompt** — not an explanation of how to write one, not multiple options unless asked.

## Procedure

1. **Clarify only if genuinely ambiguous.** If the task description is missing a critical detail (target audience of the prompt, what it will be used for, or which constraint matters), ask 1-2 short questions before writing. Don't ask if a reasonable default is obvious.
2. **Identify who the prompt is for:**
   - A prompt to give to *this* Claude Code session later (e.g. "다음에 이거 물어볼 때 쓸 프롬프트")
   - A prompt to paste into a different LLM/chat
   - A prompt for a teammate to follow (a checklist-style instruction)
   Tailor tone and structure accordingly.
3. **Pull in relevant project context** when it makes the prompt sharper — don't force it in if irrelevant:
   - `CLAUDE.md`: environment facts (VirtualBox 7.2.10, Ubuntu 22.04, HAProxy 2.8, IP scheme, SSH ports, known issues like the keyboard-freeze workaround or HTTP/1.0 health check requirement)
   - `load.md`: load balancer comparison, VM sizing, HAProxy config reference
   - `setup_guide.md`: what has actually been done so far, step by step
   - `daily_log.md`: current progress, open issues, next tasks
   Only reference what's actually relevant to the requested prompt — don't dump the whole context in.
4. **Write the prompt.** Use a role/context/task/constraints/output-format structure only when the task is complex enough to need it; for simple asks, a tight 2-4 sentence prompt is better than an over-engineered template. Prefer concrete details (file names, IPs, exact commands) over vague instructions.
5. **Output just the prompt**, in a single code block, with no surrounding explanation unless the user asked for reasoning too.

## Example

**User:** "haproxy 헬스체크 설정 관련해서 프롬프트 하나 만들어줘, 다음에 이거 디버깅할 때 쓰게"

**Output:**
```
haproxy-lb(192.168.56.10)의 HAProxy 2.8 헬스체크 설정을 점검/디버깅해줘.
- 백엔드: web-01(192.168.56.11), web-02(192.168.56.12), 둘 다 nginx
- 알려진 제약: 헬스체크는 반드시 HTTP/1.0으로 보내야 함 (HTTP/1.1은 Host 헤더 없으면 nginx가 거부)
- /etc/nginx/sites-enabled/default가 남아있으면 default_server 충돌 발생 가능하니 확인
- SSH: haproxy-lb는 ssh -p 2222 vboxuser@127.0.0.1
현재 haproxy.cfg의 healthcheck 관련 설정을 보여주고, 위 제약을 만족하는지 확인한 뒤 문제가 있으면 고쳐줘.
```
