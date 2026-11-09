---
name: yxg-mqtt-use
description: 将 @yxg/mqtt 接入或升级到微信小程序、uni-app App、原生桥接 App、浏览器或 WebView 项目，按现有架构完成平台适配、登录与网络生命周期、订阅收发及验证。用于明确使用 @yxg/mqtt 的集成、迁移和接入排障；不用于泛指 MQTT 服务端或声网 RTC/RTM 接入。
---

# @yxg/mqtt 集成

将用户指定的库版本接入目标项目，交付可构建、可验证的实际代码与接线。分析请求只做分析；接入请求直接实施已授权的最小改动，不止给方案。保留已有业务接口和用户工作区改动，不因接入而重写整个通信层。

本技能按 `@yxg/mqtt@0.1.0` 编写。以后使用时，以目标实际包的 exports、声明和实现为准；发现差异就调整接线并指出版本差别，不把这里的快照当作永远最新。

## 先定位项目和制品

- 读取目标 AGENTS.md、依赖清单／锁文件、框架与构建入口。追踪现有 MQTT/HTTP 入口、登录退出、设备路由、页面订阅、网络与前后台的真实调用链；只读实际相关文件，避免全仓倾倒和暴露凭据。
- 从已安装版本、项目 vendor 归档、用户提供路径或维护者源码确认制品；不要假设公共 npm registry 存在。默认交付是 private tgz。可从有权限访问的 `wxfengg/yxg-mqtt` GitHub Release 获取 tgz，或使用维护者提供的归档；私有仓库需当前用户自己的访问权限。缺少制品时先检查当前环境，再询问获取位置，不猜测本机路径。
- 核对 tgz 的 package.json、五个 exports 目标、微信桥文件及 SHA-256（有可信交付记录时与其比对）；不要仅看同名文件／版本号。复制到目标项目既有 vendor 目录后用原包管理器安装本地 tgz，并验证锁文件实际指向此文件。用户授权接入包含安装指定库时执行；额外框架／原生插件／工具链升级需另有范围依据，不借机升级。
- 核对依赖库自己解析的 mqtt 是否为声明版本。目标可能另有旧 mqtt，不粗暴升级或删除它。宿主不需要为了接入再直接安装 mqtt，也不应升级宿主 Vite 来“对齐库 Vite”。
- 凭据取自项目现有登录／配置提供者，不能复制 Demo App.vue、输出完整连接对象、把 token 写入文档或技能。前端构建 env 不是保密保险箱；沿用项目认证体系。

## 选择接入路径

先读 [公共合同与最小接线](references/api-contract.md)，再只读目标平台的参考：

| 目标 | 路径与必读参考 |
| --- | --- |
| 原生微信小程序、uni-app MP-WEIXIN | `@yxg/mqtt/weixin` → [微信](references/weixin.md) |
| 浏览器、Vue/React/H5 | `@yxg/mqtt/browser` → [网页](references/browser.md) |
| uni-app APP、原生 Android/iOS、嵌入式 WebView | 先辨认真实 JS 运行环境与桥能力 → [App](references/app.md) |
| 已有可验证的独占 MQTT 引擎 | 根入口 createSession 或 /native；读 [App 的桥合同](references/app.md) |

根据业务复杂度选择一条管理路径，不叠加两个所有者：

- 一个连接／已有稳定 facade：平台工厂 + MqttSession；多连接需要时使用 SessionManager。
- 需要统一设备路由或 MQTT/HTTP 调度：可选 `/communication`，再读 [设备协调器](references/communication.md)。没有此需求就不强制引入。
- 不支持的平台或缺少原生桥：继续完成能独立验证的配置、路由和生命周期工作，明确缺口；不能用空桥、假的 ACK 或“所有 App 都有 WebSocket”冒充完成。

## 落实接线

1. 保留项目服务／handler／Store 结构，把平台创建封装放在现有通信边界。配置从实际接口或已批准策略映射，不编造 Topic、clientId、HTTP 接口、QoS、retain、ACL、设备 ACK。
2. 同一批资源只有一个连接、重连、订阅和待发队列所有者。旧链路与新链路切换需要明确边界，避免同 clientId 双连接；不要把已提交／结果未知的旧命令搬到新队列或 HTTP。
3. 先接登录代次、退出清理和初始化竞态，再接订阅／消息／发送；页面卸载只释放页面拥有的监听／句柄，不随意销毁全局共享连接。
4. 原接口若同步返回 boolean，不直接替换成 Promise/Ticket 后宣称兼容；在已有 facade 明确转换。bool accepted 仅为本地接纳，业务成功仍由项目协议决定。
5. 已知通道可用 MqttSendOptions/HttpSendOptions；动态设备 ID 需运行时通道判断。真实错误和 evidence 原样按合同处理，不吞错返回成功。
6. 沿 [分层验收](references/validation.md) 完成可自动执行的类型、构建、契约测试；只有必要的手机／云服务动作才交给用户。不要主动发送复位、解绑、控制设备等副作用指令。

## 交付标准

交付实际接入文件、入口调用位置、依赖／锁变化、采用的版本和指纹、已执行结果以及尚需外部动作。记录微信延迟加载检查、登录退出／后台策略和 unknown 处理方式。运行时、模拟、本地协议对端、真实 Broker、真机、用户报告的证据分别写清；“版本为 0.1.0”不表示未测平台已验证。不自行提交、发布包或部署业务。

已有使用文档优先更新；项目记录写清兼容影响，特别是 onDelivery 与 ticket.done 的重复提示风险。无需为每次小接入生成新框架或大量独立报告。
