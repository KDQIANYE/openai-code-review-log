# 代码评审报告

## 变更概述

本次变更仅将 `Message.java` 中微信模板消息的 `template_id` 默认值进行了替换，属于**配置类变更**而非逻辑变更。虽改动极小，但暴露出的设计问题值得重点关注。

---

## 问题清单

### 🔴 P0：配置硬编码，违背配置与代码分离原则

`template_id`、`touser`、`url` 三项全部硬编码在源码中，导致：

- **每次切换模板、更换接收人都要改代码 → 重新构建 → 重新发布**，本次 diff 本身就是该问题的直接体现
- 若 SDK 被多个项目/环境复用，无法按环境差异化配置
- 领域模型（`domain/model` 包）不应承载环境相关的默认值，污染了领域层的纯粹性

**建议方案：**

```java
public class Message {
    private String touser;
    private String templateId;
    private String url;

    // 提供构造器或 Builder，由调用方注入
    public static Builder builder() { ... }
}
```

配合配置文件：

```yaml
wx:
  message:
    to-user: ozn_M3MnNHT0VsTZaXI93_PoHkIY
    template-id: D0Btxh5e4XoFvHw63vDgmrRXL3ep8D3J3cvD4MFQKR8
    url: https://weixin.qq.com
```

### 🟠 P1：敏感信息入库

`touser` 是**特定用户的 OpenID**，属于用户身份标识数据，硬编码提交到代码仓库存在：

- 隐私合规风险（可追溯到具体微信账号）
- 代码仓库一旦开源/泄露，该信息即暴露

个人测试用的 OpenID 应走本地配置或环境变量，而非入库。

### 🟠 P2：切换模板的隐式风险

替换 `template_id` 意味着**消息渲染的变量占位符可能全部变化**：

- 新模板的变量（如 `{{title.DATA}}`、`{{result.DATA}}`）必须与 `data` Map 中填充的 key 严格一致
- 若不匹配，微信不会报错，但用户收到的消息会**字段缺失或显示空白**，属于静默故障

**建议：** PR 描述中应附上新模板的变量定义截图，并附一条实际推送的消息截图作为验证证据。

### 🟡 P3：命名不符合 Java 规范

- `template_id` → `templateId`
- `touser` → `toUser`

若序列化时需与微信 API 字段对齐，使用注解而非妥协命名：

```java
@JsonProperty("template_id")
private String templateId;

@JsonProperty("touser")
private String toUser;
```

### 🟡 P4：`data` 结构可读性差

`Map<String, Map<String, String>>` 双层嵌套 Map 语义不明，建议封装值对象：

```java
private Map<String, TemplateValue> data;

public class TemplateValue {
    private String value;
    private String color; // 可选
}
```

---

## 评审结论

| 项目 | 结论 |
|------|------|
| 变更意图 | ✅ 明确（切换通知模板） |
| 实现方式 | ❌ 不合理（硬编码替换硬编码） |
| 可合并性 | ⚠️ **建议修改后再合并** |

**核心意见：** 这是一次"治标不治本"的修改。同样的代码下个月切换模板时还会产生一模一样的 diff。建议借此机会将模板 ID、接收人等配置外部化，一次性解决 P0/P1 问题；短期内至少应在 PR 中补充模板变量一致性的验证说明（P2），避免线上消息渲染异常。