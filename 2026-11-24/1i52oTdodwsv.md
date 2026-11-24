# 代码评审报告

## 总体评价

本次变更为 SDK 增加了「评审结果推送微信模板消息」的能力，功能方向没问题，但实现上存在**敏感信息泄露、测试数据残留、异常吞噬、大段重复代码**等阻塞性问题。**结论：Request Changes，不建议直接合并。**

---

## 一、🔴 阻塞性问题（必须修复）

### 1. 微信密钥硬编码提交到仓库（安全红线）

```java
private static final String APPID = "wxa7c2d2b6f78fe71c";
private static final String SECRET = "3e8cb344807bb6a94e66092b0a72b364";
```

- 密钥一旦进入 Git 历史，即使后续删除也可追溯。**本次 diff 之前的一组旧密钥（`wx5a228ff69e28a91f`）同样已在历史中，新旧两组 SECRET 都应立即在微信公众平台作废并轮换。**
- 正确做法是外置配置，SDK 使用方注入：

```java
private static final String APPID = System.getenv("WX_APPID");
private static final String SECRET = System.getenv("WX_SECRET");
```

### 2. `pushMessage` 中打印 access_token

```java
String accessToken = WXAccessTokenUtils.getAccessToken();
System.out.println(accessToken);   // ← 凭证打到日志，泄露风险
```

### 3. 硬编码的调试数据残留在生产代码中

`Message` 类的默认值：

```java
private String url = "https://github.com/yihanlo/.../2026-11-24/tU854yXQnbfJ.md";
```

- 日期是 **2026-11-24（未来时间）**，文件名是随机串，明显是调试时复制的脏数据；
- 虽然 `pushMessage` 里会 `setUrl(logUrl)` 覆盖，但默认值一旦在遗漏路径上生效，用户会收到错误链接。默认值应为 `null` 或必填构造。

同时 `touser`、`template_id` 也是环境相关配置，硬编码会导致这个 SDK 对其他使用者完全不可复用。

### 4. 异常被吞掉，推送失败主流程无感知

```java
} catch (Exception e) {
    e.printStackTrace();
}
```

- 推送失败后调用方完全不知道，`codeReview` 依旧"成功"返回；
- 未检查 HTTP 状态码（4xx/5xx 时 `getInputStream()` 直接抛异常，且从未读 `getErrorStream()`）；
- 微信返回的 `errcode/errmsg` 也没有校验——**即使微信拒绝发送（如 token 失效），这里也显示"成功"**；
- 顺带一提：`scanner.useDelimiter("\\A").next()` 在响应体为空时会抛 `NoSuchElementException`。

---

## 二、🟠 设计问题

### 1. 约 80 行代码原样复制两份

`sendPostRequest`、`Message` 内部类在 `OpenAiCodeReview` 和 `ApiTest` 中完全重复。应抽取：

| 重复内容 | 建议归宿 |
|---|---|
| `Message` | `infrastructure/wechat/dto/WxTemplateMessage` |
| `sendPostRequest` | 通用 `HttpUtils`（或直接引入 OkHttp） |
| `pushMessage` 逻辑 | `WXMessageNotifier`（通知实现类） |

### 2. 职责膨胀，"上帝类"趋势

`OpenAiCodeReview` 目前承担：拉代码 → 调 OpenAI → 写日志 → 推微信 → HTTP 工具 → DTO 定义。建议按层拆分，并为后续通知渠道（钉钉/邮件/飞书）预留策略接口：

```
com.gao.love.sdk
 ├── domain/service        OpenAiCodeReview（只做编排）
 └── infrastructure
      ├── openai/          现有 OpenAI 交互
      ├── git/             现有 JGit 交互
      └── wechat/
           ├── WXMessageNotifier        implements MessageNotifier
           ├── WXAccessTokenProvider
           └── dto/WxTemplateMessage
```

### 3. access_token 无缓存

每次推送都实时请求微信 token 接口，而 token 有效期 7200s。频繁获取容易触发微信调用频次限制。建议缓存并在**过期前 5 分钟**刷新：

