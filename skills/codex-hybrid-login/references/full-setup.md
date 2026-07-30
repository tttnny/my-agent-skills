# Codex CLI 混合登录 — 完整配置流程

> 通过 SSH 远程配置 Linux 服务器的 Codex CLI 混合登录（ChatGPT OAuth + 自定义 API 端点）。

---

## 前提变量

这些变量由 SKILL.md 在开始时向用户收集：

| 变量 | 说明 |
|------|------|
| `SSH_USER` | SSH 用户名 |
| `SSH_HOST` | 服务器 IP 或域名 |
| `SSH_PASSWORD` | SSH 密码（仅无免密时使用） |

`model`、`openai_base_url`、`model_catalog_json`、API token 等均从 Mac 的 `~/.codex/config.toml` 自动提取。

免密 SSH？是 → 直接 `ssh`；否 → 安装 `sshpass`，所有命令加 `sshpass -p '<SSH_PASSWORD>'`

---

## 执行纪律

- **所有步骤必须严格顺序执行，不得并行**。并行会导致步骤间的进程残留相互干扰（例如步骤 1 杀进程时，步骤 2 的 `codex --version` 又启动了新进程）。
- 每个步骤内的多条 `ssh` 命令可以合并到同一个 SSH 会话中执行，但跨步骤的 `ssh` 必须串行。

---

## 步骤

### 0. 读取 Mac 配置（第一步就做）

后续步骤需要 Mac config.toml 中的 model、openai_base_url、provider 配置等信息：

```bash
cat ~/.codex/config.toml
```

记录以下关键字段，后续步骤会用到：
- `model` — 模型名称
- `openai_base_url` — 顶层字段（如不存在，从 `[model_providers.codex_local_access].base_url` 取值）
- `[model_providers.custom]` 整段
- `[model_providers.codex_local_access]` 整段
- `model_catalog_json` 本地文件路径

### 1. 清除旧认证（仅删登录相关文件）

**关键规则**：`pkill` 必须用 `"[c]odex"` 模式（避免匹配 SSH 自身）；**杀进程和删文件分两次 SSH**（pkill 会断开会话导致同会话的 `rm` 不执行）；**不删 `cockpit-provider-model-catalog.json`**（config.toml 引用它，删除会导致后续 login 报错）。

```bash
# 1) 只杀进程（接受 exit 255，无需重试）
ssh <SSH_USER>@<SSH_HOST> 'pkill -9 -f "[c]odex" 2>/dev/null; echo "killed"'

# 2) 重新连接，确认进程已死后删除文件（此时无进程锁定，rm 一定成功）
ssh <SSH_USER>@<SSH_HOST> 'rm -f ~/.codex/auth.json ~/.codex/models_cache.json && echo "files cleaned"'

# 3) 验证
ssh <SSH_USER>@<SSH_HOST> 'ps aux | grep "[c]odex" || echo "All killed"'
ssh <SSH_USER>@<SSH_HOST> 'ls ~/.codex/auth.json 2>/dev/null || echo "auth.json removed"'
ssh <SSH_USER>@<SSH_HOST> 'ls ~/.codex/models_cache.json 2>/dev/null || echo "models_cache removed"'
```

### 2. 检查远程环境

**在步骤 1 完全结束后再执行此步骤**，不要并行。

```bash
# 检查 codex 路径和版本
ssh <SSH_USER>@<SSH_HOST> '
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
which codex && codex --version
'

# 检查 config.toml 和 model_catalog_json 是否存在
ssh <SSH_USER>@<SSH_HOST> 'ls -la ~/.codex/config.toml ~/.codex/cockpit-provider-model-catalog.json 2>/dev/null || echo "MISSING"'
```

如果 `codex --version` 报错 `Error loading configuration: No such file or directory`，说明 config.toml 引用了不存在的文件。运行以下诊断：
```bash
ssh <SSH_USER>@<SSH_HOST> '
grep -oP "(source|model_catalog_json)\s*=\s*\"\K[^\"]+" ~/.codex/config.toml 2>/dev/null | while read p; do
  [ -e "$p" ] && echo "OK: $p" || echo "MISSING: $p"
done
'
```

### 3. OAuth 登录（设备码模式）

远程服务器无法打开本地浏览器，使用设备码认证。

