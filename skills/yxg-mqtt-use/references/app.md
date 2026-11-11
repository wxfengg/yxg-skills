# App：先确认运行环境与真实引擎

“uni-app”“Android”“iOS”不是选择 /browser 或 /native 的充分依据。确认 MQTT 代码运行在 WebView、服务 JS 引擎还是原生模块，以及实际 WebSocket/SocketTask/API 能力。

| 情况 | 接线 |
| --- | --- |
| H5 页面或具有真实原生 WebSocket 的 WebView | 验证平台支持后用 /browser，继续读[网页](browser.md) |
| 微信小程序构建目标 | 用 /weixin，继续读[微信](weixin.md)；不要因同一仓库也构建 App 就共用微信工厂 |
| 原生 MQTT SDK／桥已存在 | 用 /native 或根 createSession，核对 EngineFactory 合同 |
| 只有 uni.connectSocket，没有已验证的浏览器 WebSocket 或 EngineFactory | 不宣称开箱可用；评估已有适配能力和授权范围，不能假扮 wx 或 Browser WebSocket |
| 没有原生桥且必须 TCP/TLS 原生客户端 | 本包不含原生实现；明确缺口，按请求决定另行实现桥或选择其它获准路径 |

先从仓库推断运行上下文与版本、同一上下文的 WebSocket 能力、已有通信／Handler 和 App 生命周期入口。只有无法确定时才问这些最少信息，不要求用户粘贴凭据或整仓代码。

本库 /native 只是薄接口：

```ts
import { createNativeClient, type EngineFactory } from '@yxg/mqtt/native';
import type { ClientOptions } from '@yxg/mqtt';
export function createForNative(options: ClientOptions, bridge: EngineFactory) {
  return createNativeClient(options, bridge);
}
```

不能用空 EngineFactory、固定 true、静态 broker-acked 或虚假 SUBACK 把接入标为完成。未获准时不新装原生插件，不修改权限／构建工具来掩盖缺少的能力。没有桥时仍可完成调用点盘点、类型边界和路由／凭据映射，向用户具体说明还需哪种真实 SDK 回调。

## 桥接合同

读实际 `dist/index.d.ts`／`dist/native.d.ts`：EngineFactory 包含 capabilities 与 create(context)，每次 create 返回新的独占 EnginePort。不要在不同登录／会话间共用原生单例连接。

- capabilities 如实声明 reconnectOwner、transportCompletion、brokerAck、subscriptionAck、qos1Replay、canRetire。唯一重连所有者；不能核心和 SDK 同时重连。
- 每次物理连接，包括原生自动重连，先 `context.beginConnection()`；null 禁止连接，否则保存 epoch，所有回调附该代次。
- emit 支持 connected/closed/error/message；error 必须 code，message 必须 topic/payload/qos/retain/dup。TypeScript EngineEvents 会校验载荷。
- port 实现 connect、publish、subscribe、unsubscribe、close、destroy；publish callback 必须关联真实操作和正确证据。subscribe 返回真实授权 QoS，unsubscribe 按真实确认完成。
- close 返回真实 remoteClosed；destroy 停止旧重连和重传、清空其 Store、隔离迟到回调。无法强制退役的旧桥不能谎称 canRetire=true。

如果现有 SDK 只有“调用成功”的布尔返回，不能推导 Broker ACK／设备 ACK；相应能力不可用，业务请求应在提交前拒绝不支持的语义。原生 SDK 的 ACK、后台冻结、线程回调、重连和强制释放要独立验证。

保留项目权限、网络／后台策略和设备业务 Handler。使用 App 的 onShow/onHide/登出入口接线；不要按页面数量增加原生连接。按[分层验收](validation.md)记录真机、系统版本和 SDK 版本，不能继承浏览器／微信测试结论。
