---
title: Spring MVC API 版本路由踩坑：一个贪婪正则如何让正确接口返回 404
slug: spring-mvc-api-version-regex-404
tags: [Java, Spring MVC, Spring Boot, API 版本, 正则表达式, Bug 排查]
published: true
excerpt: 记录海外 TMS 船期详情接口的一次真实 404 排查：URL 中明明是 v1，自定义 ApiVersionCondition 却从 scheduleId 的 Kv2g... 中识别出 v2，导致 Spring MVC 淘汰正确的 Handler。本文完整梳理 RequestMappingHandlerMapping、RequestCondition、路径变量、贪婪正则的匹配链路与修复方案。
cover: ./spring-mvc-api-version-regex-404.png
metaTitle: Spring MVC API 版本路由 404：ApiVersionCondition 正则匹配踩坑与修复
metaDescription: 结合真实项目排查 Spring MVC 接口 404：ApiVersionCondition 的贪婪正则把路径参数 Kv2g... 误判为 v2，详解 RequestMappingHandlerMapping、RequestCondition、路径匹配流程、根因与修复。
metaKeywords: Spring MVC 404,ApiVersionCondition,RequestCondition,RequestMappingHandlerMapping,API 版本控制,路径匹配,Java 正则表达式,PathVariable
---

这次问题出现在海外 TMS 的船期详情接口。请求 URL 明明带的是 `v1`，Controller 也确实声明了对应的 GET 路由，但请求始终返回 404：

```text
GET http://localhost:8080/tms/overseas/v1/vessel-schedule/Kv2gMYM0mPDovhPG-VdIQg
```

最后定位到的结果很反直觉：Spring MVC 的普通路径条件已经匹配成功，自定义的 `ApiVersionCondition` 却把最后一个路径参数 `Kv2gMYM0mPDovhPG-VdIQg` 中的 `v2` 当成了 API 版本。Controller 要求版本 `1`，条件解析出来的却是 `2`，正确的 Handler 因此被候选集合淘汰，最终表现为 404。

![API 版本正则误匹配导致 Spring MVC 路由 404](./spring-mvc-api-version-regex-404.png)

本文基于项目实际使用的 Spring Boot 2.2.7.RELEASE、Spring Framework 5.2.6.RELEASE 和 `common-web` 1.27.x 梳理这次问题。重点不只是改一条正则，而是把 `@ApiVersion` 如何接入 Spring MVC、路由为什么先匹配又失败、`@PathVariable` 在什么阶段提取，以及这类问题应该怎样排查和回归测试完整串起来。

## 问题现场

船期详情 Controller 的关键代码如下：

```java
@Tag(name = "船期查询", description = "基于 4portun 点到点船期数据的检索与详情")
@ApiVersion
@RestController
@RequestMapping("/{version}")
public class VesselScheduleController {

    @Resource
    private VesselScheduleFacade vesselScheduleFacade;

    @Operation(operationId = "vesselScheduleDetail", summary = "查询船期详情")
    @GetMapping("/vessel-schedule/{scheduleId}")
    public VesselScheduleDto getScheduleDetail(
            @Parameter(description = "船期唯一标识，来自查询结果的 scheduleId")
            @PathVariable("scheduleId") String scheduleId) {
        return vesselScheduleFacade.getScheduleDetail(scheduleId);
    }
}
```

类上的路径与方法上的路径组合后是：

```text
/{version}/vessel-schedule/{scheduleId}
```