如果 `model_catalog_json` 指向的文件缺失（步骤 2 已验证），需临时移除该行：
```bash
ssh <SSH_USER>@<SSH_HOST> 'cp ~/.codex/config.toml ~/.codex/config.toml.login.bak && grep -v "model_catalog_json" ~/.codex/config.toml > /tmp/config_nocatalog.toml && mv /tmp/config_nocatalog.toml ~/.codex/config.toml'
```
> 登录成功后，步骤 4 会重建 config.toml 恢复该行。

```bash
# 后台启动 login（避免 agent 卡住等待授权）
ssh <SSH_USER>@<SSH_HOST> '
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
nohup codex login --device-auth > /tmp/codex_login_output.txt 2>&1 &
echo "login started, PID: $!"
'
sleep 5

# 读取验证链接和验证码（如果为空，再等 3 秒重试）
# 用 sed 过滤 ANSI 转义码，输出更干净
ssh <SSH_USER>@<SSH_HOST> 'cat /tmp/codex_login_output.txt | sed "s/\x1b\[[0-9;]*m//g"'
```

把输出的链接和验证码（格式如 `Z75M-ZVYUH`）清晰展示给用户。输出中 `[v0.145.0]` 和 `(expires in 15 minutes)` 等 ANSI 控制码会被自动过滤掉。登录后检查 `auth.json` 是否生成：
```bash
ssh <SSH_USER>@<SSH_HOST> 'ls -la ~/.codex/auth.json 2>/dev/null && echo "Login OK" || echo "Not yet"'
```
**等用户确认登录完成后再继续。**

### 4. 检查并（按需）重建 config.toml

**先对比 Mac 和 Linux 的 config.toml，判断是否需要重建**。很多时候 Linux 配置已经是正确的，跳过重建可避免误删 Linux 独有的 marketplace/projects 配置。

```bash
# 读取 Linux 当前 config.toml
ssh <SSH_USER>@<SSH_HOST> 'cat ~/.codex/config.toml'
```

**快速判断 — 以下条件全部满足则跳过重建**：
- ✅ 顶层字段齐全：`model`、`model_provider`、`openai_base_url`、`model_catalog_json`、`model_reasoning_effort` 都在文件最开头（第一个 `[section]` 之前）
- ✅ `model` 值与 Mac 一致
- ✅ `openai_base_url` 值与 Mac 的 provider base_url 一致
- ✅ `model_catalog_json` 是绝对路径（如 `/home/<SSH_USER>/.codex/...`）
- ✅ `[model_providers.custom]` 和 `[model_providers.codex_local_access]` 段存在且内容合理

**如需重建**，遵循以下规则：

**三条 TOML 铁律**：
1. **顶层字段必须在文件最开头** — `model`、`model_provider`、`model_catalog_json`、`model_reasoning_effort`、`openai_base_url` 必须放在第一个 `[section]` 之前，否则 TOML 解析为某个 section 的子字段导致不生效
2. **`openai_base_url` 是顶层字段**，不是 `[model_providers.codex_local_access]` 的字段。它控制桌面端从哪里拉取模型列表，不设则走默认 OpenAI（返回 gpt-5.5 等通用模型而非自定义 API 的模型）。如 Mac 没有此顶层字段，从 `[model_providers.codex_local_access].base_url` 取值
3. **`model_catalog_json` 必须是绝对路径**（如 `/home/<SSH_USER>/.codex/cockpit-provider-model-catalog.json`），不能用 `~` 开头

**必须保留/确保的顶层字段**（放在文件最开头）：
```toml
model_reasoning_effort = "medium"
model_provider = "codex_local_access"
model = "<从 Mac config.toml 的 model 字段读取>"
model_catalog_json = "/home/<SSH_USER>/.codex/cockpit-provider-model-catalog.json"
openai_base_url = "<从 Mac config.toml 的 openai_base_url 字段读取，无则取 provider base_url>"
```

**从 Mac config.toml 复制到 Linux 的段落**：
- `[model_providers.custom]`
- `[model_providers.codex_local_access]`（注意：`experimental_bearer_token` 使用 Linux 上已有的有效 token）

