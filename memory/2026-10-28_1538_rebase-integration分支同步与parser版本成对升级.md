# rebase origin/integration 分支同步与 parser 版本成对升级

- **执行时间**：2026-10-28 15:00 ~ 15:40
- **仓库**：`D:\Code\JavaCode\opendataloader-pdf-parse\veraPDF-validation`
- **分支**：`valueonline-20260722`
- **远程**：`origin`（github.com:UnicornLee/veraPDF-validation.git）、`repo-b`（ssh://git@git.valueonline.cn:9999/xiaoboli/veraPDF-validation.git）
- **关联仓库**：`D:\Code\JavaCode\opendataloader-pdf-parse\opendataloader-pdf`（分支 `ocr-unification-20260820`，作为消费方）

---

## 一、任务目标

1. 将 `valueonline-20260722` rebase 到 `origin/integration`
2. 解决 rebase 冲突，拿不定的点与用户确认
3. 推送远程，并 `mvn install` 到本地仓库
4. 追加需求：推送 `repo-b`；核查消费方 `opendataloader-pdf` 的 parser 版本是否需要升级

## 二、rebase 前后状态

| 项目 | 值 |
| --- | --- |
| rebase 前 HEAD（= origin = repo-b） | `a4a25536` |
| rebase 基底（merge-base） | `dc9d6f60` |
| 本地领先 integration | 10 个提交 |
| integration 领先本地 | 5 个提交 |
| rebase 后 HEAD | `0e4b958c` |

### integration 带进来的 5 个提交

| 提交 | 说明 | 对本仓影响 |
| --- | --- | --- |
| `e12da84a` | `PDNameTreeNode.getNames()` 键类型 String → COSString | **破坏性**：要求 parser ≥ 1.31.68 |
| `f4eef08a` | `getremappedStandardType` 返回参数调整 | 无冲突 |
| `eb9b71f8` | 每个 line chunk 只保留绘制它的 operator | **与本地 `e782d4e6` 等价**，产生冲突 |
| `587e98fb` | marked content 类父标签方法修复 | 无冲突 |
| `d43fca96` | GFPDAnnot / GFSAAnnotation 改用 `PDAnnotation.getAlt()` | 无冲突 |

### rebase 后的 10 个本地提交（由旧到新）

1. `c8638854` LinkChunk 的 strokeColor 识别
2. `08248f63` 增加显式 JPEG2000 依赖并强制加载 ImageReader SPI
3. `af9a4b3b` ChunkParser 实现 stripe collapsing（细四边形表格边框折叠）
4. `5d9cd310` 修复颜色空间处理中的资源处理器引用错误
5. `c588d968` 版本升至 1.31.99-valueonline
6. `bd5202ea` 增加 distributionManagement（stock 仓库）
7. `755c2f19` 更新 wcag-algorithms 依赖版本
8. `0e3bd0e0` 停止向每个 line chunk 复制完整 StreamInfo 列表（冲突解决后生成）
9. `38df8807` 版本升至 1.31.100-valueonline
10. `0e4b958c` pin veraPDF 依赖版本（含本次 parser 升到 1.31.68，amend 而成）

## 三、冲突处理（唯一一处）

- **文件**：`wcag-validation/src/main/java/org/verapdf/gf/model/factory/chunks/ChunkParser.java`
- **位置**：`processLineArts()` 中 `lineChunks` 非空分支
- **性质**：**纯注释冲突**，标记外的两行代码双方完全一致（`artifacts.addAll(lineChunks); lineChunks.clear();`）
- **原因**：本地提交 `e782d4e6` 与上游 `eb9b71f8` 是同一问题的等价修复——都删除了"给每个 LineChunk 深拷贝整份 pending StreamInfo 列表"的双重循环，仅注释措辞不同

### 解决方式

