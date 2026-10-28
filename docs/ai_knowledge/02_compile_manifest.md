# 编译系统：Manifest 增量合并

> 最后核对：2026-10-28
> 一致性规则：文档与代码冲突时，以代码为准。

---

## 1. 文档定位

本页只覆盖 AndroidManifest 的增量合并：从变更 manifest 到 APK-scoped `AndroidManifest.xml` overlay。

资源 flat/link 细节见 `02_compile_resource.md`；源码到 dex 的顺序见 `02_compile_source.md`；release 混淆映射见 `02_compile_obfuscation.md`。

---

## 2. 核心源码索引

| 类/接口 | 文件 | 作用 |
|---|---|---|
| `AndroidManifestCompiler` | `main/src/main/java/com/sickworm/intellij/jugg/compiler/manifest/AndroidManifestCompiler.kt` | Manifest 编译入口；按 APK 归属读取基准 merged manifest，补 placeholder 后产出部署用 manifest overlay |
| `AndroidManifestMerger` | `main/src/main/java/com/sickworm/intellij/jugg/compiler/manifest/AndroidManifestMerger.kt` | 在已合并 manifest 上套用变更 diff；不是重新跑标准 ManifestMerger2 |
| `ManifestDiffer` | `main/src/main/java/com/sickworm/intellij/jugg/compiler/manifest/ManifestDiffer.kt` | 比较原 manifest 与变更 manifest，生成需要 patch 到 merged manifest 的节点/属性 |
| `ManifestNodeMatcher` | `main/src/main/java/com/sickworm/intellij/jugg/compiler/manifest/ManifestDiffer.kt` | 在 merged manifest 子树里找相对节点，决定新增节点或递归更新 |

---

## 3. 核心数据流

| 数据 | 生产者 | 消费者 | 关键约束 |
|---|---|---|---|
| 基准 merged manifest | Gradle 上次构建产物，或 Jugg 上轮写入的目标 APK Manifest 缓存 | `AndroidManifestCompiler` | 主 App 沿用 `tempModule/res/AndroidManifest.xml`；library Test APK 使用独立缓存与对应 androidTest module 的完整 Manifest，不能回退到主 App |
| `ChangedManifestFile` | `AndroidManifestCompiler` | `ManifestDiffer` | application 类 manifest 使用当前目标 APK 的 `applicationId`；library 显式配置同名 placeholder 时保留原值，否则补目标 APK 值；存在 namespace 时补 `JUGG_NAMESPACE_IN_GRADLE` |
| Manifest diff element | `ManifestDiffer` | `AndroidManifestMerger` | 只携带新增节点和新增/更新属性；删除节点、删除属性和 `tools:node="remove"` 不进入 patch |

---

## 4. 核心调用链路

```text
ResourceOverlayCompiler.doApkCompile()
  -> 按 APK scoped 任务调用 AndroidManifestCompiler.doApkCompile()
  -> 选择基准 manifest：优先当前目标 APK 的 Jugg 缓存，否则主 App 用 application module，Test APK 用对应 androidTest module 的完整 Manifest
  -> 为变更 manifest 补当前目标 APK 的 applicationId / namespace placeholder，并为 library manifest 找上次构建相对 manifest
  -> ManifestDiffer.diff() 只提取真实新增/变更节点
  -> AndroidManifestMerger.merge() 将 diff patch 到基准 merged manifest
  -> 成功后写回当前 APK 的独立缓存，并输出 apkPath 绑定的 CompileOutput.Type.Res
```

这条链路的关键点是“patch merged manifest”，不是重新跑完整 Gradle manifest merge。标准 `ManifestMerger2` 需要完整 placeholder、variant/flavor manifest、依赖 manifest 和 merge feature 上下文；增量现场不能保证这些输入与上次 Gradle 构建完全一致。Jugg 因此把上次最终 merged manifest 当作稳定基线，只套用能够确定恢复的局部变化。

Manifest patch 是有意保守的：`ManifestDiffer` 只遍历新 manifest 中存在的节点和属性，新增节点或属性变化才进入 `DiffElement.changedChildren/changedAttributes`；旧 manifest 中存在、当前已删除的节点或属性不会生成删除操作。`tools:node="remove"` 会被视为无 patch，其他 `tools:*` 属性也不会写进最终 merged manifest。这些删除或完整 merge 指令不会单独导致增量编译失败或自动回退，已安装 APK 继续保留原有 merged manifest 内容；只有需要让删除真正生效时，才通过完整 Gradle merge 刷新基线。

---

## 5. 隐形约束 / 设计思路 / 已知边界

- Manifest 输出为空是有效结果：library manifest 未变更、diff 后无变化时不会输出 `AndroidManifest.xml`，避免触发无意义 APK repackage。
- `AndroidManifestCompiler` 会把成功合并结果复制回目标 APK 的缓存；主 App 保留 `tempModule/res/AndroidManifest.xml`，Test APK 按 APK 唯一 key 隔离。后续增量优先使用对应缓存，不能只看 Gradle merged manifest。
- 普通 library main Manifest 仍按 APK 归属分别 patch 到主 APK 与 self-targeting library Test APK；App main Manifest 不会因此进入 library Test APK。两份输出保留各自 package、application 配置和 instrumentation。
- `ModuleBuildPathInfo.mergedManifest` 在既有 merged manifest 候选中选择最新文件；androidTest variant 还包含 `packaged_manifests/<variant>` 并同步到远端构建基线缓存。测试基线缺失时明确失败并提示完整 Gradle 构建，禁止使用主 App Manifest 顶替。
- library manifest 会先和 `oldManifest` 做 CRC 比较；未变化直接跳过，避免对依赖库 manifest 做重复 patch。
- Manifest merge 会忽略 `tools:*` 属性、manifest `package` 和 application `android:name` 更新；这不是漏合并，而是为了避免增量 patch 覆盖运行时关键身份。
- 删除节点、删除属性和 `tools:node="remove"` 被故意忽略。增量路径只做可确定的新增/更新，避免错误删除最终 merged manifest 中由其他 source set 或依赖贡献的声明。
- `tools:replace` 等 merge 指令在最终 merged manifest 中已经丢失完整上下文，不能把它们当普通属性直接 patch；需要完整语义时走 Gradle fallback。
- 保留旧声明可能产生开发期 false positive，例如源码删除 Activity 后基线 manifest 暂时仍有注册；这是保守增量的已知代价，不应通过在 patch 层猜测删除来源来修复。

---

## 6. 排查入口

| 现象 | 优先入口 |
|---|---|
| Manifest 修改后未生效 | `AndroidManifestCompiler.doApkCompile()`：确认是否被 CRC、empty diff 或 `filterResources` 过滤 |
| 删除节点或 `tools:remove` 后声明仍存在 | 当前增量 patch 不处理删除语义；执行完整 Gradle build 刷新 merged manifest 基线 |
| manifest 合并结果覆盖了不该覆盖的字段 | `AndroidManifestMerger.merge()`：检查 `tools:*`、`package`、`android:name` 的忽略规则 |
| aapt2 link 后触发不必要重打包 | `ResourceOverlayCompiler.filterResources(...)`：确认无 manifest 变更时是否仍输出根 `AndroidManifest.xml` |

---

## 7. 关联文档

- 资源编译：`02_compile_resource.md`
- 源码编译：`02_compile_source.md`
- 编译核心调度：`02_compile_core.md`
- 混淆映射：`02_compile_obfuscation.md`
