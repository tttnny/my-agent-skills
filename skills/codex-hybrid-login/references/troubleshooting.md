# 常见问题速查

| 错误 | 原因 | 解决 |
|------|------|------|
| `Error loading configuration: No such file or directory (os error 2)` | config.toml 引用了不存在的文件（如已删除的 `model_catalog_json`） | 临时移除 `model_catalog_json` 行完成登录，步骤 4 重建 config.toml 后恢复 |
| SSH 执行 `pkill` 后 `exit code 255` | 杀掉 codex 子进程导致 SSH 会话重置 | **这是预期行为，不是错误**。步骤 1/6 已设计为分两次 SSH：先杀进程（接受 255），再重连删文件 |
| pkill 后 rm 未执行、文件删不掉 | pkill 导致 SSH 断开，同会话的 rm 没机会执行 | **不要在同一次 SSH 中 pkill + rm**。分两次 SSH |
| `/model` 显示 gpt-5.5 / 5.4-mini 等通用模型 | `openai_base_url` 未设置或放错位置 | `openai_base_url` 作为**顶层字段**放在文件最开头 |
| TOML 顶层字段不生效 | model/model_provider 等放在了 `[projects.*]` 等段落之后 | 所有顶层字段必须在第一个 `[section]` 之前 |
| 模型列表不更新 | app-server 缓存 | `pkill -9 -f "[c]odex"` + 删 `models_cache.json` 再重启 |
| `auth.json` 删不掉 | 有 codex 进程在运行 | 先 `pkill -9 -f "[c]odex"` 再删除（分两次 SSH） |
| `codex login` 输出 localhost 地址 | 远程服务器不能打开本地浏览器 | 用 `codex login --device-auth` 替代 |
| `missing field xxx` | model_catalog_json 缺字段 | 从 Mac 原样拷贝，不要手动改 |
| 401 unauthorized | API 密钥错误 | 检查 `experimental_bearer_token` |
| `Not inside a trusted directory` | 目录未信任 | config.toml 添加 `[projects."路径"]` + `trust_level = "trusted"` |
| `failed to parse model_catalog_json` | JSON 格式错误或路径问题 | 路径必须绝对路径，用 `python3 -m json.tool` 验证 |
| login 后台命令无输出 | nohup 输出延迟 | `sleep 5` 后再读输出；如果仍为空，再等 3 秒重试 |
| `which codex` 返回空 | nvm 环境未加载 | 需要先 `export NVM_DIR` + `source nvm.sh` |
| ps/grep 验证时误匹配并行命令 | 步骤间并行导致误判 | **严禁并行执行**，必须等上一步完全结束再执行下一步 |
| login 输出含 `[v0.145.0]` 等乱码字符 | codex 输出包含 ANSI 转义码 | 用 `sed "s/\x1b\[[0-9;]*m//g"` 过滤 |

## 关键注意事项

- **严格顺序执行**：所有步骤串行，不得并行
- **pkill 必须用 `"[c]odex"` 模式**：避免匹配 SSH 命令自身；pkill 后 SSH 断开（exit 255）是正常现象，重新连接即可
- **杀进程和删文件分两次 SSH**：pkill 会断开当前会话，同会话的 `rm` 不会执行
- **`model_catalog_json` 用绝对路径**：必须 `/home/...` 开头，不能用 `~`
- **顶层字段在文件最开头**：`model`、`model_provider`、`openai_base_url` 等必须在第一个 `[section]` 之前
- **`openai_base_url` 是顶层字段**：不在 `[model_providers.*]` 内，Mac 无此字段时从 provider base_url 取值
- **改配置后必须删 `models_cache.json` + 重启**：否则模型列表不更新
- API 密钥不要输出到日志或截图
