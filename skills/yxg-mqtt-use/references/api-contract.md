# 公共合同与接线判断

先读目标制品的 `package.json`、exports 指向的声明／实现和对应 `docs/guide/api.md`。下列名称是 `0.1.0` 系列定位线索，不是跨版本签名保证。源码、归档、安装包和宿主输出可能不同；需要新能力时逐层核对，不能只看版本字符串。

## 入口与配置

| 入口 | 定位 API |
| --- | --- |
| @yxg/mqtt | createSession、createSessionManager、MqttError |
| /browser | createBrowserClient |
| /weixin | createWeixinClient |
| /native | createNativeClient、EngineFactory |
| /communication | createDeviceCommunication、CommunicationError |

这些入口采用 ESM 命名导出。返回 MqttSession 的工厂不自动连接，持有实例后调用 session.connect；Manager 先 ensure 取得 Session；DeviceCommunication 使用 startSession 激活，完成不代表 SUBACK。业务不引用未导出的包子路径或 dist hash chunk；构建适配若需识别内部模块，见[微信](weixin.md)。源码 example 不保证随 tgz 分发，也不是公共导出，不能生成不存在的示例 import。

- 从实际类型填写 connection、policy；浏览器／微信还需 transport。`sessionKey` 是非敏感登录代次，不是 token 或微信 session_key。配置沿用项目授权策略，没有统一生产默认值。
- 时间字段通常为 ms，`keepaliveSeconds` 为秒；容量和字节使用整数。transport 发送／接收限额与业务队列限额独立。
- 本基线支持 MQTT 3.1.1、QoS 0/1 和固定重连（maxMs=baseMs、jitterRatio=0，stableResetMs 必填但无行为作用）。不要生成 MQTT5、QoS2、持久队列或指数退避配置；未来制品变化时重新核对这些限制。
- 接收 payload 为 Uint8Array，发送按合同用 string/Uint8Array；按真实协议编解码，不默认将二进制 JSON.parse。

## 生命周期与归属

1. 创建前保存可撤销的登录／路由代次；异步工厂返回后先复查归属和前后台意图，失效就销毁本次局部实例。失败清理不能引用可能已经换成新实例的全局变量。
2. 尽早让拥有者持有 Session，connect 前应用初始化期间最新网络提示，并先注册消息／状态／订阅监听。不要在 import 时连接，也不要用无法撤销的全局 start Promise 屏蔽退出清理。
3. connect 等待失败不会撤销连接意图；放弃需 disconnect 或 destroy。disconnect 后可重连，destroy 后新建。快速恢复若撞上未结束的 disconnect，应等关闭屏障结束后按当前意图恢复，不能吞掉 BUSY 后丢失恢复机会。
4. 退出先撤销登录／路由归属，再等待资源销毁；晚到工厂、HTTP、设备列表和异步元数据都复查归属。涉及播放／页面状态的 await 后还应检查业务操作代次，不能让旧结果覆盖新的暂停／停止或在卸载后提示。
5. `on` 返回取消监听函数；页面只释放自己的监听／句柄，不销毁共享连接。核心事件同步调用，不接管异步回调拒绝，宿主须处理并在 await 后复查。

`setNetworkAvailable(false)` 只暂停重试，不主动关闭现有 socket；后台断开沿用宿主策略。`replaceConnection` 保留 scope／订阅，不能代替切账号；`invalidateScope` 有在途操作时可能退役整条共享连接。

多连接采用 SessionManager 时，ensure 不自动 connect，只有完整配置相同才能合并，不能仅按 host。ensure 可能同步抛错或异步拒绝，将整个 await 放入 try/catch。manager 拥有 Session 的配置与销毁，通过 remove/ensure 改配，不绕过它调用 destroy/replaceConnection/updatePolicy。manager.destroy 等待在途创建与清理，是终态；下次登录新建。工厂取消上下文是否可用以实际制品为准；即使支持取消，迟到资源仍须清理。

## 首次订阅与恢复