将冲突标记与两侧注释替换为一段合并注释（保留双方要点：上游说明"每行已由 processLineChunk 自带 operator"，本地说明"附加代价是 O(行数×算子数) 对象、曾导致 OOM"），代码逻辑零改动：

```java
if (lineChunks != null && !lineChunks.isEmpty()) {
    // Each line already holds its own operator (processLineChunk) and the line
    // chunks are added raw, as parseLineArts does. Do not attach the region's
    // stream positions to every line chunk: that materializes lineChunks.size()
    // x pending.size() StreamInfo objects and exhausted the heap on vector-heavy
    // pages. The region's list is attached to the LineArtChunk built from the
    // bounding boxes below instead, once per box.
    artifacts.addAll(lineChunks);
    lineChunks.clear();
}
```

### 事后验证

确认上游在 `processLineChunk` 新增的 per-operator 写入已自动合入（第 887-889 行），否则合并注释中"每行已自带 operator"的说法不成立：

```java
if (StaticContainers.isDataLoader()) {
    lineChunk.getStreamInfos().add(new StreamInfo(operatorIndex, xObjectName, null));
}
```

> 说明：该提交相对 integration 的**实质代码贡献为零，只剩注释**。保留而未 `--skip`，是为了留存"7,500 条笔画 → 5,600 万对象 → OOM"的事故记录。

## 四、rebase 后的编译失败与根因

rebase 完成后 `mvn compile` 失败，**不是冲突导致**，而是上游 `e12da84a` 与本地依赖 pin 相冲突：

```
GFFeatureParser.java:[429,64] 不兼容的类型:
java.util.Map<String,COSObject> 无法转换为 java.util.Map<COSString,COSObject>
GFFeatureParser.java:[442,64] 同上
```

### 根因：API 由 parser 版本决定

用 `javap` 对比 parser jar 的 `org.verapdf.pd.PDNameTreeNode`：

| parser 版本 | `getNames()` 键类型 | `getObject` 签名 |
| --- | --- | --- |
| ≤ 1.31.65（含 pin 的 1.31.62） | `Map<String, COSObject>` | `getObject(String)` |
| 1.31.68 | `Map<COSString, COSObject>` | `getObject(COSString)`，**String 重载被删除** |

- 本地 pin 提交 `a4a25536` 当初 pin 1.31.62，理由是"当时源码用 String API、1.31.68 编译不过"
- rebase 后源码已切到 COSString API → pinned 1.31.62 反向不兼容

### 处置（经用户确认）

`pom.xml` 的 `verapdf.parser.version`：`1.31.62` → `1.31.68`；`core`/`verapdf-xmp-core` 保持 `1.31.45`、`pdf-model` 保持 `1.31.14`（已验证无需同步升）。改动并入原 pin 提交并更新提交说明，最终形成 `0e4b958c`。

验证：`mvn -Dverapdf.parser.version=1.31.68 compile` → 5 个模块全部 BUILD SUCCESS。

## 五、安装到本地仓库

```
mvn -B clean install -DskipTests
```

- 结果：BUILD SUCCESS
- 产物版本：`1.31.100-valueonline`
- 安装位置：`D:\Maven_Repo`（该机 Maven `localRepository` 不是 `~/.m2`，在 settings.xml 中配置为 `D:\Maven_Repo`）
- 已安装模块：`validation`(pom)、`feature-reporting`、`wcag-validation`、`metadata-fixer`、`validation-model`
- 核对：安装出的 pom 中 `<verapdf.parser.version>1.31.68</verapdf.parser.version>` 已生效

## 六、推送结果

| 远程 | 分支 | 结果 |
| --- | --- | --- |
| origin | valueonline-20260722 | 强制更新 `a4a25536` → `0e4b958c`（`--force-with-lease`） |
| repo-b | valueonline-20260722 | 强制更新 `a4a25536` → `0e4b958c`（`--force-with-lease`） |

