# CPA-NORT

中文 | [English](README_EN.md)

CPA-NORT 是基于 [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) 的二开版本，重点优化“导入的 OAuth JSON 只有可用 `access_token`、没有可用 `refresh_token`”这一类使用场景。

本项目保留 CLIProxyAPI 原有的兼容接口，同时改进无 refresh token 凭据的运行表现：

- 提供 OpenAI/Codex 兼容 API 接口
- 继承 Gemini、Claude、Codex、Antigravity、Kimi、OpenAI-compatible 等 provider 支持
- 支持从 JSON 文件加载多账号凭据
- 支持轮询与会话粘性路由
- 支持流式、非流式，以及受支持场景下的 WebSocket 传输
- 支持无 `refresh_token` OAuth JSON，不再进入后台刷新循环
- 提供全局开关，可禁用 OAuth/file auth 的后台自动刷新检查

## 为什么有这个版本

有些导入的 auth JSON 里，`access_token` 仍然可用，但没有 `refresh_token`，或者 `refresh_token` 已经无法再次使用。传统后台自动刷新逻辑可能会反复把这些凭据加入刷新调度，造成日志噪音，甚至出现间歇性卡顿。

CPA-NORT 会把缺少 `refresh_token` 的 auth 文件按 access-token-only 凭据处理：

- 实际请求继续使用现有 `access_token`。
- 如果 access token 真的失效，上游返回的 `401` 仍会在真实请求中被正常检测。
- 对依赖 refresh token 的 provider，缺少 `refresh_token` 时直接跳过后台刷新调度。
- 不会为了“测活”而调用刷新接口。

## 快速开始

```bash
go build -o cli-proxy-api ./cmd/server
./cli-proxy-api --config config.yaml
```

Windows PowerShell：

```powershell
go build -o cli-proxy-api.exe ./cmd/server
.\cli-proxy-api.exe --config config.yaml
```

## 配置

创建或编辑 `config.yaml`：

```yaml
host: "127.0.0.1"
port: 8317
auth-dir: "./auths"

# 可选保险开关。为 true 时禁用所有 OAuth/file auth 后台自动刷新检查。
disable-auth-auto-refresh: false

request-retry: 3
max-retry-credentials: 0
max-retry-interval: 30
```

把 OAuth JSON 放到 `auth-dir` 下，例如：

```text
auths/
  token_user@example.com.json
```

Codex/OpenAI access-token-only 文件通常只需要类似字段：

```json
{
  "type": "codex",
  "access_token": "eyJ...",
  "email": "user@example.com",
  "expired": "2026-05-11T12:00:00Z"
}
```

如果缺少 `refresh_token`，CPA-NORT 会对无法无刷新凭据续期的 provider 跳过后台刷新调度。

## OAuth Client 凭据

CPA-NORT 不在仓库中硬编码 Google OAuth client 凭据。如果需要使用 Gemini CLI 或 Antigravity 的交互式 OAuth 登录流程，请通过环境变量提供：

```bash
export CPA_NORT_GEMINI_OAUTH_CLIENT_ID="..."
export CPA_NORT_GEMINI_OAUTH_CLIENT_SECRET="..."
export CPA_NORT_ANTIGRAVITY_OAUTH_CLIENT_ID="..."
export CPA_NORT_ANTIGRAVITY_OAUTH_CLIENT_SECRET="..."
```

如果只是导入 access-token-only JSON，通常不需要这些变量，除非你希望代理执行 OAuth 登录或刷新流程。

## API 使用

把 OpenAI-compatible 客户端指向代理：

```bash
export OPENAI_BASE_URL="http://127.0.0.1:8317/v1"
export OPENAI_API_KEY="any-configured-proxy-key"
```

可调用的兼容接口包括：

- `/v1/chat/completions`
- `/v1/responses`
- `/v1/models`

实际可用模型取决于你的 auth 文件和 `config.yaml`。

## 开发

```bash
gofmt -w .
go test ./...
go build -o test-output ./cmd/server
```

如果不需要编译产物，验证后可删除 `test-output`。

## 注意事项

- access-token-only JSON 仍然是临时凭据。上游 access token 过期后，请求可能返回 `401`，届时需要重新导入或重新登录。
- 如果想完全禁用所有 provider 的 OAuth/file auth 后台刷新检查，可以设置 `disable-auth-auto-refresh: true`。
- 不要提交真实 token 或 auth JSON。请保护好 `auths/` 目录和凭据文件。

## 上游项目

本项目基于 CLIProxyAPI：

- GitHub: <https://github.com/router-for-me/CLIProxyAPI>
- License: MIT

## 许可证

MIT License，详见 [LICENSE](LICENSE)。