```java
public class WXAccessTokenProvider {
    private volatile String token;
    private volatile long expireAtMillis;

    public synchronized String getAccessToken() {
        if (token != null && System.currentTimeMillis() < expireAtMillis) {
            return token;
        }
        // 请求新 token，按 expires_in 计算，提前 5 分钟过期
        ...
    }
}
```

---

## 三、🟡 代码质量问题

1. **双大括号匿名类初始化（反模式）**：会生成额外的匿名内部类、隐式持有外部引用，序列化也可能踩坑：

```java
// 现状
data.put(key, new HashMap<String, String>() {{ put("value", value); }});
// 建议
data.put(key, Collections.singletonMap("value", value));
```

2. **HttpURLConnection 缺超时与释放**：外网调用必须设置超时，否则可能长时间挂死：

```java
conn.setConnectTimeout(5_000);
conn.setReadTimeout(10_000);
// finally 中 conn.disconnect();
```

3. **字段命名不满足 Java 规范但又是 JSON 需要**：`template_id` 这类 snake_case 字段建议用 fastjson 注解保持 Java 命名规范：

```java
@JSONField(name = "template_id")
private String templateId;
```

4. **调试输出泛滥**：`"s 评审结果 = "`、`"s1 = "`、`"s1 push消息  = "`（还有多余空格、且 s1 打印了两遍）。应引入 SLF4J 按级别输出。

5. **通配符导入** `import java.util.*;`，建议按需导入。

6. **命名语义问题**：
   - 变量 `s`、`s1` 无业务含义，建议 `reviewResult`、`logUrl`；
   - `Message.put()` 语义不清，建议 `addData(key, value)`；
   - 测试方法 `test_wx` 违反驼峰，应为 `testWxPushMessage`。

7. **语义不一致**：测试里 `"review"` 放的是评审内容摘要（`"feat: 新加功能"`），而生产代码里放的是**日志链接**（`message.put("review", logUrl)`）。模板里 review 应该推送评审摘要，链接放 `url` 字段（点击跳转），两者混用会让消息内容失去意义。

8. **测试依赖外部真实服务**：`test_wx` 直连微信接口，无法在 CI 中稳定运行，建议加 `@Ignore`/`@Tag("integration")` 或 Mock 外部依赖。

---

## 四、💡 建议的目标形态（示意）

```java
public class WXMessageNotifier implements MessageNotifier {

    private static final String SEND_URL =
            "https://api.weixin.qq.com/cgi-bin/message/template/send?access_token=%s";

    private final WXAccessTokenProvider tokenProvider;
    private final WxProperties props;   // touser / template_id 由配置注入

    @Override
    public void notify(NotifyMessage msg) {
        WxTemplateMessage body = WxTemplateMessage.builder()
                .touser(props.getTouser())
                .templateId(props.getTemplateId())
                .url(msg.getLogUrl())
                .addData("project", props.getProject())
                .addData("review", msg.getReviewSummary())   // 推摘要，而非链接
                .build();

        String resp = HttpUtils.postJson(
                String.format(SEND_URL, tokenProvider.getAccessToken()),
                JSON.toJSONString(body));

        WxResponse result = JSON.parseObject(resp, WxResponse.class);
        if (result.getErrcode() != 0) {
            throw new NotifyException("微信推送失败: " + result.getErrmsg());
        }
    }
}
```

---

## 五、评审结论

| 类别 | 项 | 处理要求 |
|---|---|---|
| 🔴 安全 | 密钥硬编码 + token 打印日志 | **必须修复，并作废已泄露的两组 SECRET** |
| 🔴 正确性 | 硬编码测试 URL 默认值、异常吞噬、未校验 errcode | 必须修复 |
| 🟠 设计 | 重复代码、职责拆分、token 缓存 | 强烈建议本次一并处理 |
| 🟡 质量 | 日志框架、命名、双大括号、超时设置 | 可本次或下迭代处理 |

修复安全问题和消除重复代码后可以重新提交评审。整体思路（评审 → 写日志 → 通知）是清晰的，把通知做成可插拔的独立模块后，这个 SDK 的扩展性会上一个台阶。