- 推 repo-b 前先 `fetch` 确认其仍停在 `a4a25536`（与 origin 原值一致，无第三方新提交），未覆盖他人工作
- 最终本地、origin、repo-b 三者均指向 `0e4b958c`

## 七、消费方 opendataloader-pdf 的版本升级（成对约束）

### 核查发现的硬约束

用 `javap` 反编译两个产物，确认调用签名不同：

| 产物 | `GFPDLinkAnnot` 调用 | 要求 parser |
| --- | --- | --- |
| `validation-model-1.31.99-valueonline` | `PDNameTreeNode.getObject(String)` | ≤ 1.31.65 |
| `validation-model-1.31.100-valueonline` | `PDNameTreeNode.getObject(COSString)` | ≥ 1.31.68 |

因此 **`verapdf.version` 与 `verapdf.parser.version` 必须成对升级**，单升任一侧都会在运行期抛 `NoSuchMethodError`。用户选择"成对升级"。

### 实际修改（`opendataloader-pdf/java/pom.xml`）

```xml
<verapdf.version>1.31.100-valueonline</verapdf.version>   <!-- 原 1.31.99-valueonline -->
<verapdf.parser.version>1.31.68</verapdf.parser.version>  <!-- 原 1.31.62 -->
```

同时在 pom 注释中写明成对约束的原因，避免后续有人只升一侧。

### 影响面

`PDNameTreeNode` 全仓仅 3 个使用点：

| 使用点 | 用法 | 运行期风险 |
| --- | --- | --- |
| `GFPDLinkAnnot.getStructureDestinationObject` | `getObject(COSString)` | **有风险**（唯一） |
| `GFCosDocument.getNamesEmbeddedFiles` | 仅 `getNames().values()` | 无（泛型擦除，字节码无差异） |
| `GFFeatureParser.reportJavaScripts/reportEmbeddedFileNode` | 仅 `getNames().values()` | 无 |

触发条件：Link 注解的 `/Dest` 为 `COS_STRING` 且文档存在 Names/Dests 名称树（Word/WPS 导出 PDF 常见）时才会走到。

### 验证

- JDK 17 下编译：`opendataloader-pdf` 4 个模块全部 BUILD SUCCESS（含 enforcer `ban-dynamic-versions`）
- 依赖树确认：`validation-model`/`wcag-validation`/`metadata-fixer` = `1.31.100-valueonline`，`parser` = `1.31.68`，`core`/`xmp-core` = `1.31.45`
- **未提交**：该仓库改动仅 `java/pom.xml`（+10/-6），仍为工作区修改

## 八、遗留事项

1. `opendataloader-pdf` 的 pom 改动**未提交**，且落在 `ocr-unification-20260820` 分支，需确认目标分支
2. 未运行 veraPDF-validation 测试套件（install 用了 `-DskipTests`）
3. 未做运行时冒烟：缺一个"含字符串命名目标链接"的 PDF 样例来验证 `GFPDLinkAnnot` 路径
4. 反向约束：若将来把 `verapdf.version` 回退到 99，parser 必须同步降回 ≤ 1.31.65

## 九、本次踩坑与命令要点

1. **PowerShell 不支持 `&&`**，多命令需用 `;` 分隔
2. **`-D` 参数含点号必须加引号**：`mvn "-Dverapdf.parser.version=1.31.68" compile`，否则被拆成 lifecycle phase 报错
3. **`Select-String -Path` 的 `**` 不递归**，跨目录搜源码改用 `Get-ChildItem -Recurse | Select-String`
4. **mvn 默认 JDK 11**，构建 `opendataloader-pdf`（source/target 17）需先 `$env:JAVA_HOME="D:\Applications\Java\jdk-17.0.0.1"`
5. **本地仓库不在 `~/.m2`**，是 `D:\Maven_Repo`，排查依赖版本时不要看错目录
6. `git push --force-with-lease` 前先 `fetch` 对应远程，确保 lease 基线是最新值
