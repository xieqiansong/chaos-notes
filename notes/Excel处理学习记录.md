# Excel 处理学习记录

工程依托：[jdk8-excel-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-office-tech/jdk8-excel-demo)（`chaos-java` 仓库，覆盖原生 POI / EasyExcel / Hutool 三大体系的读写、大文件流式、模板填充，并用相同数据做导入/导出压测横评）。

## 1. 安装

```bash
cd jdk8-platform/jdk8-office-tech/jdk8-excel-demo
mvn test     # 跑全部测试（含横评），零外部依赖
mvn exec:java -Dexec.mainClass=lan.chaos.excel.DemoApp   # 分节打印每个能力（0 ~ 8）
mvn test -Dtest=ExcelBenchTest -Dbench.rows=100000       # 大文件下看内存分野
```

技术栈：JDK 8 + Apache POI 5.5.1（HSSF/XSSF/SXSSF）+ Alibaba EasyExcel 4.0.3 + Hutool-all。3 个测试类 16 用例全过。

## 2. 能力清单

| 节 | 能力 | 要点 |
|---|---|---|
| 1 | 基础 | HSSF(.xls，硬上限 **65536** 行) / XSSF(.xlsx 全内存) / SXSSF（滑动窗口流式，**刷出的行不可再改**）|
| 2 | 样式与公式 | `CellStyle` 必须复用（进程上限 **64000** 个）；公式必须先求值再读，否则读回 0 |
| 3 | 大文件读取 | 用户模型（全内存，OOM 头号元凶）vs **SAX 事件模型**（流式、内存恒定）|
| 4 | EasyExcel | 注解式写、CSV、监听器**分批读**（批量入库接入点）、模板填充 |
| 5 | Hutool 轻量封装（对照）| 三行写出；底层仍是 POI 全内存 |
| 7 | **导出横评** | 同一份 2 万行×7 列喂 6 个写方案对比耗时/内存/文件大小 |
| 8 | **导入横评** | 读同一文件，4 个读方案对比 |

## 3. 压测横评结论

- 导出：**EasyExcel / CSV 最快最省**；Hutool 全内存封装最慢最吃内存（印证「底层仍是 POI 全内存」）。
- 导入：**EasyExcel 监听器综合最优**（快且省）。
- 完整数据见模块 `TEST-REPORT.md` 与 `target/bench-results.md`。

## 踩坑

- **HSSF 上限 65536 行**；XSSF 内存随行数线性增长；SXSSF 刷出窗口的行不可再读、改样式会失败。
- **单元格样式对象有 64000 上限**，必须复用，否则抛异常。
- **公式读取前必须求值**，否则为 0。
- **SAX 读取**：共享字符串表整体进内存、需解析 `cellReference` 处理空单元格、只能顺序读。
- Hutool 不自带 POI；`@Alias` 与 `@ExcelProperty` 不互通；写已存在文件会脏数据（须先删）。
- EasyExcel 底层仍是 POI；**监听器内不能逐条写库，须攒批**（`PageReadListener`）。
- **版本守门**：POI 与 poi-ooxml 同版本 5.5.1（收敛掉 EasyExcel 传递的 5.2.5）；commons-compress / commons-io 不被 Spring Boot BOM 压回（防 CVE 修复被吞）。

## 进阶方向

- 超大文件一律 SXSSF（写）/ SAX 或 EasyExcel 监听器（读）控内存。
- 模板填充、多 sheet、样式复用策略、读写版本矩阵守门。

## 参考来源

- [Apache POI](https://poi.apache.org/)
- [Alibaba EasyExcel](https://github.com/alibaba/easyexcel)
- [Hutool Excel](https://hutool.cn/docs/#/poi/Excel%E5%B7%A5%E5%85%B7-ExcelUtil)