- `subscribe` 可同步抛错，返回句柄 `{id, ready}` 后仍须处理 ready 拒绝。ready 只结算一次；首次等待与持续 `subscription` 事件共同维护当前连接状态，历史 ready 成功不是重连后的确认。
- 采用当前 `connectionEpoch`、有效句柄／scope、路由代次和连接状态判断 confirmed；Topic 广播不证明已超时或替换的句柄恢复。共享 Topic 且请求 QoS 未变化时，新句柄可能直接复用已有确认，只结算 ready 而不再广播 confirmed。
- 先注册消息监听，再订阅；同帧 SUBACK 后的消息可能早于 ready.then。按消息当前 epoch 和有效 scope／路由接收，不能只因宿主 ready 标志尚未更新就丢掉首条消息。
- 拒绝后先区分协议拒绝、调用超时、取消和会话清理，依据实际合同确认句柄是否仍保留；不能假定所有 ready 拒绝都会删除句柄。显式重试先作废旧 attempt 并释放旧句柄，Topic 修正同时撤销旧路由；不在 rejected 回调里形成忙循环。
- 当前订阅确认后，宿主可按仍有效的业务意图补查；订阅恢复本身不授权补发旧控制命令。需跨连接失效的异步 guard 必须捕获当次 epoch，不能在下一次就绪时重新变真。此规则不改变[协调器的 isCurrent 合同](communication.md)。
- `unsubscribe` 接收句柄 id，不是 Topic；共享句柄可能保留远端订阅。超时句柄已移除时，释放旧 id 不证明收到 UNSUBACK；需要清理屏障的流程须检查实际退订结果。

订阅门禁使用实际合同支持的 `requireSubscriptions`，并对应有效 scope；它不能替代消息监听或业务回复。首次／重连拒绝、显式重试和首帧消息的验收见[恢复场景](validation.md#按改动选择恢复场景)。

## 提交与结果证据

publish 返回 Ticket，始终处理 `ticket.done`，包括 accepted=false。离线缺省拒绝；获准排队时 `offline.expiresAt` 是绝对时间，只约束首次提交。

| 值 | 能确认的事实 |
| --- | --- |
| accepted | 本地接纳，未证明发送或业务成功 |
| transport-sent | QoS0 传输完成；浏览器为发送缓冲排空 |
| broker-acked | QoS1 PUBACK，未证明设备处理 |
| server-accepted | HTTP 及业务服务明确受理，未证明设备处理 |
| not-submitted | 未提交或已明确证明未受理，是否新发仍由业务授权与有效期决定 |
| submitted-unknown | 已提交但结果未知，不得自动转 HTTP／跨会话补发 |

cancel 提交前可移除，提交后仅停止本地等待。retryable 是错误分类，不是幂等或重发许可。业务 ACK、在线状态、消息关联均由项目协议定义；同步 facade 的 boolean 也不能升级为“设备成功”。

## 容量与业务接纳

仅在实际制品支持时使用 `capacity` 事件、`snapshot().capacity`、CAPACITY 的 `reason/requiredBytes/capacity`。核对包内声明与实现；旧制品缺能力时说明限制，保留已批准策略，不静默升级或自造轮询／第二队列。[协调器](communication.md) 未必透传核心详情。

遇 accepted=false 时结合 evidence、订阅和容量证据判断，不能把所有 CAPACITY 都当暂满：

| 条件 | 处理判断 |
| --- | --- |
| 数量／累计字节暂满 | 仅对获准且仍有效的未提交意图等待真实容量释放；唤醒后重查身份、路由、订阅和有效期 |
| 单条超过 inflight 字节上限 | 当前策略下无法提交，按实际所需字节判断，不擅自放宽资源策略 |
| 单条超过 outbox 字节上限 | 不能靠 outbox 变空排入；是否可在连接／订阅就绪后直接提交另按实际合同判断，不因 outbox-bytes 无限重试 |
| outbox 禁用且订阅未确认 | 恢复订阅门禁；容量通知不能代替当前 SUBACK，outbox-disabled 本身也不是唯一根因证据 |

- 宿主已有待发列表时，明确“选中、调用、接纳、结算”的出队位置。容量拒绝发生在接纳前；宿主先出队再忽略拒绝是宿主丢弃，不能定性为库丢消息。是否保留／重试仍按业务策略。
- 防止容量释放发生在拒绝处理前而漏掉唤醒；按事件代次或重新检查状态处理竞态，避免忙轮询。进入 accepted=true 后结束该接纳重试意图，done 为 unknown 也不因此再次发送。
- `wait-only` 或 ticket.done 结束不等于释放协议容量；迟到 ACK／实际完成或引擎退役才可能释放。不要以 Promise 数量推算可发送槽位。
- 核验同步接纳预留、策略缩限及共享连接下不同设备的进展；发现旧制品缺陷先确认适用性和升级授权，不要求所有宿主重复打库补丁。

## 无回复时的最小取证链

网络提示、引擎加载、连接、当前 SUBACK、业务查询提交、新回复是不同阶段；每层记录可关联的非敏感操作／代次及证据，不能从 connected 或本地 accepted 推导设备在线。

若状态查询无可见回包、另一协议查询却能回复，或 HTTP transmit 受理但 pull 为空：先检查过滤前接收、解码和路由拒绝原因，再用脱敏关联信息核对后端投递／拉取链路。空数组只说明这次未取到消息。证据不足时保留待核实项，不能归因于 QoS、库、设备或后端中的某一方；不为诊断输出凭据、完整连接配置或原始敏感载荷。