`@ApiVersion` 没有显式传值，它的默认版本是 `1`：

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ApiVersion {

    /**
     * 标识版本号，从1开始
     */
    String value() default "1";
}
```

所以从业务代码看，请求的每一部分都对得上：

| 路由模板 | 实际值 | 结果 |
|---|---|---|
| `{version}` | `v1` | 匹配 |
| `vessel-schedule` | `vessel-schedule` | 匹配 |
| `{scheduleId}` | `Kv2gMYM0mPDovhPG-VdIQg` | 匹配任意单个路径段 |
| `@ApiVersion` | 默认值 `1` | 理论上应匹配 `v1` |

但接口仍然 404，而且断点进不了 `getScheduleDetail`。这说明问题发生在 Controller 调用之前，更准确地说，是发生在 Spring MVC 选择 HandlerMethod 的阶段。

## 项目里的 API 版本机制是怎样接入 Spring MVC 的

这个项目没有只靠 `/{version}` 路径变量做版本控制，而是扩展了 Spring MVC 的 `RequestMappingHandlerMapping`，给每个带 `@ApiVersion` 的映射附加一个自定义条件。

### 用 WebMvcRegistrations 替换 HandlerMapping

`common-web` 中通过 `WebMvcRegistrations` 注册了自定义实现：

```java
@Bean
public WebMvcRegistrations webMvcRegistrationsHandlerMapping() {
    return new WebMvcRegistrations() {
        @Override
        public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
            return new CustomRequestMappingHandlerMapping();
        }
    };
}
```

Spring Boot 创建 MVC 基础设施时，会使用这里返回的 `CustomRequestMappingHandlerMapping`。它仍然具备标准 `RequestMappingHandlerMapping` 的全部能力，只是在扫描 Controller 映射时额外读取 `@ApiVersion`。

### 把 @ApiVersion 转换成 RequestCondition

自定义 HandlerMapping 的实现很短：

```java
public class CustomRequestMappingHandlerMapping extends RequestMappingHandlerMapping {

    @Override
    protected RequestCondition<?> getCustomTypeCondition(Class<?> handlerType) {
        // 扫描类上的 @ApiVersion
        ApiVersion apiVersion = AnnotationUtils.findAnnotation(handlerType, ApiVersion.class);
        return createCondition(apiVersion);
    }

    @Override
    protected RequestCondition<?> getCustomMethodCondition(Method method) {
        // 扫描方法上的 @ApiVersion
        ApiVersion apiVersion = AnnotationUtils.findAnnotation(method, ApiVersion.class);
        return createCondition(apiVersion);
    }

    private RequestCondition<ApiVersionCondition> createCondition(ApiVersion apiVersion) {
        return apiVersion == null ? null : new ApiVersionCondition(apiVersion.value());
    }
}
```

这里有两个入口：

- `getCustomTypeCondition` 读取 Controller 类上的版本。
- `getCustomMethodCondition` 读取具体方法上的版本。

Spring 在启动阶段扫描 `@RequestMapping` 时，会分别创建类级和方法级 `RequestMappingInfo`，再调用 `combine` 合并。项目的 `ApiVersionCondition.combine` 直接返回方法级条件，因此方法上的 `@ApiVersion` 可以覆盖类上的版本：

```java
@Override
public ApiVersionCondition combine(ApiVersionCondition other) {
    // 方法上的注解优于类上的注解
    return new ApiVersionCondition(other.getApiVersion());
}
```

对于当前 Controller，类上只有一个无参数的 `@ApiVersion`，最终注册的自定义条件就是版本 `1`。

## RequestCondition 的三个方法分别在什么时候执行

理解这次 404，需要先区分映射注册与请求匹配两个阶段。

### combine：启动注册阶段合并条件

`combine` 用于把类级条件与方法级条件合成最终映射。它不是每次请求都重新决定覆盖关系，而是在 HandlerMethod 注册时形成完整的 `RequestMappingInfo`。

### getMatchingCondition：请求阶段决定候选是否保留

请求到达后，Spring 会对每个可能的 `RequestMappingInfo` 调用 `getMatchingCondition`。返回一个条件实例表示匹配，返回 `null` 表示整个映射不匹配。

项目中的核心逻辑是：

```java
@Override
public ApiVersionCondition getMatchingCondition(HttpServletRequest request) {
    Matcher m = VERSION_PREFIX_PATTERN.matcher(request.getRequestURI());
    if (m.find()) {
        String version = m.group(1);
        if (this.compareTo(version)) {
            return this;
        }
    }
    request.setAttribute(API_VERSION_CONDITION_NULL_KEY, true);
    return null;
}
```

注意这里读取的是完整 `request.getRequestURI()`，再用 `Matcher.find()` 从整条 URI 中搜索版本。它没有直接读取 Controller 路径模板中的 `{version}`。

### compareTo：多个候选都匹配时决定优先级

只有至少两个映射都通过 `getMatchingCondition` 后，`compareTo` 才参与“谁更具体、谁排在前面”的比较。它不是用来判断单个候选是否匹配的；本次问题在 `getMatchingCondition` 返回 `null` 时已经结束，所以并没有走到多候选排序。

项目当前的实现是：

```java
@Override
public int compareTo(ApiVersionCondition other, HttpServletRequest request) {
    return this.compareTo(other.getApiVersion()) ? 1 : -1;
}
```

这段代码的注释写的是“版本号较大的优先”，实际却只判断字符串是否相等，并没有比较版本大小。由于 `getMatchingCondition` 采用严格相等匹配，不同版本通常不会同时留下来，因此它与本次 404 无关；但从 `RequestCondition` 的比较契约看，这仍是一处值得单独补测试和重构的技术债。

## Spring MVC 为什么会“路径匹配成功，但接口仍然 404”

一个 `RequestMappingInfo` 不只有 URL 路径。它由 HTTP Method、params、headers、consumes、produces、路径模式以及自定义条件共同组成。Spring Framework 5.2.6 中的匹配顺序可以概括为：

```text
HTTP Method
  -> params
  -> headers
  -> consumes
  -> produces
  -> URL patterns
  -> custom RequestCondition
