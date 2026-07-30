# Codex CLI — 仅更新 API 端点

> 只修改 Linux config.toml 中的 `openai_base_url`，不涉及 OAuth 登录或模型目录更新。

---

## 前提

此模式适用于以下场景：
- Linux 上的 Codex 已经配置好混合登录，`auth.json` 有效
- 仅 API 端点地址发生变化
- 模型目录（`cockpit-provider-model-catalog.json`）不需要更新

## 执行纪律

**所有步骤必须严格顺序执行，不得并行。**

---

## 步骤

### 0. 读取 Mac 配置

```bash
cat ~/.codex/config.toml
```

提取新的 `openai_base_url`（如果没有顶层字段，从 `[model_providers.codex_local_access].base_url` 取值）。

### 1. 修改 Linux config.toml 中的 openai_base_url

先读 Linux 当前的 config.toml：
```bash
ssh <SSH_USER>@<SSH_HOST> 'cat ~/.codex/config.toml'
```

定位 `openai_base_url` 行，将其值替换为 Mac 上的值。只需修改这一行，其他所有内容保持不变。

用 `sed` 直接替换：
```bash
ssh <SSH_USER>@<SSH_HOST> "sed -i 's|^openai_base_url = .*|openai_base_url = \"<新的URL>\"|' ~/.codex/config.toml"
```

验证修改：
```bash
ssh <SSH_USER>@<SSH_HOST> 'grep "^openai_base_url" ~/.codex/config.toml'
```

### 2. 杀掉进程 + 清缓存

**杀进程和删文件必须分两次 SSH 执行。**

```bash
# 1) 杀进程（接受 exit 255）
ssh <SSH_USER>@<SSH_HOST> 'pkill -9 -f "[c]odex" 2>/dev/null; echo "killed"'

# 2) 删除缓存文件
ssh <SSH_USER>@<SSH_HOST> 'rm -f ~/.codex/models_cache.json && echo "cache cleaned"'

# 3) 验证
ssh <SSH_USER>@<SSH_HOST> 'ps aux | grep "[c]odex" || echo "All killed"'
ssh <SSH_USER>@<SSH_HOST> 'ls ~/.codex/models_cache.json 2>/dev/null || echo "models_cache removed"'
```

### 3. 启动并验证

```bash
ssh <SSH_USER>@<SSH_HOST> '
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
nohup codex > /tmp/codex_startup.log 2>&1 &
sleep 3
echo "codex PID: $(pgrep -f "[c]odex" | head -1)"
'
```

健康检查：
```bash
ssh <SSH_USER>@<SSH_HOST> 'pgrep -f "[c]odex" && echo "Codex is running" || echo "Codex NOT running"'
ssh <SSH_USER>@<SSH_HOST> 'grep -i "fatal\|panic\|error" /tmp/codex_startup.log 2>/dev/null || echo "No startup errors"'
```
