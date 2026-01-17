# MultiLogin API 使用指南

## 1. 核心概念

要使用 MultiLogin 的 API，核心在于通过 `MultiLoginAPIProvider` 获取 API 实例，然后利用该实例查询玩家数据或服务信息。

*   **`MultiLoginAPI`**: 核心接口，提供所有查询功能。
*   **`MultiLoginAPIProvider`**: 静态工厂类，用于获取 `MultiLoginAPI` 的单例对象。
*   **`MultiLoginPlayerData`**: 玩家数据对象，包含玩家的原始游戏档案（GameProfile）和使用的验证服务（IService）。

## 2. 完整代码实例

假设你正在编写一个 **Velocity 插件**，你想在玩家登录时获取他使用的是哪个验证服务器（是正版还是某个外置登录）。

```java
import com.velocitypowered.api.event.Subscribe;
import com.velocitypowered.api.event.connection.PostLoginEvent;
import com.velocitypowered.api.proxy.Player;
import moe.caa.multilogin.api.MultiLoginAPI;
import moe.caa.multilogin.api.MultiLoginAPIProvider;
import moe.caa.multilogin.api.data.MultiLoginPlayerData;
import moe.caa.multilogin.api.profile.GameProfile;
import moe.caa.multilogin.api.service.IService;
import moe.caa.multilogin.api.service.ServiceType;

import java.util.UUID;

public class MyPluginListener {

    @Subscribe
    public void onPlayerLogin(PostLoginEvent event) {
        Player player = event.getPlayer();
        UUID playerUUID = player.getUniqueId();

        // 1. 获取 API 实例
        // 注意：建议在插件启动时检查 API 是否可用，或者进行判空，防止 MultiLogin 未加载
        MultiLoginAPI api = MultiLoginAPIProvider.getApi();
        
        if (api == null) {
            System.out.println("MultiLogin 尚未加载！");
            return;
        }

        // 2. 获取玩家的 MultiLogin 数据
        // getPlayerData 可能会返回 null（如果该玩家不是通过 MultiLogin 流程处理的，或者是异常情况）
        MultiLoginPlayerData playerData = api.getPlayerData(playerUUID);

        if (playerData != null) {
            // 3. 获取玩家使用的验证服务信息
            IService loginService = playerData.getLoginService();
            
            // 获取服务名称（例如 "Official", "MyServer", "LittleSkin" 等）
            String serviceName = loginService.getServiceName();
            // 获取服务 ID（配置文件中定义的 ID）
            int serviceId = loginService.getServiceId();
            // 获取服务类型（OFFICIAL, CUSTOM_YGGDRASIL, FLOODGATE 等）
            ServiceType serviceType = loginService.getServiceType();

            // 4. 获取玩家的原始游戏档案 (GameProfile)
            // 这包含玩家在验证服务器上的原始 UUID 和名称
            GameProfile originalProfile = playerData.getOnlineProfile();
            String originalName = originalProfile.getName();
            UUID originalUUID = originalProfile.getId();

            // --- 打印示例输出 ---
            System.out.println("玩家 " + player.getUsername() + " 已登录。");
            System.out.println("  - 来源服务: " + serviceName + " (ID: " + serviceId + ")");
            System.out.println("  - 服务类型: " + serviceType);
            System.out.println("  - 原始名称: " + originalName);
            System.out.println("  - 原始 UUID: " + originalUUID);
            
            // 实例：如果是 Floodgate (基岩版) 玩家，做特殊处理
            if (serviceType == ServiceType.FLOODGATE) {
                System.out.println("  -> 这是一个基岩版玩家！");
            }
        } else {
            System.out.println("无法获取玩家 " + player.getUsername() + " 的 MultiLogin 数据。");
        }
    }
}
```

## 3. 其他常用操作

### 获取所有已配置的验证服务
如果你需要列出服务器当前支持的所有验证方式：

```java
// 获取所有注册的服务列表
for (IService service : api.getServices()) {
    System.out.println("已加载服务: " + service.getServiceName() 
        + " (ID: " + service.getServiceId() + ", Type: " + service.getServiceType() + ")");
}
```

## 4. 项目依赖配置 (Gradle)

你需要将 `multilogin-api` 添加到你的构建脚本中。由于该项目使用了 Shadow 插件，通常建议使用 `compileOnly` 引入 API，因为运行时 MultiLogin 插件会提供具体的实现类。

**build.gradle 示例:**

```groovy
repositories {
    mavenLocal() // 如果你已经在本地构建并发布了 api
    // 或者添加私有仓库地址
}

dependencies {
    // 引入 API 模块，版本号请参考实际构建版本
    compileOnly 'moe.caa:multilogin-api:x.y.z' 
}
```

## 5. 总结

1.  **入口点**：`MultiLoginAPIProvider.getApi()`。
2.  **关键方法**：`api.getPlayerData(uuid)` 是最常用的方法，用于连接 Velocity 的玩家对象和 MultiLogin 的认证数据。
3.  **数据结构**：
    *   `IService`: 告诉你玩家是从哪里来的（正版、皮肤站A、皮肤站B）。
    *   `GameProfile`: 告诉你玩家在那个来源里的原始信息。
