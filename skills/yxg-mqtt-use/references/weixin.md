# 微信小程序／uni-app MP-WEIXIN

先识别原生小程序或 uni-app、开发／发行构建、实际基础库和库制品。公共生命周期与订阅规则见[公共合同](api-contract.md)，不要把一个宿主的构建钩子复制到另一个项目。

```ts
import { createWeixinClient, type WeixinOptions } from '@yxg/mqtt/weixin';
export function createWeixinSessionFactory(platform: WeixinOptions['platform']) {
  return (options: Omit<WeixinOptions, 'platform'>) =>
    createWeixinClient({ ...options, platform });
}
```

实际传入 `platform: { connectSocket: input => wx.connectSocket(input) }`，必须同步返回独占 SocketTask，不能返回 Promise。沿用项目已有 wx 类型或 WeixinOptions['platform']，不用 any 全局或新依赖掩盖类型问题。每次物理连接持有自己的 SocketTask，不用 wx.sendSocketMessage 等全局 API 替代。

## 环境初始化、延迟求值与分包

所需顺序是：公开入口导入 → 工厂校验／快照 → 初始化缺失的 AbortController/AbortSignal、navigator、queueMicrotask → 加载并求值 MQTT 引擎 → 创建 Session → 显式 connect。依赖可能在模块求值时捕获全局；其它模块提前 import mqtt 会破坏顺序，异常也可能发生在工厂接管之前。

1. 沿实际安装包追踪公开入口、异步桥和引擎的引用链。`0.1.0` 旧归档可见 `src/weixin/load-mqtt.js → mqtt`，较新构建可见 `→ dist/weixin/mqtt-engine.js`；这是辨认线索，不能据此猜测当前包内容或绕过 exports 编写业务 import。
2. 检查宿主开发和发行输出的真实 module ID、依赖图和求值入口。uni-app 可能合并动态 import，把 MQTT 放进 common/vendor 并提前求值；pnpm、vendor、符号链接或库引擎调整也会使原匹配规则失效。
3. 分开验收“环境初始化后才求值”和“实际移出微信主包”。延迟执行本身不等于分包；需要分包时检查注册、输出位置、下载后执行和失败传播。已有异步分包可使用项目获准的 require.async 等机制，核对基础库能力，不固定分包名或第三方插件。
4. 仅当目标输出证明有问题时，在已有构建边界最小调整，验证真实消费产物；不全局替换动态 import，不手改 node_modules 或生成物。主包限制以微信最终预览／上传统计为准（含插件影响），磁盘总和与本地依赖分析不能替代；外部操作仍遵守任务授权。

## 兼容适配与退役

遇到裸 navigator 与 globalThis.navigator 不一致，先确认目标引擎实际读取方式和产物局部绑定；已有库构建若已绑定，不在宿主重复注入，更不全局伪造浏览器身份。

遇到 `length must be a positive 32-bit integer` 一类初始化异常，错误文本本身不足以确定根因。已确认的一种情形是微信将标准方法包装为 getter，get-intrinsic 将 getter 当成方法，不能概括为函数 length 缺失。确认需目标堆栈／描述符证据或对应完整引擎复现；只改空壳 mock 不能证明修复。

若仍需兼容修补，限定已核验依赖产物、唯一锚点与有限方法集合，保留 Map.size 等真实 accessor；锚点或依赖变化即停止套用并重新核验，不全局改内建对象。部分新制品已在微信专用引擎中包含该修补及局部 navigator：先检查实际引擎，确认覆盖后撤除宿主重复补丁与不再需要的临时诊断，回归普通环境、getter 包装和真实 accessor，再记录目标真机结果。旧制品仍依赖的适配不能无条件删除，单次用户成功报告也不代表所有机型兼容。

## 加载失败与取消

- 引擎加载发生在 Session 创建前，首次模块下载失败不会触发连接重试。区分环境初始化、下载／求值和连接失败，不能只显示“环境不支持”而丢失阶段与原因。
- 核对制品是否支持 `engineLoad.timeoutMs/signal` 以及 SessionManager 工厂第二参数中的 signal；两端都支持才透传。缺少时沿用现有受控加载边界并明确限制，不照搬新签名。未配置加载期限的制品可能一直等待，不能声称天然有超时或自动重试。
- 加载等待受登录归属、前后台意图、截止时间和取消约束；重试预算由宿主已批准策略决定，不固定秒数／次数。逻辑超时或取消不等于物理取消 import／微信下载，仍处理迟到成功与拒绝，阻止旧代次创建或激活 Session。
- 保留 stage/reason 和供受控调查的 cause；不可枚举 cause 也不是脱敏数据，不自动打印完整 error、stack、errMsg、mod 或连接对象。只输出本次诊断必要且已脱敏的信息。

## 前后台与网络恢复

- 沿现有 App 级入口接 onShow/onHide 与网络监听，按拥有者清理，避免反复注册。后台可能漏掉网络事件，回前台应补采样；用请求／事件代次防止旧快照覆盖新事件。hide、退出和新登录撤销旧采样，后台迟到结果不能恢复通信。
- 查询网络失败不证明离线或在线；项目若允许，可尝试受归属保护的连接，结果仍以协议证据为准。应用初始化中的网络提示应在取得 Session 后、connect 前更新。
- 自管 Session 沿用后台策略，采用 disconnect 时处理快速 hide/show 的关闭屏障。采用协调器则按[协调器后台合同](communication.md)恢复，不混用 dispose 与 pause-http，也不让迟到 startSession 重新激活后台通信。
- 恢复顺序是加载／连接、当前订阅确认、仍有意义的业务补查与新回包；库不负责 Vuex、播放查询或固定业务轮询周期。

WSS/TLS、socket 合法域名、独立 HTTP request 域名、clientId/ACL 由实际服务核验。scheme 被识别不等于所有微信环境可用；没有完整最低版本／机型矩阵时不臆造支持范围。按[分层验收](validation.md)检查产物和故障场景，开发者工具、模拟 SocketTask 与真机证据分别记录。
