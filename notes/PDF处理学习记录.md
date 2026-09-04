# PDF 处理学习记录

工程依托：[jdk8-pdf-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-office-tech/jdk8-pdf-demo)（`chaos-java` 仓库，基于 Apache PDFBox 3.0.x，覆盖绘制/字体嵌入/表格/提取/合并拆分/大文档内存横评）。

## 1. 安装

```bash
cd jdk8-platform/jdk8-office-tech/jdk8-pdf-demo
mvn test                                             # 编译 + 校验
mvn exec:java -Dexec.mainClass=lan.chaos.pdf.DemoApp     # 分节打印每个能力（0 ~ 6）
mvn exec:java -Dexec.mainClass=lan.chaos.pdf.DemoApp -Dbench.pages=500   # 大文档内存横评
```

技术栈：JDK 8 + Apache PDFBox 3.0.x（含 fontbox / pdfbox-io，同版本）。产物在 `target/out/`。

## 2. 能力清单

| 节 | 能力 | 要点 |
|---|---|---|
| 1 | **中文字体嵌入（第一大坑）** | 标准 14 字体写中文直接抛异常；`PDType0Font.load` 嵌入真实字体；`.ttc` 集合需拆包；子集化；字体探测与失败提示 |
| 2 | 结构化文档绘制 | 坐标系原点在左下、自动分页 |
| 3 | 表格绘制 | PDF **没有表格对象**，全部用线条 + 文本画出来 |
| 4 | 内容提取（乱序坑） | `PDFTextStripper` 的 `setSortByPosition` 排序 vs 默认内容流顺序；表格/扫描件不可靠 |
| 5 | 合并与拆分 | `PDDocument` 合并、按页码范围拆分 |
| 6 | 大文档内存模型横评 | 不同构造策略内存对比 |

## 踩坑

- **PDFBox 3.0 破坏性变更**（旧博客会编译不过）：读取 `PDDocument.load` → `Loader.loadPDF`；`PDPageContentStream` 新增 `(doc, page)` 两参构造；`PDType1Font.HELVETICA` 常量移除 → `new PDType1Font(Standard14Fonts.FontName.HELVETICA)`；异常文案 `No glyph for U+4E2D` → `... is not available in the font ...`，断言应匹配码点而非整句。
- **中文字体必须嵌入**：标准 14 字体无任何中文字形；字体文件 10~20MB 且授权各异，**不能随包**，应作部署物；全量嵌入致 PDF 暴涨，需**子集化**（通常降到几十 KB）；容器镜像漏装字体是线上最常见事故，须有明确失败提示。
- **内容提取不可靠**：PDF 是「版面格式」而非「结构化文档」，无段落/表格/阅读顺序；默认按内容流顺序输出，双栏/表格必然乱序；开启排序只是按 (y,x) 排序，旋转页/跨页多栏仍会串；**表格提取基本不可靠**，数据交换别选 PDF；扫描件需先 OCR。
- **中文宽度需字体 `getStringWidth`**；坐标系 y 轴向下需换算。

## 进阶方向

- 模板填充 / 表单（AcroForm）、数字签名、水印。
- 大文档用增量保存 / 对象流式写入控制内存。
- 提取场景优先选结构化格式（Excel/数据库），PDF 只做呈现场景。

## 参考来源

- [Apache PDFBox](https://pdfbox.apache.org/)
