# MultiLogin 运行原理与流程深度解析

本文档将像调试器（Debugger）一样，按照**代码执行流（Call Stack）**和**类与方法的具体调用顺序**，一步步拆解 MultiLogin 的运行机制。

## 一、 启动初始化流程 (Initialization Process)

这是插件启动时的“偷天换日”过程，核心目标是修改 Velocity 的底层数据包映射表。

**入口：** `MultiLoginVelocity.java` 类中的 `onInitialize` 方法（监听 `ProxyInitializeEvent` 事件）。

1.  **`MultiLoginVelocity.onInitialize(ProxyInitializeEvent event)`**
    *   **第 71 行**: `multiCoreAPI = pluginLoader.getCoreObject();`
        *   **动作**: 通过自定义类加载器反射实例化 `MultiCore` 对象。
    *   **第 72 行**: `multiCoreAPI.load();`
        *   **跳转**: 进入 `MultiCore.java` 的 `load()` 方法。
        *   **动作**: 初始化数据库 (`sqlManager.init()`)、配置文件 (`pluginConfig.reload()`)、语言文件等基础服务。
        *   **关键**: 调用 `setupFloodgate()`，如果有 Floodgate 插件，则实例化 `FloodgateAuthenticationService` 并注册。
    *   **第 73 行**: `injector = ... newInstance();`
        *   **动作**: 反射实例化 `moe.caa.multilogin.velocity.injector.VelocityInjector`。这样做是为了让注入代码与主插件逻辑隔离。
    *   **第 74 行 (核心)**: `injector.inject(multiCoreAPI);`
        *   **跳转**: 进入 `VelocityInjector.java` 的 `inject()` 方法。
        *   **详细步骤**:
            1.  调用 `MultiInitialLoginSessionHandler.init()`:
                *   **动作**: 这里面全都是反射查找 (`Class.forName`, `getDeclaredMethod`)。它预先获取了 Velocity 内部类 `InitialLoginSessionHandler` 的私有字段（如 `login` 包、`verify` 数组）和方法（如 `assertState`）的 `MethodHandle` 句柄，为后续“手术”做准备。
            2.  调用 `getServerboundPacketRegistry(StateRegistry.LOGIN)`:
                *   **动作**: 获取 Velocity `LOGIN` 阶段（登录状态）的“发往服务端”数据包注册表。
            3.  调用 `redirectInput(..., EncryptionResponsePacket.class, ...)`:
                *   **动作**: 这是最关键的一步。它遍历 Velocity 的协议注册表 (`packetIdToSupplier` Map)。
                *   **修改**: 找到所有对应 `EncryptionResponsePacket`（加密响应包）的条目，把它的构造工厂（Supplier）替换成 `MultiEncryptionResponse::new`。
                *   **结果**: 以后 Velocity 只要收到加密响应包，**不再创建原生的包对象，而是创建插件自定义的 `MultiEncryptionResponse` 对象**。
            4.  调用 `redirectInput(..., ServerLoginPacket.class, ...)`:
                *   **动作**: 同上，把 `ServerLoginPacket`（登录开始包）替换为 `MultiServerLogin`。
    *   **第 75 行**: `injector.registerChatSession(...)`
        *   **动作**: 注册 `PlayerSessionPacketBlocker`，用于在高版本拦截和处理聊天会话包，防止因签名问题被踢出。

---

## 二、 Java 版玩家登录流程 (Java Player Login Flow)

当玩家点击“加入服务器”时，数据流如何在代码中穿梭：

**阶段 1：网络包拦截**
1.  **Velocity 内部网络层**:
    *   接收到玩家发来的 `EncryptionResponsePacket`（加密响应包）。
    *   查找注册表（在初始化时已被我们修改）。
    *   **实例化**: 创建了一个 `MultiEncryptionResponse` 对象（而非原生对象）。
    *   **调用**: 调用该对象的 `handle(MinecraftSessionHandler)` 方法。

**阶段 2：接管控制权**
2.  **`MultiEncryptionResponse.handle(handler)`**
    *   **判断**: 检查当前的 `handler` 是否是 `InitialLoginSessionHandler`（Velocity 处理登录握手的标准处理器）。
    *   **动作**: `new MultiInitialLoginSessionHandler(handler, multiCoreAPI)`。
        *   创建一个“影子处理器”，把原生的 handler 包装起来。
    *   **调用**: `multiInitialLoginSessionHandler.handle(this)`。**此时逻辑正式进入插件核心。**

