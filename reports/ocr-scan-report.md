# OCR 代码评审扫描报告

## 一、扫描概览

| 项目 | 内容 |
| --- | --- |
| 扫描工具 | ocr review |
| 扫描目标 | `C:/ACodeTime/ideaspace/test-sca` |
| 变更文件数 | 7 个 |
| 实际评审文件数 | 6 个 |
| 跳过文件 | `src/main/java/com/example/testSca/controller/chinese.zip`(二进制文件);另有 1 个文件被 include/exclude 规则过滤 |
| 评审意见总数 | **22 条** |
| Token 消耗 | 约 169,155(输入 ~147,437 / 输出 ~21,718;缓存读取 ~57,600) |
| 扫描耗时 | 5 分 47 秒 |

> 注:`oss.java`、`build.gradle`、`HelloController.java`、`ZipController.java` 因行数低于阈值(50 行)跳过了 Plan 阶段,直接进入评审。

## 二、问题统计

### 2.1 按严重级别

| 严重级别 | 数量 | 占比 |
| --- | --- | --- |
| 🔴 严重(critical) | 2 | 9.1% |
| 🟠 高(high) | 6 | 27.3% |
| 🟡 中(medium) | 7 | 31.8% |
| 🟢 低(low) | 7 | 31.8% |
| **合计** | **22** | 100% |

### 2.2 按问题类别

| 类别 | 数量 |
| --- | --- |
| 缺陷(bug) | 8 |
| 安全(security) | 3 |
| 可维护性(maintainability) | 4 |
| 代码风格(style) | 3 |
| 性能(performance) | 1 |
| 其他(other) | 3 |

### 2.3 按文件分布

| 文件 | 问题数 | 严重 | 高 | 中 | 低 |
| --- | --- | --- | --- | --- | --- |
| `controller/UtfController.java` | 10 | 1 | 4 | 2 | 3 |
| `controller/ZipController.java` | 6 | 0 | 1 | 2 | 3 |
| `build.gradle` | 4 | 1 | 0 | 3 | 0 |
| `exception/GlobalExceptionHandler.java` | 2 | 0 | 1 | 0 | 1 |
| `controller/HelloController.java` | 0 | — | — | — | — |
| `oss/oss.java` | 0 | — | — | — | — |

## 三、重点风险(需优先处理)

### 🔴 严重级别(2 项)

#### 1. 路径穿越漏洞(CWE-22 / CWE-200)
- **位置**:`UtfController.java:67-68`
- **类别**:security · critical
- **描述**:用户传入的 `name` 参数未经任何校验直接拼接到文件路径中,攻击者可通过 `..\..\windows\win.ini` 或绝对路径读取服务器任意文件,且文件内容会随 HTTP 响应返回。
- **修复建议**:使用 `FilenameUtils.getName()` 剥离目录部分,限定文件访问在指定基目录内,并通过 `getCanonicalPath()` 校验最终路径未越界;拒绝包含 `..`、路径分隔符或绝对路径前缀的文件名。

#### 2. 仓库凭据明文硬编码
- **位置**:`build.gradle:19-23`
- **类别**:security · critical
- **描述**:Maven 仓库的用户名/密码明文硬编码并提交到版本库,任何有仓库访问权限的人都能看到,且即使后续删除,git 历史中仍会保留。
- **修复建议**:**立即轮换该密码**;凭据改由 `gradle.properties`(排除在 VCS 之外)、环境变量或 CI 密钥库注入,例如 `findProperty('giteeMavenUser') ?: System.getenv('GITEE_MAVEN_USER')`。

### 🟠 高级别(6 项)

| # | 位置 | 类别 | 问题摘要 |
| --- | --- | --- | --- |
| 3 | `GlobalExceptionHandler.java:12-13` | bug | 整个 `@ControllerAdvice` 类被注释,全局异常处理失效;全库无其他兜底 handler,异常将回落到 Spring 默认处理,可能向 API 客户端暴露绝对路径与堆栈等内部信息,统一的 `Resp.error(code, msg)` 响应契约也随之失效 |
| 4 | `ZipController.java:40-41` | bug | catch 块静默吞掉 `IOException`,文件缺失/损坏时接口仍返回路径且无任何日志;建议 `log.error(...)` 记录后重新抛出,或直接去掉 try-catch 交给全局异常处理 |
| 5 | `UtfController.java:22` | bug | 硬编码开发者本机绝对路径 `C:\Users\nicex\Desktop\ut16.txt`,部署到任何服务器都会抛 `FileNotFoundException`;应外置到配置(如 `@Value`)或作为请求参数 |
| 6 | `UtfController.java:35-36` | bug | `detector.detect()` 返回 null 时,下一行调用 `charsetMatch.getName()` 会抛 NPE;应仿照 `getCharset()` 的判空逻辑做兜底 |
| 7 | `UtfController.java:32-34` | bug | `FileInputStream` 未关闭(无 try-with-resources / finally),反复请求会耗尽文件描述符导致 "Too many open files" |
| 8 | `UtfController.java:46-47` | bug | 同上,另一处 `FileInputStream` 资源泄漏,应改用 try-with-resources |

