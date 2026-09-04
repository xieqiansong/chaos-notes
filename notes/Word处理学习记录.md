# Word 处理学习记录

工程依托：[jdk8-word-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-office-tech/jdk8-word-demo)（`chaos-java` 仓库，基于 Apache POI 5.5.1，覆盖结构化文档/样式/复杂表格/读取/模板填充/大文档内存横评）。

## 1. 安装

```bash
cd jdk8-platform/jdk8-office-tech/jdk8-word-demo
mvn test     # 编译 + 校验
mvn exec:java -Dexec.mainClass=lan.chaos.word.DemoApp     # 分节打印每个能力（0 ~ 6）
mvn exec:java -Dexec.mainClass=lan.chaos.word.DemoApp -Dbench.rows=20000   # 大文档内存横评
```

技术栈：JDK 8 + Apache POI 5.5.1（`poi-ooxml-full` + `poi` + `poi-scratchpad` 读 .doc）。产物在 `target/out/`。

## 2. 能力清单

| 节 | 能力 | 要点 |
|---|---|---|
| 1 | XWPF 结构化文档 | 标题/正文/有序无序列表/表格/图片/页眉页脚/分页 |
| 2 | 样式与中文字体（EastAsia 坑） | 中文字体须同时设 ASCII 与 EastAsia |
| 3 | 复杂表格 | 合并单元格/列宽/背景色 |
| 4 | 读取 | XWPF(.docx) vs HWPF(.doc，功能受限) |
| 5 | **模板填充（跨 run 坑 + 表格行复制）** | 占位符替换、合并 Run、`setText(t,0)` 覆盖写 |
| 6 | 大文档内存模型横评 | 不同构造策略内存对比 |

## 踩坑

- **依赖顺序**：`poi-ooxml-full` 必须排在 `poi-ooxml` 之前，否则 `STMerge` 等 Word schema 解析不到；并从三处排除 `poi-ooxml-lite`，避免裁剪版 schema 与 full 抢同一包（split package）。
- **没有流式写模型**：`XWPFDocument` 必须整篇在内存构建、一次性写盘（对比 Excel 的 SXSSF）——超大文档（数万段）会直接吃内存。
- **中文字体 EastAsia**：只设 ASCII 字体，中文会回退成默认字体；须 `setFont` 同时覆盖东亚字符集。
- **模板填充跨 run 坑**：Word 没有官方占位符引擎，本质是 XML 字符串替换；`${author}` 常被拆成多个 `Run`，逐 Run 替换永远匹配不到——须**整段文本替换 + 合并 Run**；`XWPFRun.setText(t, 0)` 才是覆盖写（否则 5.5.1 下是追加，会破坏占位符本身）。
- **列表编号**：真实可编辑编号需 `XWPFNumbering`（abstractNum + num）；前缀拼字符串只是视觉。
- **页眉页脚**：`getHeaderFooterPolicy()` 可能为 null，须先 `createHeaderFooterPolicy()` 再 `createHeader`。

## 进阶方向

- 分页/分节、目录域、邮件合并、复杂页眉页脚与图表。
- 超大文档拆分段落构建或换专门的文档生成服务（避免整篇驻留内存）。

## 参考来源

- [Apache POI (HWPF/XWPF)](https://poi.apache.org/)