**阶段 3：影子处理器执行 (`MultiInitialLoginSessionHandler.java`)**
3.  **`handle(EncryptionResponsePacket packet)`**
    *   **第 149-150 行 (模拟状态)**:
        *   使用反射调用原生 handler 的 `assertState` 和 `setCurrentState`。告诉 Velocity：“别慌，流程走到这一步是正常的”，防止 Velocity 报错。
    *   **第 163-176 行 (解密)**:
        *   调用 `EncryptionUtils.decryptRsa(...)`。使用服务器私钥解密玩家发来的 `SharedSecret`（共享密钥）。
    *   **第 184 行 (异步验证)**:
        *   `multiCoreAPI.getAuthHandler().auth(username, serverId, ip)`。
        *   **跳转**: 进入 `AuthHandler.java`。

**阶段 4：多路验证逻辑 (`AuthHandler.java` & `YggdrasilAuthenticationService.java`)**
4.  **`AuthHandler.auth(...)`**
    *   **调用**: `yggdrasilAuthenticationService.hasJoined(...)`。
5.  **`YggdrasilAuthenticationService.hasJoined(...)`**
    *   **逻辑**:
        *   查询数据库 `core.getSqlManager()...`：这个玩家上次是用哪个验证服登录的？
        *   **排序**: 将上次登录的验证服放入 `primaries` 集合（优先验证），其他的放入 `secondaries`。
    *   **调用**: `EntrustFlows.run(...)`。
        *   这是一个任务流引擎，它会并发或顺序地向 Mojang、LittleSkin 等服务器发送 HTTP GET 请求（`hasJoined` 接口）。
    *   **返回**: 只要有一个服务器返回“验证通过”，就返回 `ALLOWED` 结果。

**阶段 5：收尾与放行 (`MultiInitialLoginSessionHandler.java`)**
6.  **回到 `handle` 方法的回调中 (第 198 行起)**:
    *   **成功**: 如果验证结果是 `AuthResult.Result.ALLOW`。
    *   **皮肤修复**: 调用 `skinRestorerHandler.doRestorer(result)`，获取该玩家在验证服上的皮肤数据。
    *   **关键动作 (第 219 行)**: `mcConnection.setActiveSessionHandler(...)`。
        *   **偷梁换柱**: 插件通过反射，构造一个新的、原生的 `AuthSessionHandler`（Velocity 用于处理加密后逻辑的处理器）。
        *   **参数**: 将我们自己验证通过生成的 `GameProfile`（包含正确的 UUID 和皮肤）传进去。
        *   **设置**: 将这个原生处理器设置给连接。
    *   **结果**: 从此刻起，MultiLogin 退出舞台，Velocity 认为这是一个刚刚通过正版验证的连接，继续处理后续的压缩和登录成功包。

---

## 三、 基岩版/Floodgate 玩家登录流程 (Floodgate Flow)

基岩版玩家不走加密包流程，而是走 Floodgate 的握手扩展流程。

**阶段 1：握手回调**
1.  **Floodgate 插件**:
    *   当基岩版玩家连接时，Floodgate 插件处理握手包。
    *   它会遍历所有注册的 `HandshakeHandler`。
2.  **`FloodgateAuthenticationService.handle(HandshakeData data)`**
    *   **位置**: `core/src/main/java/.../floodgate/FloodgateAuthenticationService.java`。
    *   **动作**: MultiLogin 在这里被调用。

**阶段 2：数据处理与登记**
3.  **`handle` 方法内部**:
    *   **第 68-70 行**: 从 `data` 中提取 `xuid` 和 `username`。
    *   **构造**: 创建一个临时的 `GameProfile`。
    *   **第 77 行**: `multiCore.getAuthHandler().checkIn(result)`。
        *   **跳转**: 进入 `AuthHandler.java` -> `ValidateAuthenticationService.java`。
        *   **目的**: 这里不发 HTTP 请求（因为信任 Floodgate），但会进行内部检查（如 IP 限制、黑白名单）并更新数据库中的玩家最后登录时间。

**阶段 3：回填数据**
4.  **回到 `handle` 方法 (第 78 行起)**:
    *   **判断**: 如果 `checkIn` 返回允许。
    *   **第 80 行**: `handshakeData.setLinkedPlayer(...)`。
        *   **动作**: 将 MultiLogin 生成的最终 `GameProfile`（可能经过了 UUID 映射或改名）回填给 Floodgate 的数据对象。
    *   **结果**: Floodgate 插件随后会读取这个 `LinkedPlayer` 信息，告诉 Velocity 以这个身份让玩家进服。Velocity 随后跳过加密验证阶段（因为是离线模式逻辑），直接让玩家进入服务器。
