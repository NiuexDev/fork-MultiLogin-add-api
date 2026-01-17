# MultiLogin API 扩展与改造方案

本方案旨在实现最简的 API 扩展，通过引入“函数式钩子（Hooks）”来让外部插件深入介入 MultiLogin 的核心流程。

## 1. 核心设计理念

不引入复杂的事件总线（EventBus），而是通过在 `MultiLoginAPI` 中开放一组**函数式回调接口（Hooks）**来实现。外部插件只需注册一个 Lambda 表达式或实现一个简单接口，就能“劫持”或“修改”核心流程。

## 2. API 层设计：定义“钩子”

在 `api` 模块中定义一个新的接口 `PluginHooks`，并在 `MultiLoginAPI` 中暴露它。

### 2.1 新增 `PluginHooks` 类

这个类作为所有回调函数的容器。为了最简，我们直接使用 Java 8 的 `Function` 和 `BiFunction`。

```java
package moe.caa.multilogin.api.hook;

import moe.caa.multilogin.api.data.MultiLoginPlayerData;
import moe.caa.multilogin.api.profile.GameProfile;
import moe.caa.multilogin.api.service.IService;

import java.util.function.BiFunction;
import java.util.function.Consumer;
import java.util.function.Function;

/**
 * 插件钩子管理器，允许外部插件介入 MultiLogin 的核心流程。
 */
public class PluginHooks {

    // ---------------------------------------------------------
    // 钩子 1: 预登录检查 (Pre-Login Check)
    // ---------------------------------------------------------
    // 输入: 玩家名, IP
    // 输出: null 表示通过; 非 null 字符串表示拒绝理由(踢出玩家)
    private BiFunction<String, String, String> preLoginHook;

    public void setPreLoginHook(BiFunction<String, String, String> hook) {
        this.preLoginHook = hook;
    }

    public BiFunction<String, String, String> getPreLoginHook() {
        return preLoginHook;
    }

    // ---------------------------------------------------------
    // 钩子 2: 档案转换与绑定 (Profile Transformation / Binding)
    // ---------------------------------------------------------
    // 输入: 原始验证通过的档案(Original Profile), 来源服务(Service)
    // 输出: 最终要进入服务器的档案(Final Profile)
    // 说明: 
    //   - 如果你返回 null，则使用 MultiLogin 默认的数据库逻辑(查库/生成)。
    //   - 如果你返回一个新的 GameProfile，插件将直接使用它作为玩家的身份(改名/换UUID/绑定)。
    private BiFunction<GameProfile, IService, GameProfile> profileTransformHook;

    public void setProfileTransformHook(BiFunction<GameProfile, IService, GameProfile> hook) {
        this.profileTransformHook = hook;
    }

    public BiFunction<GameProfile, IService, GameProfile> getProfileTransformHook() {
        return profileTransformHook;
    }

    // ---------------------------------------------------------
    // 钩子 3: 登录后通知 (Post-Login Notification)
    // ---------------------------------------------------------
    // 输入: 最终生成的玩家数据
    // 用途: 统计、日志、跨服同步通知等
    private Consumer<MultiLoginPlayerData> postLoginHook;

    public void setPostLoginHook(Consumer<MultiLoginPlayerData> hook) {
        this.postLoginHook = hook;
    }

    public Consumer<MultiLoginPlayerData> getPostLoginHook() {
        return postLoginHook;
    }
}
```

### 2.2 修改 `MultiLoginAPI`

在 API 接口中增加获取 Hooks 的方法。

```java
public interface MultiLoginAPI {
    // ... 原有方法 ...

    /**
     * 获取钩子管理器，用于注册自定义逻辑
     */
    @NotNull PluginHooks getPluginHooks();
}
```

## 3. Core 层设计：核心埋点

在代码的关键位置插入检查逻辑：如果外部注册了 Hook，就执行 Hook；否则执行默认逻辑。

### 3.1 埋点：预登录拦截

**位置**: `moe.caa.multilogin.core.auth.AuthHandler` 的 `auth` 方法开头。

```java
// AuthHandler.java

public LoginAuthResult auth(String username, String serverId, String ip) {
    // [Hook 埋点]
    var hook = core.getPluginHooks().getPreLoginHook();
    if (hook != null) {
        String kickReason = hook.apply(username, ip);
        if (kickReason != null) {
            // 如果 Hook 返回了拒绝理由，直接拦截
            return LoginAuthResult.ofDisallowedByPreLogin(kickReason);
        }
    }

    // ... 继续执行原有的 Yggdrasil 验证逻辑 ...
}
```

### 3.2 埋点：档案绑定与改名 (最核心的需求)

**位置**: `moe.caa.multilogin.core.auth.validate.ValidateAuthenticationService` 的 `checkIn` 方法。
或者更深入一点，在 `AssignInGameFlows.java` (负责分配 UUID 的地方) 之前。

为了让外部插件拥有最大权限，建议在 `ValidateAuthenticationService.checkIn` 的最开始进行拦截。

```java
// ValidateAuthenticationService.java

public ValidateAuthenticationResult checkIn(BaseServiceAuthenticationResult baseResult) {
    // [Hook 埋点]
    var hook = core.getPluginHooks().getProfileTransformHook();
    
    if (hook != null) {
        // 调用外部插件的逻辑
        GameProfile original = baseResult.getResponse();
        IService service = baseResult.getServiceConfig();
        
        // 让外部插件决定这个玩家最后变成谁
        GameProfile transformed = hook.apply(original, service);
        
        if (transformed != null) {
            // 外部插件接管了！直接使用它返回的档案，跳过 MultiLogin 默认繁琐的 SQL 检查
            return ValidateAuthenticationResult.ofAllowed(transformed);
        }
    }

    // ... 如果 Hook 返回 null，则继续执行原有的 sequenceFlows (查表、正则、分配UUID) ...
    ValidateContext context = new ValidateContext(baseResult);
    // ...
}
```

**效果**：
外部插件只需要写一行代码：
```java
api.getPluginHooks().setProfileTransformHook((original, service) -> {
    if (original.getName().equals("Notch")) {
        // 把 Notch 强制改名为 "Notch_Fake" 并分配一个固定的 UUID
        return new GameProfile(UUID.fromString("0000-0000..."), "Notch_Fake", ...);
    }
    return null; // 其他人照旧
});
```
这样就完美实现了“改名”、“绑定”和“分配”的完全接管。

### 3.3 埋点：登录后通知

**位置**: `AuthHandler` 的 `checkIn` 方法末尾，当结果为 ALLOWED 时。

```java
// AuthHandler.java

if (validateAuthenticationResult.getReason() == ValidateAuthenticationResult.Reason.ALLOWED) {
    // ... 原有逻辑：写入缓存 ...
    
    // [Hook 埋点]
    var hook = core.getPluginHooks().getPostLoginHook();
    if (hook != null) {
        // 通知外部插件
        hook.accept(playerData);
    }
    
    return LoginAuthResult.ofAllowed(...);
}
```

## 4. 总结

这个方案**不需要**重构整个项目结构，也**不需要**引入复杂的事件监听器类。
你只需要：
1.  加一个 `PluginHooks` 类（纯 POJO）。
2.  在 `MultiLoginAPI` 加一个 Getter。
3.  在 `AuthHandler` 和 `ValidateAuthenticationService` 加几个 `if (hook != null)` 的判断。

这就是实现你需求的**代码量最小、侵入性最低**的方案。
