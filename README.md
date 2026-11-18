# sdk-ipad — WeCom iPad Protocol SDK

A standalone Go module (`module github.com/forage33/sdk-ipad`) that wraps every interaction with a WeCom iPad protocol service: **API calls**, the **callback dispatch pipeline**, the **callback data contracts**, and the **HTTP transport adapter**.

The module depends only on the standard library plus `go.uber.org/zap` and `github.com/tidwall/gjson`. It has no dependency on any host application, so it can be built, tested, and tagged independently. Repository: <https://github.com/forage33/sdk-ipad>.

Architecture notes (sharding, shared worker pool, hook mechanism, message flow, concurrency safety, and lifecycle) are documented in [ARCHITECTURE.md](ARCHITECTURE.md).

## Quick Start

### 1. Create an SDK instance

```go
cfg := &sdk_ipad.SdkConfig{
    ShardConfig:   sdk_ipad.DefaultShardConfig(),   // raise the shard count for large account fleets (see DefaultDispatcherConfig)
    WorkerPool:    sdk_ipad.DefaultWorkerPoolConfig(),
    PriorityQueue: sdk_ipad.DefaultPriorityQueueConfig(),
    Logger:        myLogger, // optional *zap.Logger; falls back to the global zap.L() when nil
}
sdk := sdk_ipad.NewSdkIPadWithConfig(cfg)
```

### 2. Call protocol endpoints

Every API method takes the protocol service address `host` as its first argument (for example `http://{host}:8083`) and the instance `uuid` as its second:

```go
// Initialize an instance, then register the callback URL.
resp, err := sdk.InItWXWork(host, vid, "", "", 0, "", "", "ipad", 0)
_, err = sdk.SetCallBackURL(host, uuid, sdk_ipad.CallbackConfig{CallbackType: sdk_ipad.CallbackTypeHTTP, URL: callbackURL})

// Send a text message.
_, err = sdk.SendTextMsg(host, uuid, sendUserID, "hello", false /* isRoom */)
```

### 3. Receive callbacks

The protocol service POSTs callbacks to an HTTP endpoint hosted by your application. The SDK ships a framework-agnostic handler (parse envelope → submit for dispatch → asynchronous retry on failure, always responding 200):

```go
// Mount it with gin.
engine.POST("/callback", gin.WrapF(
    sdk_ipad.CallbackHTTPHandler(adapter.HandleCallback, sdk_ipad.DefaultCallbackHandlerOptions()),
))
```

Business code declares the callback types it cares about through hooks (full example in `example_hooks.go`; end-to-end coverage in `dispatcher_test.go`):

```go
hook := myHook // implements sdk_ipad.Hook
hook.RegisterHandler(sdk_ipad.MsgCallback, handleMsgCallback)   // 102000 chat message
hook.RegisterHandler(sdk_ipad.LoginSuccessReturn, handleLogin)  // 104001 login succeeded
adapter.RegisterHook(hook)
```

Handlers consume strongly typed data from the **contract layer** and never touch raw JSON field names:

```go
func handleMsgCallback(ctx *sdk_ipad.HookContext, msg *sdk_ipad.CallBackMsg) error {
    env := msg.Envelope()              // common envelope: msgtype/sender/receiver/is_room/...
    switch env.MsgType {
    case sdk_ipad.ChatMsgText:
        text := msg.AsTextMsg()        // strongly typed payload
    case sdk_ipad.ChatMsgVoice:
        voice := msg.AsVoiceMsg()      // voice_id/aes_key/voice_time normalized here
    }
    return nil
}
```

## Repository Layout

### Entry points and configuration

| File | Responsibility |
|---|---|
| `sdk.go` | Package documentation and main entry point: the `SdkIPad` struct, the `SdkConfig` global configuration (shards / worker pool / priority queue (reserved) / logger injection), constructors, account context access, and the `HandleCallback` entry, plus a reference table of known protocol error codes |
| `sdk_interface.go` | `SdkInterface` definition: the SDK methods used by the service layer (`SdkIPad` implements it implicitly), provided for mocking or substitution |

### API calls (one file per endpoint)

