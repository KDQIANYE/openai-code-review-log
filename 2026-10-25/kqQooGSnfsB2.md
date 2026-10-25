# 代码评审报告

## 总体结论：❌ **不建议合并（Request Changes）**

本次变更为代码评审流程增加了微信模板消息通知能力，功能主流程可用，但存在**一个必须立即处理的凭证泄露问题**，以及多处架构与健壮性缺陷。以下按严重程度分级说明。

---

## 🔴 阻塞级问题（P0，必须修复）

### 1. 微信 AppSecret 硬编码且已泄露到 Git 历史

```java
private static final String APPID = "wx11b81060efd869e5";
private static final String SECRET = "b4a3e6126f2f4fa4b7b808cd9f57bc1f";
```

**风险**：任何能读到仓库的人（包括离职员工、仓库被 fork/泄露的场景）都能获取该公众号的接口凭证，可伪造消息、耗尽 API 配额。且该 Secret 已进入 Git 历史，**即使本次删掉，历史提交中依然可见**。

**修复要求**：
1. **立即到微信公众平台重置该 AppSecret**（这不只是代码问题，是安全事件）；
2. 改为环境变量 / CI Secrets 注入：

```java
private static final String APPID  = System.getenv("WX_APPID");
private static final String SECRET = System.getenv("WX_SECRET");
// 或在 SDK 构造函数中传入，由调用方决定配置来源
```

3. 建议在 CI 中接入 secret 扫描（如 gitleaks）防止再次发生。

### 2. `getAccessToken()` 可能返回 null，下游会产生无效请求

```java
String accessToken = WXAccessTokenUtils.getAccessToken(); // 可能返回 null
String url = String.format("...?access_token=%s", accessToken); // access_token=null
```

失败时 URL 会变成 `access_token=null`，微信返回一个语焉不详的错误，排障成本高。**失败应快速失败（fail-fast）**，要么抛出明确异常中断流程，要么在拼接前做防御性校验。

---

## 🟠 严重问题（P1，强烈建议本次修复）

### 3. 代码重复，且已经在同一次提交中发生"漂移"

- `sendPostRequest` 在 `OpenAiCodeReview` 和 `ApiTest` 中各复制了一份；
- `Message` 类复制了一份到 `ApiTest` 内部，且**两份已经不一致**：

| 字段 | `domain.model.Message` | `ApiTest.Message`（内部类） |
|---|---|---|
| template_id | `GLlAM-Q4jd...` | `D0Btxh5e4X...` |
| 默认 url | `https://weixin.qq.com` | 一个硬编码的 GitHub 日志链接 |
| `put` 可见性 | public（推断） | **private** |

同一个类在同一 commit 里就有两个版本，这是重复代码危险性的最直接证据。测试应直接复用 `domain.model.Message`。

### 4. 架构：`OpenAiCodeReview` 职责过重，违反 SRP

主类同时承担了：Git 操作、AI 评审、日志写入、微信 token 获取、HTTP 通信、消息组装。建议引入通知抽象，使主流程只依赖接口，未来接入钉钉/飞书/邮件时无需改动主类（开闭原则）：

```java
public interface ReviewNotifier {
    void notify(String project, String reviewLogUrl);
}

public class WxTemplateNotifier implements ReviewNotifier { ... }

// 主流程
notifier.notify(projectName, logUrl);
```

### 5. 异常被静默吞掉

```java
} catch (Exception e) {
    e.printStackTrace();  // 调用方无法感知失败，CI 日志难以检索
}
```

通知失败可以不阻断评审流程（这是合理的降级策略），但必须：
- 使用 SLF4J 记录 `warn` 级别日志，而不是 `printStackTrace`；
- 失败信息中带上上下文（哪个项目、哪个 logUrl、微信返回的 errcode）。

### 6. 未校验 HTTP 状态码与微信业务错误码

`sendPostRequest` 中：如果微信返回 4xx/5xx，`conn.getInputStream()` 会直接抛 `IOException`；更关键的是，**微信接口即使 HTTP 200，业务失败也会在 body 中返回 `errcode`/`errmsg`**（如 token 过期 `40001`、openid 无效 `40003`）。当前代码完全不检查，失败会被当成成功。

### 7. Token 无缓存，`expires_in` 被解析但从未使用

每次代码评审都请求一次新 token。微信 `getAccessToken` 有**每日调用次数限制**，CI 高频触发（多人并行提交时）会撞限额，导致所有人收不到通知。应实现带过期的缓存：

```java
private static final long SAFETY_MARGIN_MS = 300_000L; // 预留5分钟余量
private volatile CachedToken cached;

public synchronized String getAccessToken() {
    if (cached != null && System.currentTimeMillis() < cached.expireAt) {
        return cached.value;
    }
    // ... 请求新 token，用 expires_in 计算 expireAt
}
```

### 8. 资源与连接管理缺陷