## 四、中级别问题(7 项)

| # | 位置 | 类别 | 问题摘要 |
| --- | --- | --- | --- |
| 9 | `build.gradle:47` | security | `commons-compress:1.21` 存在已知漏洞(CVE-2024-25710、CVE-2024-26308,精心构造的压缩包可导致 DoS),且该依赖正用于处理压缩文件;建议升级至 1.26.0+ |
| 10 | `build.gradle:48-49` | maintainability | 同时声明 `hutool-ai:5.8.38` 与 `hutool-all:5.5.3`,`hutool-all` 已包含全部模块,两个不同版本会引起类路径冲突;应保留单一对齐版本(如 `hutool-all:5.8.38`) |
| 11 | `build.gradle:8` | other | 新增的版本号为 SNAPSHOT(`25.08.26-1-SNAPSHOT`),生产发布不应使用快照版本,应改为固定版本 `25.08.26-1` |
| 12 | `ZipController.java:27` | maintainability | 硬编码 Windows 绝对路径 `D:\CodeTime\encode\chinese.zip`,换机器/换系统即失效;应通过 `@Value`、`application.properties` 或请求参数外置 |
| 13 | `ZipController.java:28` | other | 变量 `encoding`("UTF-8")声明后从未使用,`ZipInputStream` 实际硬编码了 `Charset.forName("gbk")`,二者不一致;应使用该变量或删除 |
| 14 | `UtfController.java:54-55` | bug | `detector.getDetectedCharset()` 可能返回 null,传入 `readFileToString` 后会静默退回平台默认字符集产生乱码;应判空并回退 `DEFAULT_ENCODING` |
| 15 | `UtfController.java:69-72` | performance | `junhuademo` 对同一文件重复打开三次(两次探测 + 一次读取),探测结果应只检测一次并复用 |

## 五、低级别问题(7 项)

| # | 位置 | 类别 | 问题摘要 |
| --- | --- | --- | --- |
| 16 | `ZipController.java:23` | other | `DEFAULT_ENCODING` 常量声明后从未引用,属死代码;删除或实际用于构造 `ZipInputStream` |
| 17 | `ZipController.java:36-37` | maintainability | Web 控制器中使用 `System.out.println` 绕过日志框架;类上已有 `@Slf4j`,应改为 `log.info(...)` |
| 18 | `ZipController.java:6-7` | style | 存在未使用的 import:`ZipArchiveEntry`、`ZipFile`、`StandardCharsets`、`Enumeration`;应移除 |
| 19 | `GlobalExceptionHandler.java:1` | maintainability | 整个类以大段注释形式保留,无 TODO 或禁用原因说明;若确定禁用应整体删除(git 历史可追溯),若临时禁用应注明原因与恢复条件 |
| 20 | `UtfController.java:57-58` | bug | 调用 `FileUtils.readFileToString` 后丢弃返回值,下一行又做了相同调用赋值给 `content`,重复 I/O;删除冗余读取 |
| 21 | `UtfController.java:98-100` | style | catch 块中使用 `e.printStackTrace()` 绕过 SLF4J 日志;应改为 `log.error("Failed to close stream", e)` |
| 22 | `UtfController.java:19-21` | style | 命名拼写错误:端点 `/setUt8` 与方法 `ut8()` 应为 `Utf8`(类名为 `UtfController`,注释也写明"手动指定UTF-8解码"),影响 API 可读性与可发现性 |

## 六、修复优先级建议

1. **立即处理(阻断合并)**
   - 轮换 `build.gradle` 中泄露的 Maven 仓库密码,并将凭据外置(#2)。
   - 修复 `UtfController.junhuademo` 的路径穿越漏洞(#1)。
2. **合并前处理**
   - 恢复或替换全局异常处理器 `GlobalExceptionHandler`,避免内部信息外泄(#3)。
   - 修复两处 `FileInputStream` 资源泄漏(#7、#8)与两处字符集探测判空缺失(#6、#14)。
   - 不再静默吞掉 `IOException`(#4)。
   - 升级 `commons-compress` 至 1.26.0+,并统一 hutool 依赖版本(#9、#10)。
3. **尽快处理**
   - 外置所有硬编码绝对路径(#5、#12),生产版本去掉 SNAPSHOT(#11)。
4. **择机清理**
   - 死代码、未用 import、`System.out.println` / `printStackTrace`、命名拼写等风格问题(#13、#15–#22)。

## 七、总体评价

本次变更整体质量风险偏高:22 条意见中有 **2 条严重 + 6 条高危**,集中于**安全(凭据泄露、路径穿越、含漏洞依赖)**与**健壮性(资源泄漏、NPE、异常吞噬、全局异常处理被禁用)**两大方向。其中凭据泄露与路径穿越两项属于可直接被利用的安全隐患,建议在合并前完成修复并轮换已暴露的密码;`UtfController.java` 是问题最集中的文件(10 条,含 1 严重 4 高危),建议对其做一次整体重构。
