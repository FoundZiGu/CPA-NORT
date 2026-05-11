# CPA-NORT

[中文](README.md) | English

CPA-NORT is a fork of [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) focused on running imported OAuth JSON credentials that may only contain a valid `access_token` and no usable `refresh_token`.

The fork keeps the original CLIProxyAPI compatibility surface while improving behavior for no-refresh-token auth files:

- OpenAI/Codex compatible API endpoints
- Gemini, Claude, Codex, Antigravity, Kimi, and OpenAI-compatible provider support inherited from CLIProxyAPI
- Multi-account auth loading from JSON files
- Round-robin and session-affinity routing
- Streaming, non-streaming, and supported WebSocket transports
- No-refresh-token OAuth JSON support without background refresh loops
- Optional global switch to disable OAuth/file auth auto-refresh checks

## Why This Fork

Some imported auth JSON files contain an access token that still works, but either omit `refresh_token` or contain a refresh token that cannot be reused. In the upstream-style auto-refresh flow, those credentials can be repeatedly scheduled for background refresh checks and may produce noisy failures or intermittent stalls.

CPA-NORT treats auth files without `refresh_token` as access-token-only credentials:

- Requests continue to use the existing `access_token`.
- Real upstream `401` responses are still detected normally during actual API calls.
- Background OAuth refresh scheduling is skipped for providers that require a refresh token.
- The proxy does not call the refresh endpoint just to prove a token can be refreshed.

## Quick Start

```bash
go build -o cli-proxy-api ./cmd/server
./cli-proxy-api --config config.yaml
```

On Windows PowerShell:

```powershell
go build -o cli-proxy-api.exe ./cmd/server
.\cli-proxy-api.exe --config config.yaml
```

## Configuration

Create or edit `config.yaml`:

```yaml
host: "127.0.0.1"
port: 8317
auth-dir: "./auths"

# Optional safety switch. When true, all background OAuth/file auth refresh checks are disabled.
disable-auth-auto-refresh: false

request-retry: 3
max-retry-credentials: 0
max-retry-interval: 30
```

Put OAuth JSON files under `auth-dir`, for example:

```text
auths/
  token_user@example.com.json
```

For Codex/OpenAI access-token-only files, the important fields are typically:

```json
{
  "type": "codex",
  "access_token": "eyJ...",
  "email": "user@example.com",
  "expired": "2026-05-11T12:00:00Z"
}
```

If `refresh_token` is missing, CPA-NORT will skip background refresh scheduling for providers that cannot refresh without it.

## OAuth Client Credentials

CPA-NORT does not hard-code Google OAuth client credentials in the repository. If you need to run interactive Gemini CLI or Antigravity OAuth login flows, provide the client credentials through environment variables:

```bash
export CPA_NORT_GEMINI_OAUTH_CLIENT_ID="..."
export CPA_NORT_GEMINI_OAUTH_CLIENT_SECRET="..."
export CPA_NORT_ANTIGRAVITY_OAUTH_CLIENT_ID="..."
export CPA_NORT_ANTIGRAVITY_OAUTH_CLIENT_SECRET="..."
```

Imported access-token-only JSON files do not need these variables unless you want the proxy to perform OAuth login or refresh flows.

## API Usage

Point OpenAI-compatible clients at the proxy:

```bash
export OPENAI_BASE_URL="http://127.0.0.1:8317/v1"
export OPENAI_API_KEY="any-configured-proxy-key"
```

Then call compatible endpoints such as:

- `/v1/chat/completions`
- `/v1/responses`
- `/v1/models`

Exact available models depend on your auth files and `config.yaml`.

## Development

```bash
gofmt -w .
go test ./...
go build -o test-output ./cmd/server
```

Remove `test-output` after compile verification if you do not need it.

## Notes

- Access-token-only JSON files are still temporary. Once the upstream access token expires, requests may return `401` and the account must be re-imported or re-authenticated.
- If you want to completely disable background OAuth/file auth refresh checks for every provider, set `disable-auth-auto-refresh: true`.
- Keep tokens and auth JSON files private. Do not commit `auths/` or real credentials.

## Upstream

This project is based on CLIProxyAPI:

- GitHub: <https://github.com/router-for-me/CLIProxyAPI>
- License: MIT

## License

MIT License. See [LICENSE](LICENSE).
