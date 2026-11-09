# 可选 DeviceCommunication：设备路由与 MQTT/HTTP

仅当用户需要统一设备路由／双通道调度，或目标已经采用此层时使用。它自己拥有 SessionManager、订阅和 HTTP 调度；禁止再为同一资源叠加 manager／重发队列。简单单连接保留 Session 足够。

读取实际包 `dist/communication/index.d.ts`、`docs/guide/communication.md`，按已有项目的请求层和 Handler 映射，不从示例猜测服务路径、错误码或协议。

## 工厂输入映射

| 参数 | 宿主提供内容 |
| --- | --- |
| createMqttSession(options) | 用目标平台工厂补 transport／platform，返回新的未连接 session |
| describeDevice(device) | 返回 null 排除不支持设备，或 id/channel 路由；MQTT 带完整 connection、publishTopic、subscribeTopic、subscriptionQos |
| protocol | encodeMqtt、decodeMqtt、matchesDevice、normalizeIncoming；按真实协议，不能一律 JSON |
| http | pull、transmit、可选 classifyError；沿用现有请求封装和认证 |
| policy | background；使用 MQTT 则填 mqtt；使用 HTTP 则填 http，按类型完整填写 |
| callbacks | onReady/onMessage/onDelivery/onAuthFailure/onError，按业务处理并复查归属 |

mqtt policy 包含 session、qos、retain、subscriptionQos、connectTimeoutMs、subscribeTimeoutMs、cleanupTimeoutMs、retryDelaysMs、offlineTtlMs；http policy 包含 pollIntervalMs、requestTimeoutMs、maxPendingMessages、capacityIncludesInFlight、waitTimeoutMs、registrationCheck、resetReadyOnPullError、readyOrder、queryRequiresIdle。沿用已批准策略，没有固定的“正式默认参数”。不为了省字段拷贝他项目超时。

Topic 必须精确；同 host 不等于同身份，不以当前选中设备推断所有回包。matchesDevice 在共享 Topic 时只允许唯一命中；设备列表删除／切通道使旧归属失效，不迁移已提交或 unknown 命令。

## HTTP 适配要点

- HttpRequestContext 的 session.data 是宿主给的 JSON 快照；不要硬编码 token 属性名或日志输出该对象。
- 请求创建时注册 `context.onCancel(() => task.abort())`，结束后注销。宿主请求层不得自动重试控制请求；物理 abort 不保证服务未执行。
- transmit 仅在 HTTP 状态与业务受理都确认后返回 `{accepted:true}`；不是简单“HTTP 200 就成功”。无法证明未受理的失败按 unknown，不能把超时变成 rejected。
- pull 返回真实消息数组；协议归一化和设备归属检查仍由项目提供。不存在通用的 /pull-msg 或 /transmit-msg 后端合同。

## 方法与语义

- startSession(context, devices)：context.key 为不透明登录代次、data 为 JSON 快照、isCurrent 为实时账号及列表归属检查。它会设 active=true，宿主拦截后台迟到调用；完成只等激活／MQTT连接，不等 SUBACK、HTTP ready。
- syncDevices：传完整列表，非法列表同步抛错，合法列表先撤销旧路由再异步激活；整个调用放 try/catch。
- ensureConnected：MQTT 只等目标连接，HTTP 只检查注册；不等同 isReady，更不是设备在线。
- refreshSubscription：安排刷新，不等新 SUBACK；无路由／HTTP 为 no-op，delay 非法同步抛错。用 onReady/isReady 判断就绪。
- setAppActive(false)：dispose 清理，回前台需要重新 startSession；pause-http 保留 MQTT，仅恢复 HTTP 调度。setAppActive(true) 不自动重建被 dispose 的会话。
- disposeSession：立即作废归属，等待在途工厂／清理；退出先撤销用户业务入口再等待它。
- captureRoute/isCurrent：返回／提供归属 guard；await 后写 Store 前复查，不能代替 isReady。

## send 和通知

send(id,payload,options) 返回 CommunicationTicket，不返回设备执行结果。未知／过期入口路由返回 OFFLINE Ticket，不触发 onDelivery（没有合法通道）。有效入口路由的每个结算 Ticket，包括 accepted=false 的配置／编码／容量拒绝，异步 onDelivery 一次。done.catch 与 onDelivery 不重复弹提示，必要时按 operationId 去重；done 始终处理。

| options | MQTT | HTTP |
| --- | --- | --- |
| qos/retain/allowOffline | 生效 | 忽略 |
| timeoutMs/signal | 生效 | INVALID_CONFIG；HTTP 使用 policy 和 cancel |
| query | 忽略 | 生效 |
| isCurrent/beforeHttpSend | INVALID_CONFIG | 生效 |
| dropIfOffline/disableRetry | 禁业务离线等待，不禁协议 QoS1 恢复 | 限制未就绪时等待 |

已知通道用 MqttSendOptions/HttpSendOptions；SendOptions 为兼容保留组合形状，类型不能从动态 id 自动知道通道。
