---
name: codex-hybrid-login
description: This skill configures Codex CLI hybrid login (ChatGPT OAuth + custom API endpoint) on a remote Linux server via SSH. Use when the user wants to set up Codex on a remote Linux machine, fix login/auth issues, update the API endpoint, or reconfigure Codex CLI. Triggered by phrases like 配置Codex, Codex混合登录, 更新Codex API, 修复Codex登录, setup Codex on server.
---

# Codex Hybrid Login

## Overview

Configure Codex CLI hybrid login on a remote Linux server via SSH. This skill supports two modes:
- **Full Setup**: Complete reconfiguration (OAuth login, rebuild config, copy models, restart)
- **API Update**: Only update the `openai_base_url` endpoint (no OAuth, no model directory changes)

## Workflow Decision Tree

Upon invocation, **the first and only action** is to ask the user which mode they need:

> 请选择操作模式：
>
> **A. 完整重新配置** — 重新做 OAuth 登录、重建 config.toml、拷贝模型目录（适用于全新配置或登录过期）
> **B. 仅更新 API** — 只修改 `openai_base_url`（适用于仅 API 端点变化，auth.json 仍然有效）

Wait for the user's choice before proceeding. Do not execute any SSH commands until the mode is confirmed.

### Collect SSH Credentials

After the mode is selected, collect the SSH connection details from the user if not already provided:

- `SSH_USER` — SSH username
- `SSH_HOST` — Server IP or hostname
- `SSH_PASSWORD` — SSH password (only needed if key-based auth is not configured)

If the user has key-based SSH (no password needed), use plain `ssh <user>@<host>`. If a password is required, prepend `sshpass -p '<password>'` to all SSH commands after installing sshpass.

### Execute by Mode

- **Mode A (Full Setup)**: Load `references/full-setup.md` and follow all steps in strict sequential order.
- **Mode B (API Update)**: Load `references/api-update.md` and follow all steps in strict sequential order.

### Troubleshooting

If any step fails or produces unexpected output, load `references/troubleshooting.md` to find the matching error and apply the documented fix.

## Critical Rules (Apply to Both Modes)

1. **All steps MUST execute strictly in sequence, never in parallel.** Parallel execution causes process state interference.
2. **Use `"[c]odex"` pattern for pkill** to avoid matching the SSH command itself.
3. **Separate pkill and rm into two SSH sessions.** pkill disconnects the current session; any `rm` in the same session will not execute.
4. **Top-level TOML fields must appear before any `[section]`.** This includes `model`, `model_provider`, `openai_base_url`, `model_catalog_json`, `model_reasoning_effort`.
5. **`model_catalog_json` must use absolute paths** (e.g., `/home/user/.codex/...`), never `~`.
6. **After any config change, delete `models_cache.json` and restart** — otherwise the model list will not refresh.
7. **Never output API keys or bearer tokens** to logs, terminal output, or screenshots.
