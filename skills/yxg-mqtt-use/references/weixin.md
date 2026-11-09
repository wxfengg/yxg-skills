# 微信小程序／uni-app MP-WEIXIN

先识别原生小程序或 uni-app、开发／发行构建与实际基础库，不把一个宿主的插件路径复制到另一个宿主。

```ts
import { createWeixinClient, type WeixinOptions } from '@yxg/mqtt/weixin';
export function createWeixinSessionFactory(platform: WeixinOptions['platform']) {
  return (options: Omit<WeixinOptions, 'platform'>) =>
    createWeixinClient({ ...options, platform });
}
```

实际传入 `platform: { connectSocket: input => wx.connectSocket(input) }`，它必须同步返回独占 SocketTask，不能返回 Promise。TypeScript 若无 wx 类型，沿用现有平台类型或把 platform 参数类型声明为 WeixinOptions['platform']，不要为方便加入 any 全局或新的类型依赖。每次物理连接持有自己的 SocketTask，不用 wx.sendSocketMessage 等全局 socket API 替代。

## 不可破坏的加载顺序

公开入口导入 → 工厂验证／快照 → 初始化缺失的 AbortController/AbortSignal、navigator、queueMicrotask → 动态加载 MQTT → 显式 connect。MQTT 的依赖可能在模块求值时捕获全局；业务或其它模块提前 import mqtt 会破坏顺序。

0.1.0 tgz 保留两个外部桥模块：

- `src/weixin/index.js`：loadWeixinMqtt 动态 import `./load-mqtt.js`。
- `src/weixin/load-mqtt.js`：实际 import mqtt。

检查目标消费产物是否把 MQTT 提前塞进入口或 common/vendor 并在工厂前求值，不能仅看库源码／Node 验收。对已存在的 uni-app 钩子先验证其实际 module ID（pnpm／vendor／符号链接路径可能不同）。只有输出证明需要时，才最小调整分块／延迟 require；不要全局改动态 import 或模板式加入插件。

历史某 uni-app 宿主还需要在 MQTT 模块局部绑定 `const navigator = globalThis.navigator`，因为裸 navigator 可能与 globalThis.navigator 不一致。这不是所有项目的必需补丁；仅在目标输出／运行证据命中时处理，不能全局伪造浏览器身份。保留已有有效适配，并验证发行输出。

## 生命周期与验证

- onShow/onHide、网络回调沿宿主现有 App 级入口接线；只注册一次，退出移除旧监听。
- 自管 Session：后台使用 disconnect 或项目已确定策略；通信协调器：dispose 要在前台重新 startSession，pause-http 仅暂停 HTTP，不能混用。
- startSession 会设置 active=true，防止后台到达的旧登录／列表请求重新激活通信。
- WSS/TLS、socket 合法域名、独立 HTTP request 域名、clientId/ACL 由真实服务核验。库识别的 scheme 不代表所有微信环境支持它。
- 环境支持最低版本没有完整矩阵，不臆造“所有 iOS/Android/鸿蒙均可用”。用户历史真机成功不替代当前版本＋当前宿主回归。

至少检查发行产物加载时序、首次 SUBACK、安全收发、物理断网恢复新消息、后台／锁屏恢复、退出无重连。开发者工具、模拟 SocketTask 和真实手机分别记录；不可执行的手机动作列成具体短步骤。