- `HttpURLConnection` 从未调用 `disconnect()`；
- `WXAccessTokenUtils` 中 `BufferedReader` 在读取中途抛异常时不会关闭（未用 try-with-resources）；
- **两个 HTTP 调用均未设置 connect/read 超时**——微信接口一旦网络抖动，整个 CI 评审任务会无限期挂起。

### 9. 测试代码打真实生产接口

`test_wx` 会真实调用微信 API 并给真实用户发消息。单元测试跑一次就发一条消息，且无任何断言。应 mock HTTP 层，或至少标注为 `@Tag("integration")` 从常规 CI 中隔离。

---

## 🟡 一般问题（P2）

10. **硬编码业务配置**：`touser`（openID）、`template_id`、`project = "big-market"`、微信 API URL 全部写死。尤其 `touser` 的本次改动本身就是通过改代码换接收人——这证明它就该是配置项。项目名应从 Git 仓库信息推导或由调用方传入。

11. **`System.out.println(accessToken)` 把凭证打进日志**——即使 Secret 以后走配置，这个 println 也会把 token 泄露到 CI 日志。同理 `System.out.println("Response: ...")` 也是。必须删除。

12. **命名不符合 Java 规范**：`getAccess_token` / `getTemplate_id` 违反驼峰约定。应写成 `getAccessToken` 并用 `@JSONField(name = "access_token")` 处理 JSON 映射；包名 `cn.KD.sdk` 含大写也不符合惯例（存量问题，可后续处理）。

13. **双括号初始化反模式**：

```java
new HashMap<String, String>() {{ put("value", value); }}
```
每次创建匿名内部类并持有外部类引用，建议 `Map.of("value", value)` 或 `Collections.singletonMap`。

14. **`Scanner.useDelimiter("\\A")` 读取响应**：响应体为空时 `next()` 会抛 `NoSuchElementException`。若坚持用 JDK 原生，Java 11 的 `java.net.http.HttpClient` + `BodyHandlers.ofString()` 能一次性解决字符集、超时、状态码三个问题，强烈建议替换 `HttpURLConnection`。

15. **字符集不一致**：`WXAccessTokenUtils` 中 `new InputStreamReader(...)` 未指定 UTF-8（平台相关），而 `sendPostRequest` 指定了。

16. 测试 Message 中的 URL 日期为 `2026-10-22`（未来时间），疑似随手粘贴的占位数据，进一步说明该类不应存在。

---

## ✅ 值得肯定的地方

- `sendPostRequest` 中 `OutputStream` 使用了 try-with-resources；
- Token 响应解析为类型化对象而非手工截取字符串；
- 功能分层意识初现（`domain.model` / `types.utils` 的目录结构），只是本次改动没有遵守这个分层。

---

## 重构示例（核心部分）

```java
public class WxTemplateNotifier implements ReviewNotifier {

    private static final HttpClient HTTP = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5)).build();

    @Override
    public void notify(String project, String logUrl) {
        try {
            String token = WxAccessTokenClient.getAccessToken(); // 带缓存，失败抛异常
            WxTemplateMessage msg = WxTemplateMessage.of(
                    WxConfig.toUser(), WxConfig.templateId(), logUrl, project);

            HttpRequest req = HttpRequest.newBuilder(URI.create(
                        "https://api.weixin.qq.com/cgi-bin/message/template/send?access_token=" + token))
                    .timeout(Duration.ofSeconds(10))
                    .header("Content-Type", "application/json; charset=utf-8")
                    .POST(HttpRequest.BodyPublishers.ofString(JSON.toJSONString(msg), StandardCharsets.UTF_8))
                    .build();

            HttpResponse<String> resp = HTTP.send(req, HttpResponse.BodyHandlers.ofString());
            WxResponse wx = JSON.parseObject(resp.body(), WxResponse.class);
            if (wx.getErrcode() != null && wx.getErrcode() != 0) {
                log.warn("微信通知发送失败: errcode={}, errmsg={}, logUrl={}",
                        wx.getErrcode(), wx.getErrmsg(), logUrl);
            }
        } catch (Exception e) {
            // 通知失败不阻断评审主流程，但必须留下可排查的日志
            log.warn("微信通知异常, logUrl={}", logUrl, e);
        }
    }
}
```

---

## 行动清单（按优先级）

| 优先级 | 行动 |
|---|---|
| **P0（立即）** | 重置微信 AppSecret；凭证改为环境变量注入；删除打印 accessToken 的日志 |
| **P1（本次）** | 修复 null token 问题；删除 `ApiTest` 中重复的 `Message`/`sendPostRequest`；补 HTTP 状态码与 errcode 校验；增加超时；token 加缓存 |
| **P2（短期）** | 抽象 `ReviewNotifier` 接口；`touser`/`template_id`/项目名配置化；引入 SLF4J；规范命名 |
| **P3（规划）** | 替换为 `java.net.http.HttpClient`；测试与生产 API 隔离 |

**一句话总结**：功能能跑，但这次提交把企业凭证写进了源码，这一条就足以否决合并。请先处理凭证泄露（重置 + 外置配置），再按上述清单逐步收紧架构与健壮性。