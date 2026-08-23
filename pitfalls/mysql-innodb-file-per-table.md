# MySQL 5.5 共享表空间改独立表空间（innodb_file_per_table）

> 分类：数据库 / InnoDB
> 踩坑耗时：长时间（表空间结构问题，改动需重建数据文件）
> 整理时间：2026-08-23
> 发生时间：约 2022-06

## 场景

生产环境的 **MySQL 5.5** 采用默认的**共享表空间**（所有 InnoDB 表数据、索引都集中在同一个 `ibdata1` 文件中）。随着业务增长，`ibdata1` 持续膨胀、难以单独收缩某个表，希望改为**每表一个文件**的独立表空间（`innodb_file_per_table = 1`）。

## 表现

- `ibdata1` 单文件越来越大，占用空间难以回收（删除表/数据后文件不会变小）。
- 数据目录中看不到按表名区分的独立 `.ibd` 文件，所有表都挤在一个大文件里。
- 备份、迁移、单独优化某张表困难。

## 排查

1. 确认当前 MySQL 版本与表空间模式：
   ```sql
   SELECT VERSION();                    -- 5.5.x
   SHOW VARIABLES LIKE 'innodb_file_per_table';   -- 当前为 OFF
   ```
2. 查看数据目录，确认只有 `ibdata1` 而无独立的 `*.ibd`。
3. 梳理全库表清单，评估数据量，规划导出与导入窗口。

## 根因

- MySQL 5.5 默认 `innodb_file_per_table = 0`，所有 InnoDB 表共用 `ibdata1`。
- 关键点：**共享表空间模式下，单纯改配置不会对已有表生效**，必须先**重建数据文件**（移动/删除 `ibdata1` 与 redo log），再重新导入数据，新表才会写入独立 `.ibd` 文件。

## 修改

1. **全量导出数据**（含存储过程、函数、触发器、事件）：
   ```bash
   mysqldump -uroot -p --socket=/tmp/mysql.sock --single-transaction --master-data=2 \
     --all-databases -E -R --triggers --set-gtid-purged=off > /tmp/all-database.dump
   ```
2. 停库：
   ```bash
   mysqladmin -uroot -p shutdown
   # 或 /etc/init.d/mysqld stop
   ```
3. **重建数据文件**（重点）：将旧的共享表空间与 redo log 移走（先备份，勿直接删）：
   ```bash
   mv /var/lib/mysql/ibdata1    /tmp
   mv /var/lib/mysql/ib_logfile0 /tmp
   mv /var/lib/mysql/ib_logfile1 /tmp
   ```
4. 修改 `my.cnf`，开启独立表空间并规划新数据文件：
   ```ini
   innodb_file_per_table = 1
   innodb_data_home_dir   = /mysqldata/data
   innodb_data_file_path  = ibdata1:256M;ibdata2:128M:autoextend
   ```
5. 启动 MySQL：
   ```bash
   /etc/init.d/mysqld start
   # 或 mysqld_safe --defaults-file=/etc/my.cnf --user=mysql &
   ```
6. **重新导入数据**：
   ```bash
   mysql -uroot -p < /tmp/all-database.dump
   ```

## 复查

- 数据目录中出现按表区分的 `*.ibd` 独立文件。
- 抽样比对关键表行数与导入前一致。
- 确认服务重启后连接与依赖应用正常，无缺失对象（存储过程/触发器/事件）。

## 预防

- 新装 MySQL 将 `innodb_file_per_table = 1` 直接写入初始 `my.cnf`，避免以后再迁移。
- 表空间改造前务必先完整备份并把旧文件**移动到临时目录而非直接删除**，便于回滚。
- 大型库改表空间属高风险操作，应安排在低峰期并预留回滚窗口。