```

任何一步返回 `null`，整个 `RequestMappingInfo` 都会被判定为不匹配。

这次请求的完整链路如下：

```text
DispatcherServlet 收到 GET 请求
  -> CustomRequestMappingHandlerMapping 查找 HandlerMethod
  -> GET 条件匹配
  -> /{version}/vessel-schedule/{scheduleId} 路径条件匹配
  -> ApiVersionCondition 检查完整 requestURI
  -> 错误提取 version = 2
  -> Controller 条件要求 apiVersion = 1
  -> getMatchingCondition 返回 null
  -> 正确的 HandlerMethod 被移出候选集合
  -> 没有可执行 Handler
  -> 返回 404
```

所以这里不是“Spring 没识别 `{scheduleId}`”，也不是业务层把数据查成空。恰恰相反，普通路径模式已经能够接受这个 `scheduleId`；失败的是随后执行的自定义版本条件。

还有一个容易误判的时序细节：在 Spring MVC 5.2 中，URI 模板变量是在选出最佳 Handler 后，由 `RequestMappingInfoHandlerMapping.handleMatch` 提取并写入 request attribute，随后参数解析器才把它绑定到 `@PathVariable`。`ApiVersionCondition` 执行时 Handler 还没有最终选定，因此不能指望此时直接取得已经解析好的 `{version}`。

## 根因：正则在整条 URI 中寻找最后一个 v+数字

修复前的正则只有一行：

```java
private static final Pattern VERSION_PREFIX_PATTERN =
        Pattern.compile(".*v(\\d+(.\\d+){0,2}).*");
```

把 Java 字符串还原成正则后是：

```regex
.*v(\d+(.\d+){0,2}).*
```

这条表达式有三个问题叠加在一起。

### 问题一：前面的 .* 是贪婪匹配

`.*` 会尽可能多地吞掉字符，然后再回溯寻找一个能够满足 `v` 加数字的位置。因此，只要 URI 后部还有 `v2`、`v3` 之类的内容，正则就更可能使用后面的那一处，而不是路径开头真正的 `v1`。

实际请求是：

```text
/tms/overseas/v1/vessel-schedule/Kv2gMYM0mPDovhPG-VdIQg
```

其中至少有两处满足“字母 v 后面跟数字”的起点：

```text
/v1/                     <- 真正的 API 版本
  ...
K v2 gMYM0mPDovhPG...    <- scheduleId 中偶然出现的字符
  ^^
```

由于开头的 `.*` 贪婪，最终捕获组 `m.group(1)` 得到的是 `2`。

### 问题二：版本前后没有路径段边界

旧正则只要求出现 `v` 和数字，不要求它们是一个独立路径段。因此下面这些业务数据都可能被当成版本：

```text
Kv2gMYM0mPDovhPG-VdIQg
invoice-v12-draft
av3x
```

对于 URL 版本来说，我们真正要识别的是 `/v1/` 这样的完整 segment，而不是任意字符串内部的 `v1`。

### 问题三：小数点没有转义

`(.\d+)` 中的 `.` 在正则里表示“任意字符”，不是字面量的小数点。因此它不仅可能匹配 `v1.2`，还可能接受类似 `v1x2` 的内容。

虽然这不是本次 `v2` 误判的直接原因，但它说明原表达式对版本格式的约束本身也不准确。

## 修复：把 API 版本限定为完整路径段

最终在修复提交 `a272c29` 中将表达式改成：

```java
/**
 * API 版本必须是一个完整的路径段，例如 /v1/、/v1.2/ 或 /v1.2.3/。
 * 避免把后续业务路径中的 v+数字（如 scheduleId=Kv2g...）误识别为 API 版本。
 */