Naming rule: `api_<domain>_<method_snake_case>.go` — e.g. `SendTextMsg` → `api_message_send_text_msg.go`, `RevokeMsg` → `api_message_revoke_msg.go`. Domain numbers match `docs-public/api`. Types shared across endpoints live in `api_common_types.go`.

| Domain | File prefix | Endpoints |
|---|---|---|
| 01 Initialization | `api_wx_init_*.go` | Instance initialization (including proxy settings), callback URL registration (HTTP/RABBITMQ), running instance queries, connection close — 7 |
| 02 Login | `api_login_*.go` | QR code retrieval, verification code check, automatic login, logout, secondary verification — 5 |
| 03 Account | `api_account_*.go` | Avatar update, profile update, personal QR code, ban details, business card, corp info — 8 |
| 04 Open Platform | `api_open_platform_*.go` | Corp user / open platform ID conversion, group ID / chatid conversion — 5 |
| 05 Labels | `api_label_*.go` | Label list, create/update/delete, label-contact association — 6 |
| 06 CDN | `api_cdn_*.go` | Image/video/file upload, large-file upload and download, WeChat file download, CDN init and auth — 12 |
| 07 Contacts | `api_contact_*.go` | Phone search, friend-add flows, internal/external contact lists, delete, remark, pin/mute/block — 23 |
| 08 Moments | `api_moments_*.go` | List/detail, publish, delete, comment, like — 7 |
| 09 Console Broadcast | `api_broadcast_*.go` | Broadcast task confirmation — 1 |
| 10 Chat Tags | `api_chat_tag_*.go` | Session tag create/update/delete — 1 |
| 13 Messaging | `api_message_*.go` | All message types, recall, mark-as-read, speech-to-text, sync, quote — 21 |
| 14 Groups | `api_group_*.go` | Group list, member management, creation, announcement, name, invitation, QR code, welcome message, leave, dissolve — 25 |

### Callback pipeline (receive, dispatch, concurrency)

| File | Responsibility |
|---|---|
| `callback_base.go` | Callback foundation: the `CallBackMsg` envelope (type/uuid/json), callback type constants (100xxx login / 102xxx message / 115xxx group / 116xxx contact / 130xxx label), and the internal `SendMsg` structure |
| `chat_msgtype.go` | Chat `msgtype` constants (ChatMsgText/Voice/Video/Revoke...), used to branch message dispatch |
| `callback_dispatcher.go` | `CallbackDispatcher`: hook registration (per message type, ordered by hook priority within a type), statistics (`DispatcherStats`), and message consumption (message priority is reserved and not used for scheduling) |
| `dispatcher_adapter.go` | `DispatcherAdapter` convenience layer: wires Dispatcher + ShardManager + WorkerPool so business code only calls `HandleCallback(msg)` |
| `shard_manager.go` | `ShardManager`: consistent hashing assigns accounts to fixed shards; manages `AccountContext` lifecycle, expiry cleanup, and per-account callback routing |
| `shared_pool.go` | `SharedWorkerPool`: dynamically scaling goroutine pool with Min/MaxWorkers, queue capacity, and a priority queue configuration (reserved, not implemented) |
| `callback_http.go` | `CallbackHTTPHandler` transport adapter: a standard `net/http` handler (JSON envelope parsing → dispatch → asynchronous retry, 3 attempts with exponential backoff → always responds 200), decoupled from web frameworks (mount with `gin.WrapF`) |
| `example_hooks.go` | Example hook implementations and the hook manager, serving as a template for business hooks |

### Callback contract layer (single source of truth for protocol fields)

| File | Responsibility |
|---|---|
| `callback_payload_msg.go` | Message callback (102000) contracts: `MessageEnvelope` (8 fields), `RevokePayload` for recall targeting, 12 message payload types (system/text/GIF/mini program/link/channel/location/red packet/card/media file/video/voice), and lenient `As*` accessors (gjson semantics, zero values when missing); the video `aes_key`/`aeskey` dual-name compatibility is resolved here |
| `callback_payload_account.go` | Contracts for the remaining domains: login (5 payload types such as `AsScanReturn`/`AsLoginSuccess`, with **strict** `json.Unmarshal` semantics returning an error); group (`RoomID`/`AsNewRoom`/`GroupMemberList`/`AsGroupNameChange` with three-level fallback); contact (`AsFriendChangeList`/`AsInnerLabelChangeList`/`AsContactEventMsg`); disconnect (`DisconnectVid`, accepting both string and number) |

