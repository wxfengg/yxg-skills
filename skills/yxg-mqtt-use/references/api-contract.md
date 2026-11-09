# 公共合同与最小接线（0.1.0 基线）

实际包内 dist/*.d.ts、dist/{browser,weixin,communication}/index.d.ts 和 docs/guide/api.md 是签名依据。源码仓库有完整 example；示例源码不保证随 tgz 分发，不能生成指向不存在 example 的 import。

## 入口与配置

| 入口 | API |
| --- | --- |
| @yxg/mqtt | createSession(options, engineFactory)、createSessionManager(factory)、MqttError |
| /browser | createBrowserClient(options): Promise<MqttSession> |
| /weixin | createWeixinClient(options): Promise<MqttSession> |
| /native | createNativeClient(options, bridgeFactory): MqttSession |
| /communication | createDeviceCommunication(options)、CommunicationError |

全为 ESM 命名导出。工厂创建不连接；先持有生命周期对象，再显式 connect。不要 require 未导出的包子路径或提前加载 mqtt。

ClientOptions 必填结构：

```ts
{
  sessionKey, // 非敏感登录代次标识，不是 token 或微信 session_key
  connection: {
    endpoint, clientId, username, password, // 后两项按认证需要
    protocolVersion: 4, clean, keepaliveSeconds, connectTimeoutMs
  },
  policy: {
    reconnect: { baseMs, maxMs, jitterRatio: 0, stableResetMs },
    outbox: { maxMessages, maxBytes },
    maxInflight, maxInflightBytes, maxInflightAgeMs,
    operationTimeoutMs, closeTimeoutMs
  }
}
```

以上为字段清单，不是可直接执行的配置。所有数值来自项目策略；没有生产默认值。仅支持固定重连：maxMs=baseMs，jitterRatio=0；stableResetMs 必填但没有行为作用。时间为 ms，keepaliveSeconds 为秒；容量／字节选整数。MQTT 3.1.1、QoS 0/1；不能生成 MQTT5、QoS2、持久化队列、指数退避配置。

浏览器／微信另需 transport：maxSendBytes、maxReceiveBytes、sendTimeoutMs、closeTimeoutMs。其限额与业务队列限额独立，不能通过复制示例值宣称适合生产。

## 可直接改用的单会话启动片段

下面是由宿主提供工厂、Topic、scope、协议处理函数的函数体；放入原项目服务层，按实际类型落地。宿主必须在调用前保存可撤销的登录代次，退出时先使 isCurrent=false；使用前后台状态时也将其纳入启动 guard。

```js
export async function openSession({ create, isCurrent, getNetworkAvailable, topic, scope, onMessage }) {
  const session = await create(); // 工厂闭包提供平台和全部配置
  const current = () => isCurrent() === true;
  try {
    if (!current()) throw new Error('SESSION_REPLACED');
    session.setNetworkAvailable(getNetworkAvailable()); // 应用初始化期间最新网络提示
    session.on('message', message => {
      if (current()) onMessage(message); // 异步处理需自己捕获拒绝并在 await 后复查
    });
    await session.connect();
    if (!current()) throw new Error('SESSION_REPLACED');
    const subscription = session.subscribe({ topic, qos: 0, scope });
    await subscription.ready;
    if (!current()) throw new Error('SESSION_REPLACED');
    return { session, subscription };
  } catch (error) {
    await session.destroy(); // 仅销毁本次局部实例，不能销毁新的全局实例
    throw error;
  }
}
```

此片段采用订阅 QoS0 作演示，按项目合同调整。退出时不仅置 guard=false，还应 await 已取得实例.destroy；创建仍在途中时，guard 使迟到实例自毁。需要立即取消正在连接的初始化时，宿主应尽早持有 session／AbortSignal 或使用 manager，使 teardown 覆盖在途创建；不要让示例成为无法终止的全局 start Promise。不要在 import 时发起连接。已有登录闭包不要重复套多层无用抽象。

## 收发与清理

- connect 等待失败不停止连接意图；放弃须 disconnect 或 destroy。disconnect 后可重连，destroy 后应新建。
- subscribe 返回 `{id,ready}`；ready 只结算一次，重连后用 subscription 事件确认当前 connectionEpoch 的 SUBACK。unsubscribe 接收句柄 id，不是 Topic；一个句柄释放不一定远端退订。
- publish 返回 Ticket；调用示例：`session.publish({topic, payload, qos, retain, scope, requireSubscriptions:[upTopic]}, {timeoutMs, signal})`。处理 ticket.done，即使 accepted=false。offline 缺省拒绝；允许排队需 `{expiresAt: 绝对时间}`，只约束首次提交。
- payload 用 string/Uint8Array，接收 Uint8Array；按真实协议编解码，不默认对所有二进制 JSON.parse。
- on 返回取消监听函数；页面监听由页面释放。核心事件监听器同步调用，不接管异步回调拒绝。
- setNetworkAvailable(false) 只暂停重试，不主动关已有 socket。网络恢复不等于 Broker 可达，更不等于设备在线。
- 退出先撤销登录／路由归属，再 await destroy，拦截迟到 HTTP、设备列表及页面回调。replaceConnection 保留 scope／订阅，不能代替切账号；invalidateScope 有在途操作时可能退役整条共享连接。

## 结果语义

| 值 | 事实 |
| --- | --- |
| accepted | 本地接纳，不是发送成功 |
| transport-sent | QoS0 传输完成；浏览器为发送缓冲排空 |
| broker-acked | QoS1 PUBACK，不是设备 ACK |
| server-accepted | HTTP 及业务服务明确受理，不是设备 ACK |
| not-submitted | 未提交或已明确证明未受理；是否新发仍由业务决定 |
| submitted-unknown | 已提交但结果未知，不得自动转 HTTP／跨会话补发 |

cancel 提交前可移除，提交后仅停止本地等待。retryable 只是错误分类，不保证可安全重发或幂等。业务设备 ACK、在线状态、命令关联均由项目负责。

## SessionManager

多 Broker／身份可 ensure(connectionId, options)，不自动 connect；完整连接配置相同才可合并，不能只按 host。ID 使用非敏感标识，不打印含密码的配置比较键。ensure 会同步校验并抛错，也可异步拒绝，把整个 await 调用放 try/catch，不能只链 .catch。

manager 拥有 Session 的销毁和配置；remove/ensure 做改配，不绕过 manager 调 destroy/replaceConnection/updatePolicy。destroy 等待在途工厂与资源释放，是终态；下次登录新建 manager。采用 communication 时它已拥有 manager，不再外加同资源管理器。