**保留 Linux 独有的段落**（不要覆盖）：
- `[marketplaces.*]` — Linux 上的 marketplace 路径
- `[projects.*]` — Linux 上的项目信任配置（`/home/` 路径的）
- `[plugins.*]` — 已启用的插件
- `[hooks.*]` — hooks 配置

**确保删除的段落**：
- `[mcp_servers.*]` — Mac 的 MCP 服务器路径不适用于 Linux
- `[shell_environment_policy.*]` — Mac 的环境变量策略
- `[desktop]`、`[windows]`、`[notify]` — Mac 专属配置
- `[features]` — Mac 专属特性开关
- 所有 Mac 路径的项目信任配置（`[projects."/Users/..."]`）

写入 Linux：
```bash
# 先备份
ssh <SSH_USER>@<SSH_HOST> 'cp ~/.codex/config.toml ~/.codex/config.toml.bak'
# 写入新配置（用 scp 或 heredoc 方式）
scp /tmp/new_config.toml <SSH_USER>@<SSH_HOST>:~/.codex/config.toml
```

### 5. 拷贝模型目录

从 Mac 原样拷贝 `cockpit-provider-model-catalog.json`（不要修改内容，JSON 校验极严）。

**先对比文件大小，相同则跳过 scp**（避免无意义传输）：
```bash
# Mac 端大小
wc -c ~/.codex/cockpit-provider-model-catalog.json
# Linux 端大小
ssh <SSH_USER>@<SSH_HOST> 'wc -c ~/.codex/cockpit-provider-model-catalog.json 2>/dev/null || echo "MISSING"'
```

如果大小一致 → **跳过 scp**，直接进入步骤 6。如果大小不同或文件缺失 → 执行拷贝：
```bash
scp ~/.codex/cockpit-provider-model-catalog.json <SSH_USER>@<SSH_HOST>:~/.codex/cockpit-provider-model-catalog.json

# 验证大小一致
ssh <SSH_USER>@<SSH_HOST> 'ls -la ~/.codex/cockpit-provider-model-catalog.json && wc -c ~/.codex/cockpit-provider-model-catalog.json'
```

### 6. 杀掉所有进程 + 清缓存

改配置后必须重启，否则 `/model` 列表不会更新（app-server 缓存旧配置）。

**与步骤 1 相同：杀进程和删文件必须分两次 SSH 执行，避免 exit 255 导致 rm 未执行。**

```bash
# 1) 只杀进程（接受 exit 255）
ssh <SSH_USER>@<SSH_HOST> 'pkill -9 -f "[c]odex" 2>/dev/null; echo "killed"'

# 2) 重新连接，删除缓存文件
ssh <SSH_USER>@<SSH_HOST> 'rm -f ~/.codex/models_cache.json && echo "cache cleaned"'

# 3) 验证
ssh <SSH_USER>@<SSH_HOST> 'ps aux | grep "[c]odex" || echo "All killed"'
ssh <SSH_USER>@<SSH_HOST> 'ls ~/.codex/models_cache.json 2>/dev/null || echo "models_cache removed"'
```

### 7. 启动并验证

在 Linux 上启动 Codex：
```bash
ssh <SSH_USER>@<SSH_HOST> '
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
nohup codex > /tmp/codex_startup.log 2>&1 &
sleep 3
echo "codex PID: $(pgrep -f "[c]odex" | head -1)"
'
```

> `stdin is not a terminal` 是 nohup 启动的正常输出，不影响功能。

启动后快速健康检查：
```bash
# 检查进程存活
ssh <SSH_USER>@<SSH_HOST> 'pgrep -f "[c]odex" && echo "Codex is running" || echo "Codex NOT running"'
# 检查启动日志是否有 FATAL/panic
ssh <SSH_USER>@<SSH_HOST> 'grep -i "fatal\|panic\|error" /tmp/codex_startup.log 2>/dev/null || echo "No startup errors"'
```

用户需 SSH 交互登录后，进入 Codex 用 `/model` 验证模型列表。如果不对：
1. 检查 `model_catalog_json` 是否为**绝对路径**（不能用 `~`）
2. 顶层字段是否在文件最开头（`[section]` 之前）
3. 删除 `models_cache.json` 清除旧缓存
4. `pkill -9 -f "[c]odex"` 杀掉所有进程再重启
5. 检查 `openai_base_url` 是否正确设置
