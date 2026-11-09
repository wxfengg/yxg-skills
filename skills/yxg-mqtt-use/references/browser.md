# 浏览器／H5／WebView

确认代码执行在真实浏览器客户端，存在原生 WebSocket、URL 和 ES 模块打包能力。SSR 的服务端加载不应创建客户端；在客户端挂载／登录完成后创建，在资源拥有者退出时清理。库构建所用 Vite 7 不要求宿主跟随升级 Vite；用现有 Vite、Webpack 或其它工具验证实际消费。

```ts
import { createBrowserClient, type BrowserOptions } from '@yxg/mqtt/browser';
export const createForBrowser = (options: BrowserOptions) => createBrowserClient(options);
```

工厂闭包必须拿到 api-contract.md 的完整配置和 transport 后再调用。HTTP 登录已提供的身份来源优先，不能将前端 env 当作安全保存密码的机制。

- 端点使用完整 ws:// 或 wss://；HTTPS 页面使用 WSS。URL 内不得夹带 username/password 或 fragment，认证放 connection 字段。
- 真实服务核对路径、端口、证书、Origin 和子协议 mqtt；CONNECT/SUBACK/PUBACK 与设备业务回包分别留证。
- 收消息先注册监听再连接／订阅，防止漏掉保留消息。发布前等待 subscribe.ready，并按需 requireSubscriptions。
- Vue/React 页面不各自创建共享业务连接；在项目已有服务／Store 生命周期持有。页面卸载只释放它自己的 listener，登录退出才销毁登录资源。开发热更新要清掉旧实例。
- 异步创建／登录初始化期间，宿主保存最近一次网络提示，在取得新 Session 后、connect 前调用 setNetworkAvailable 应用最新状态，避免漏掉初始化期间的 offline 事件；前后台期望状态也应在激活前复核。
- navigator.onLine 等只是网络提示；转发 setNetworkAvailable 不等于证明 Broker 在线。tab 隐藏时不要无依据销毁：保留项目已定后台策略，浏览器冻结会推迟 timer。
- 不注入 wx，不复制小程序 loader 插件到网页。

验收：使用真实 tgz 的现有项目生产构建；客户端导入、SUBACK、已授权安全回环、掉线后的新消息、卸载／登录切换后无残留。可复用库的 Chrome+本地协议验收作参考，但仍核对目标页面 CSP／WSS／Origin。没有后端配置时完成构建和可注入代码，明确实网待验，不伪造成功连接。