> Parsing semantics: login payloads historically used `json.Unmarshal` and discarded malformed input, so strict semantics are preserved. Group, contact, disconnect, and message payloads historically read fields individually through gjson, so lenient zero-value semantics are preserved. **New accessors must follow the semantics of their domain.**

### Infrastructure

| File | Responsibility |
|---|---|
| `http_client.go` | HTTP client wrapper: JSON POST, automatic retry on network errors, and `optimizedTransport` tuned for high concurrency across protocol servers (connection-pool sizing rationale is documented in comments) |
| `flex_int64.go` | `FlexInt64` tolerant type: protocol integer fields may arrive as either number or string |

### Tests

| File | Responsibility |
|---|---|
| `dispatcher_test.go` | Dispatcher cases (hook registration/consumption, priority queue tags) |
| `dispatcher_adapter_test.go` | Adapter cases (isolated instance, no global singleton) |
| `hook_manager_test.go` | Hook manager cases |
| `shard_manager_test.go` | Shard manager cases and benchmarks |
| `shared_pool_test.go` | Worker pool cases and benchmarks |
| `callback_payload_msg_test.go` | Message contract cases (lenient parsing, dual-name compatibility, string views) |
| `callback_payload_account_test.go` | Login/group/contact/disconnect contract cases (strict semantics, three-level fallback, dual-type compatibility) |
| `callback_http_test.go` | Callback HTTP transport cases (response protocol, asynchronous retry) |
| `flex_int64_test.go` | `FlexInt64` dual-format parsing cases |
| `group_qrcode_test.go` | Group QR code path normalization cases |

### Documentation

| Path | Content |
|---|---|
| `ARCHITECTURE.md` | Architecture notes: sharding, shared worker pool, hook mechanism, message flow, concurrency safety, and lifecycle (repository root) |
| `docs-public/` | Public documentation generated locally by `tools/docgen`; not committed to this repository |

## Configuration Reference (SdkConfig)

| Field | Description |
|---|---|
| `ShardConfig` | Shard count, max accounts, account TTL, cleanup interval (`ShardManager`) |
| `WorkerPool` | Worker pool min/max, queue capacity, scale-up check interval (`SharedWorkerPool`) |
| `PriorityQueue` | Priority levels, per-level capacity, high-priority capacity (callback dispatch; **reserved** — scheduling is not implemented and the queue is a single FIFO today) |
| `Logger` | `*zap.Logger` injected by the host (field style); falls back to `zap.L()` when nil. The SDK does not depend on any host logging wrapper |

## Standalone Module and Releases

- The repository carries its own `go.mod`/`go.sum`; run `go build ./...`, `go test ./...`, and `golangci-lint run ./...` inside the repository directory.
- Consume it with `go get github.com/forage33/sdk-ipad@v0.1.0` (use a `replace` directive pointing at a local copy during integration).
- Release by pushing a semantic version tag (for example `v0.1.0`); consumers upgrade by tag and the import path stays unchanged.
- When protocol fields evolve, change **this repository only** (prefer the contract-layer files).

## Contact

- QQ: 1801533361
- GitHub Issues: <https://github.com/forage33/sdk-ipad/issues>

## Disclaimer

- This project is a generic HTTP client wrapper. It only provides the ability to communicate with a self-hosted or duly authorized protocol service, and contains no protocol implementation, injection, or bypass code.
- This project is not affiliated with, endorsed by, or authorized by Tencent. "WeCom", "WeChat Work", and "WeChat" are trademarks of Tencent and are used here only to describe the applicable scenario objectively.
- Users must ensure they have lawful authorization for the accounts, data, and protocol services they operate, and must comply with the terms of service of the relevant platform and all applicable laws and regulations. Any consequences arising from the use of this project are the sole responsibility of the user.
- This project is provided "as is", without warranty of any kind, express or implied.
