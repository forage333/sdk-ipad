# sdk-ipad 使用文档

> 企业微信 iPad 协议 SDK 的对外文档，仅收录 SDK 已封装的接口与回调事件。

`github.com/forage33/sdk-ipad` 封装与企微 iPad 协议服务的交互：主动调用 API、回调分发机制、回调数据契约与 HTTP 传输适配。
SDK 仅依赖标准库、`go.uber.org/zap` 与 `github.com/tidwall/gjson`，不依赖任何宿主项目代码。

## 文档导航

| 文档 | 内容 |
|---|---|
| [API 接口文档](api/README.md) | 11 个域 · 109 个接口：初始化、登录、账号、标签、CDN、联系人、朋友圈、控制台群发、聊天标签、消息、群操作 |
| [回调消息文档](callback/README.md) | 6 个域 · 57 个事件：登录、消息、群、联系人、标签变动、聊天标签与断开 |
| [架构说明](ARCHITECTURE.md) | 分片管理、共享 Worker 池、Hook 机制、消息处理流程、并发安全与生命周期 |

## 环境要求

- Go 1.25 及以上
- 可访问企业微信 iPad 协议服务的网络环境

## 安装

```bash
go get github.com/forage33/sdk-ipad@v0.1.0
```

## 快速开始

### 1. 创建 SDK 实例

```go
cfg := &sdk_ipad.SdkConfig{
    ShardConfig:   sdk_ipad.DefaultShardConfig(),
    WorkerPool:    sdk_ipad.DefaultWorkerPoolConfig(),
    PriorityQueue: sdk_ipad.DefaultPriorityQueueConfig(),
    Logger:        logger, // 可选，*zap.Logger；nil 时回退全局 zap.L()
}
sdk := sdk_ipad.NewSdkIPadWithConfig(cfg)
```

### 2. 初始化实例并设置回调地址

所有 API 方法第一个参数为协议服务地址 `host`，第二个参数为实例 `uuid`：

```go
resp, err := sdk.InItWXWork(host, vid, "", "", 0, "", "", "ipad", 0)
if err != nil {
    return err
}
_, err = sdk.SetCallBackURL(host, resp.Data.UUID, sdk_ipad.CallbackConfig{
    CallbackType: sdk_ipad.CallbackTypeHTTP,
    URL:          callbackURL,
})
```

### 3. 调用协议接口

```go
// 发送文本消息，isRoom 表示是否为群消息
_, err = sdk.SendTextMsg(host, uuid, sendUserID, "hello", false)
```

### 4. 接收回调

协议服务将回调 POST 到调用方暴露的 HTTP 端点。SDK 提供与 Web 框架解耦的标准 handler
（信封解析 → 分发提交 → 失败异步重试，始终应答 200）：

```go
engine.POST("/callback", gin.WrapF(
    sdk_ipad.CallbackHTTPHandler(adapter.HandleCallback, sdk_ipad.DefaultCallbackHandlerOptions()),
))
```

业务侧通过 Hook 声明关心的回调类型：

```go
hook.RegisterHandler(sdk_ipad.MsgCallback, handleMsgCallback)   // 102000 聊天消息
hook.RegisterHandler(sdk_ipad.LoginSuccessReturn, handleLogin)  // 104001 登录成功
adapter.RegisterHook(hook)
```

处理函数从契约层取强类型数据，不接触原始 JSON 字段名：

```go
func handleMsgCallback(ctx *sdk_ipad.HookContext, msg *sdk_ipad.CallBackMsg) error {
    env := msg.Envelope() // 通用信封：msgtype/sender/receiver/is_room/...
    switch env.MsgType {
    case sdk_ipad.ChatMsgText:
        text := msg.AsTextMsg()
    case sdk_ipad.ChatMsgVoice:
        voice := msg.AsVoiceMsg()
    }
    return nil
}
```

## 配置参考（SdkConfig）

| 字段 | 说明 |
|---|---|
| `ShardConfig` | 分片数、单分片最大账号数、账号上下文 TTL 与清理间隔 |
| `WorkerPool` | Worker 池最小/最大数量、队列容量、扩容检查间隔 |
| `PriorityQueue` | 优先级档位数与容量（预留配置，当前调度为单一 FIFO） |
| `Logger` | `*zap.Logger`，由调用方注入；nil 时回退 `zap.L()` |

## 相关链接

- 源码仓库：<https://github.com/forage33/sdk-ipad>
- 本文档仅覆盖 SDK 已封装的接口与回调；协议字段语义以 SDK 契约层定义为准。
