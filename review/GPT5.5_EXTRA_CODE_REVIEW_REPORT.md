# a2a-t-sdk-java 新增代码检视报告

> 检视日期：2026-06-18  
> 排重基线：`GLM5.1-CODE_REVIEW_REPORT.md`  
> 说明：以下只保留未在 GLM 报告中明确提出的问题；已由 GLM 覆盖的功能、安全、架构和实现问题不再重复列出。

## 一、功能问题

### [N-F-01] Slot schema 的 `description` 会被 `x-a2at-value-constraint` 覆盖，导致描述语义丢失并触发现有测试失败

- **严重度**：HIGH
- **文件**：
  - `a2a-t-prompt/src/main/java/net/openan/a2at/sdk/prompt/resources/model/PromptSlotJsonProperty.java:26`
  - `a2a-t-resources/src/main/resources/prompt_resources/slots/energy_saving/zh-CN/slot.json:8-12`
- **问题**：`PromptSlotJsonProperty.description` 同时标注了 `@JsonProperty("description")` 和 `@JsonAlias({"description", "x-a2at-value-constraint"})`。当资源文件同时包含标准 JSON Schema `description` 和扩展字段 `x-a2at-value-constraint` 时，Jackson 会把两个字段都映射到同一个 record component，后出现的扩展字段会覆盖原始 `description`。
- **证据**：`energy_saving/zh-CN/slot.json` 中 `"任务对象"` 的 `description` 是“明确指定节能区域信息...”，但 `x-a2at-value-constraint` 是“地理/逻辑的区域名称...”。`mvn test` 中两个 client slot schema loader 测试均失败，因为实际 `description()` 已不是测试期望的 `description` 文本。
- **影响**：调用方无法区分“给人/LLM看的描述”和“值约束”；依赖 `description()` 做提示词生成、展示或校验解释时会得到约束文本而非字段描述。当前全量测试也因此在 client 模块失败。
- **建议**：将 `x-a2at-value-constraint` 建模为独立字段，例如 `valueConstraint`，或明确合并策略，不能把它作为 `description` 的 alias。

## 二、安全风险

### [N-S-01] Central Publishing 插件挂在默认 build 且 `autoPublish=true`，存在误发布风险

- **严重度**：HIGH
- **文件**：`pom.xml:130-140`
- **问题**：`central-publishing-maven-plugin` 被配置在父 POM 的普通 `<build><plugins>` 中，并设置 `<extensions>true</extensions>`、`<autoPublish>true</autoPublish>`、`<waitUntil>published</waitUntil>`。这不是 release profile 的受控发布步骤，而是所有继承父 POM 的常规构建都会加载的发布扩展。
- **影响**：一旦构建环境存在 Central 凭据，常规 `mvn deploy` 或 CI 发布阶段可能直接自动发布到 Maven Central，缺少人工 staging/release 审核门禁。对 SDK 项目这是供应链发布风险，错误版本或包含测试/敏感配置的 artifact 可能被发布为正式版本。
- **建议**：将 Central Publishing 插件移入 `release` profile，并默认关闭 `autoPublish`，改成显式 release job 或显式 Maven profile 参数触发。

## 三、架构设计不合理

### [N-A-01] `A2ATServer` 构造一次会装配两套完整 prompt compliance runtime

- **严重度**：MEDIUM
- **文件**：
  - `a2a-t-server/src/main/java/net/openan/a2at/sdk/server/A2ATServer.java:31-37`
  - `a2a-t-server/src/main/java/net/openan/a2at/sdk/server/assembly/DefaultA2ATServerBuilder.java:142-147`
- **问题**：`A2ATServer` 构造器先调用 `builder.buildPromptComplianceOrchestrator()` 存入 `promptComplianceOrchestrator`，随后调用 `builder.buildNegotiationOrchestrator()`。但 `buildNegotiationOrchestrator()` 内部又调用一次 `buildPromptComplianceOrchestrator()` 并注入 negotiation handler。
- **影响**：同一个 server 实例内存在两套 compliance orchestrator。LLM-backed 配置下会重复创建 `LLMClient`、重复加载 prompt 资源、重复构建 extractor/validator；如果将来这些组件持有缓存、连接池、统计或限流状态，直接 API 调用和 negotiation 调用会走不同实例，行为和观测数据可能分裂。
- **建议**：`DefaultA2ATServerBuilder` 应支持传入或复用已构建的 `ServerPromptComplianceOrchestrator`；`A2ATServer` 构造器中只构建一次，并将同一实例交给 negotiation builder。

## 四、功能正确但实现不合理

### [N-I-01] 测试直接依赖 `a2a-t-resources/src/main/resources`，破坏模块边界且容易与已打包资源漂移

- **严重度**：MEDIUM
- **文件**：`a2a-t-client/src/test/java/net/openan/a2at/sdk/client/prompt/loader/LocalFileClientSlotSchemaLoaderTest.java:15-18`
- **问题**：client 模块测试通过相对路径 `../a2a-t-resources/src/main/resources/prompt_resources` 直接读取另一个模块的源码资源目录，而不是使用测试夹具、classpath artifact 或受控临时目录。
- **影响**：测试依赖当前工作目录和源码布局，单独在 `a2a-t-client` 模块目录运行、IDE 以不同 working directory 运行、或发布源码包外运行时都可能失败。它还把资源模块的内容变化直接变成 client 单元测试失败，当前 `mvn test` 的两个失败就暴露了这种耦合。
- **建议**：client loader 测试应使用本模块 `src/test/resources` 或 `@TempDir` 写入最小 slot schema；跨模块资源集成验证应放在单独 integration test 或 reactor 级测试中。

## 验证记录

- 已运行：`mvn test`
- 结果：失败于 `a2a-t-client` 模块。
- 失败用例：
  - `DefaultClasspathClientSlotSchemaLoaderTest.loadSlotSchemaReadsJsonSchemaFromClasspathPromptResources`
  - `LocalFileClientSlotSchemaLoaderTest.loadSlotSchemaDeserializesJsonSchemaModelFromLocalFile`
- 根因：`PromptSlotJsonProperty.description` 把 `x-a2at-value-constraint` 当作 `description` alias，覆盖了资源文件中的原始 `description`。