private static final Pattern VERSION_PREFIX_PATTERN =
        Pattern.compile("(?:^|/)v(\\d+(?:\\.\\d+){0,2})(?:/|$)");
```

逐段拆开看：

| 表达式 | 含义 |
|---|---|
| `(?:^|/)` | 前面必须是字符串开头或 `/` |
| `v` | 固定的小写版本前缀 |
| `(\d+(?:\.\d+){0,2})` | 捕获一到三段数字版本，如 `1`、`1.2`、`1.2.3` |
| `(?:/|$)` | 后面必须是 `/` 或字符串结尾 |

两个 `(?:...)` 是非捕获组，只负责限定结构，不会改变 `m.group(1)` 的编号。内部小数点写成 `\.`，在 Java 字符串经过一次转义后，正则引擎收到的才是 `\.`，也就是字面量的点。

修复后，对实际 URI 使用 `find()` 时：

```text
/tms/overseas/v1/vessel-schedule/Kv2gMYM0mPDovhPG-VdIQg
              ^^
              只匹配这个完整路径段
```

`Kv2g...` 中的 `v2` 前面是 `K`、后面是 `g`，不满足路径段边界，因此不会进入候选。捕获结果恢复为 `1`，与 `@ApiVersion` 的默认值一致，`getMatchingCondition` 返回当前条件，HandlerMethod 得以保留。

这次提交同时把 `common-web` 从 1.27.0 升级到 1.27.1；上层 `common-web-plus` 1.21.1 引用 `common-web` 1.27.1，业务服务重新解析并发布依赖后即可带入修复。

## 应该怎样补回归测试

这类问题最适合直接围绕 `ApiVersionCondition` 写参数化测试。不要只测试 `/v1/order/1` 这种过于干净的路径，必须把业务 ID 中包含 `v+数字` 的情况固化下来。

下面是一组最小的 JUnit 4 测试思路：

```java
@RunWith(Parameterized.class)
public class ApiVersionConditionTest {

    @Parameterized.Parameters(name = "{index}: {0} -> v{1}")
    public static Object[][] data() {
        return new Object[][]{
                {"/tms/overseas/v1/vessel-schedule/Kv2gMYM0mPDovhPG-VdIQg", "1", true},
                {"/tms/overseas/v1/orders/invoice-v12-draft", "1", true},
                {"/tms/overseas/v1/orders/123", "1", true},
                {"/tms/overseas/v1.2/orders/123", "1.2", true},
                {"/tms/overseas/v1.2.3/orders/123", "1.2.3", true},
                {"/tms/overseas/v1x2/orders/123", "1.2", false},
                {"/tms/overseas/orders/Kv2gMYM0mPDovhPG-VdIQg", "2", false}
        };
    }

    @Parameterized.Parameter(0)
    public String uri;

    @Parameterized.Parameter(1)
    public String version;

    @Parameterized.Parameter(2)
    public boolean expected;

    @Test
    public void shouldMatchOnlyCompleteVersionSegment() {
        MockHttpServletRequest request = new MockHttpServletRequest("GET", uri);
        ApiVersionCondition condition = new ApiVersionCondition(version);

        assertEquals(expected, condition.getMatchingCondition(request) != null);
    }
}
```

此外还应增加一条真正经过 Spring MVC 路由层的 MockMvc 集成测试：使用原始 `scheduleId` 请求详情接口并断言不是 404。现有 `VesselScheduleControllerTest` 直接调用 Java 方法，只能证明 Controller 会把参数委托给 Facade，无法覆盖 `RequestMappingHandlerMapping` 和自定义 `RequestCondition`，所以没有提前发现这次问题。

可以把测试边界理解成：

| 测试类型 | 能覆盖什么 | 能否发现本次问题 |
|---|---|---|
| 直接调用 Controller 方法 | 参数传递、Facade 委托 | 不能 |
| `ApiVersionCondition` 单元测试 | 正则解析、版本条件返回值 | 能 |
| MockMvc 路由测试 | HTTP Method、路径模式、自定义条件、Handler 选择 | 能 |
| 服务层测试 | 缓存、远程调用、业务异常 | 不能 |

## 排查这类 404 的实用顺序

看到“路由看起来完全正确，但 Controller 断点不进”的 404，可以按下面的顺序缩小范围。

### 先确认请求是否到达当前服务

检查网关转发日志、服务访问日志和最终 `requestURI`。网关前缀、Servlet context path 与服务内部 lookup path 可能不同，自定义代码如果直接读取 `getRequestURI()`，尤其要确认它实际拿到的是哪一条路径。

### 再确认映射是否在启动时注册

开启 Spring MVC 映射日志，或检查 Actuator mappings，确认 Controller 对应的 HandlerMethod 已注册。没有注册通常是组件扫描、条件装配或注解声明问题。

### 区分“普通路径条件”与“完整 RequestMappingInfo”

URL pattern 匹配只是其中一项。HTTP Method、请求头、Content-Type、Accept 和自定义 `RequestCondition` 都能淘汰候选。不要看到路径模板能对上，就默认 Spring 一定会调用 Controller。

### 对自定义条件逐项看输入与返回值

这次最有效的断点是 `ApiVersionCondition.getMatchingCondition`：

```text
request.getRequestURI()
m.find()
m.group(1)
this.getApiVersion()
最终返回 this 还是 null
```

当 `m.group(1)` 显示为 `2` 时，问题就从“神秘的 Spring 404”缩小成了“正则从错误位置取值”。

### 用失败的真实数据做最小复现

如果把 `scheduleId` 随手换成 `123`，接口可能立刻恢复，这正是数据相关路由 Bug 的典型特征。排查时不要过早清洗或替换原始 ID，偶然出现的字符组合往往就是根因。

## 还能进一步改进什么

当前修复已经解决了误把业务 ID 当版本的问题，但这套版本机制还有几个可以继续收紧的地方。

第一，命名为 `VERSION_PREFIX_PATTERN`，实际约束的是“URI 中任意一个完整版本路径段”，并没有限定它一定处在固定前缀位置。如果系统路由结构稳定，可以基于服务内部 lookup path 明确版本段的位置，减少语义歧义。

第二，路径模板本身也可以收紧，例如：

```java
@RequestMapping("/{version:v\\d+(?:\\.\\d+)?(?:\\.\\d+)?}")
```

这样普通 URL pattern 会先拒绝非法版本格式。不过它只保证格式合法，不会自动完成 `@ApiVersion("1")` 与 `v1` 的值比较，因此是否保留自定义条件仍取决于项目的版本策略。

第三，如果版本控制需要支持“未声明 v3 时回退到 v2”或“多个候选按版本高低选择”，就不能再用当前的严格相等判断和 `compareTo` 实现。应该先明确精确匹配、向下兼容还是最高可用版本，再为 `getMatchingCondition` 与 `compareTo` 分别定义一致的规则。

第四，自定义条件不应顺手在每个失败候选上写一个全局 request attribute。一次请求会尝试多个映射，某个无关候选失败并不代表最终没有匹配。若要输出清晰的版本错误，最好在 HandlerMapping 的无匹配分支统一分析，而不是把候选遍历过程中的中间状态当成最终结论。

## 总结

这次 Bug 的表象是一个普通 404，真正的因果链却跨了四层：

```text
业务生成的 scheduleId 中偶然包含 v2
  -> 旧正则使用贪婪 .* 且没有路径段边界
  -> ApiVersionCondition 从完整 requestURI 提取出版本 2
  -> Spring MVC 淘汰版本条件为 1 的正确 HandlerMethod
  -> 接口返回 404
```

最终修复并不复杂：把版本限定为 `/v1/`、`/v1.2/`、`/v1.2.3/` 这样的完整路径段，并正确转义小数点。真正值得留下来的经验是：Spring MVC 的路由结果由一组条件共同决定；路径模板能够匹配，不等于整个 `RequestMappingInfo` 能够匹配。只要项目扩展了 `RequestCondition`，排查 404 时就必须把自定义条件放进第一批检查项。

参考资料：

- [Spring Framework 5.2.6 RequestCondition Javadoc](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/web/servlet/mvc/condition/RequestCondition.html)
- [Spring Framework 5.2.6 RequestMappingHandlerMapping Javadoc](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerMapping.html)
- [Spring MVC Annotated Controllers](https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/spring-framework-reference/web.html#mvc-controller)
