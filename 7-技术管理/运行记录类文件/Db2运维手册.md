# DB2数据库日常巡检手册

> **适用版本**：IBM DB2 11.5 for Linux/UNIX/Windows
> **适用项目**：鹿角隧道、鹿角隧道东延伸段、六纵线（六横线至三环高速段）、宝山嘉陵江大桥智慧管理智能监控指挥中心项目
> **维护单位**：中机创新科技发展（重庆）有限公司


## 一、手册说明

### 1.1 目的

规范DB2数据库日常巡检操作，及时发现和解决数据库运行中的问题，预防潜在故障，保障智慧管理平台核心数据的安全性和业务连续性。

### 1.2 适用范围

本手册适用于本项目所使用的所有DB2数据库实例的日常巡检与维护工作。

### 1.3 巡检频率

| 频率     | 说明                 |
| -------- | -------------------- |
| 每日巡检 | 每个工作日开盘前完成 |
| 每周巡检 | 每周一上午完成       |
| 每月巡检 | 每月第一个工作日完成 |


## 二、每日巡检

### 2.1 实例与数据库状态检查

**检查目标**：确认DB2实例及数据库运行正常。

**操作步骤**：

#### 2.1.1 查看实例列表与当前实例

```bash
db2ilist                    # 列出所有DB2实例
db2 get instance            # 确认当前实例
```

#### 2.1.2 检查DB2系统进程

```bash
ps -ef | grep db2           # 检查DB2相关进程是否存在
ps -ef | grep db2sysc       # 确认db2sysc核心进程运行中
```

若进程不存在，执行 `db2start` 启动实例。

#### 2.1.3 检查管理服务器进程

```bash
ps -ef | grep dasusr1       # 确认DAS管理服务正常运行
```

若未启动，执行 `db2adminstart` 启动。

#### 2.1.4 测试数据库连接

```bash
db2 list active databases   # 查看当前活动数据库列表
db2 connect to <数据库名>    # 尝试连接数据库
```

连接成功即表示数据库服务正常。

#### 2.1.5 检查数据库状态

```bash
db2 get db cfg for <数据库名> | grep -E "Database status|Backup pending|Restore pending"
```

关注是否存在“备份挂起”或“恢复挂起”状态。

### 2.2 表空间检查

**检查目标**：确认表空间状态正常，使用率未超阈值。

**操作步骤**：

#### 2.2.1 查看表空间状态

```bash
db2 list tablespaces show detail        # 查看所有表空间详细信息
```

**正常状态**：单分区表空间状态显示为 `0x0000`。

#### 2.2.2 查看表空间使用率

```bash
db2 "select TBSP_NAME, TBSP_TYPE, TBSP_USABLE_PAGES, TBSP_USED_PAGES, 
        decimal(float(TBSP_USED_PAGES)/float(TBSP_USABLE_PAGES)*100,5,2) as USAGE_PCT 
 from sysibmadm.TBSP_UTILIZATION"
```

**告警阈值**：
- 使用率 ≥ 80%：**关注**，评估扩容需求
- 使用率 ≥ 90%：**告警**，立即扩容或清理

#### 2.2.3 查看表空间详细信息

```bash
db2pd -db <数据库名> -tablespaces       # 快速查看表空间状态
```

### 2.3 存储空间检查

**检查目标**：确认数据库存储路径磁盘空间充足。

```bash
df -h /path/to/db2/storage              # 查看DB2存储路径磁盘空间
```

**告警阈值**：剩余空间 < 20% 时需及时扩容。

### 2.4 日志检查

**检查目标**：确认数据库日志状态正常，日志空间充足。

#### 2.4.1 查看日志配置

```bash
db2 get db cfg for <数据库名> | grep -i log
```

关注以下参数：
- `LOGPRIMARY`：主日志文件数
- `LOGSECOND`：辅助日志文件数
- `LOGFILSIZ`：每个日志文件大小（页数）
- `NEWLOGPATH`：日志存储路径

#### 2.4.2 查看活动日志状态

```bash
db2pd -db <数据库名> -logs              # 查看当前活动日志
```

检查是否存在日志使用率过高或日志目录满的情况。

### 2.5 错误日志检查

**检查目标**：及时发现数据库运行错误。

```bash
# 检查DB2诊断日志（diaglog）
tail -n 100 ~/sqllib/db2dump/db2diag.log | grep -i -E "error|failed|severe"

# 检查系统日志中的DB2相关记录
tail -n 100 /var/log/messages | grep -i db2
```

如有严重错误（SEVERE）或无法恢复的错误，需立即上报。

### 2.6 备份状态检查（每日）

**检查目标**：确认最近一次备份成功，确保数据可恢复。

```bash
db2 list history backup all for <数据库名> | head -20    # 查看最近备份记录
```

确认最近一次备份时间为24小时内，且状态为成功。


## 三、每周巡检

### 3.1 缓冲池命中率检查

**检查目标**：评估数据库缓存效率。

```bash
db2pd -db <数据库名> -bufferpools        # 查看缓冲池命中率
```

**正常标准**：缓冲池命中率应 **> 95%**。若低于此值，需考虑增加缓冲池大小或优化SQL。

### 3.2 锁与死锁监控

**检查目标**：发现锁等待和死锁问题。

#### 3.2.1 查看锁等待

```bash
db2 get snapshot for locks on <数据库名> | grep -i "lock waits"
```

#### 3.2.2 查看详细锁信息

```bash
db2pd -db <数据库名> -locks show detail
```

若锁等待数量持续增长，需排查是否存在长事务或未提交事务。

### 3.3 内存使用检查

**检查目标**：确认DB2内存分配合理，无内存泄漏。

```bash
db2mtrk -i -v                            # 监控实例内存使用情况
```

重点关注缓冲池、排序堆、包缓存等内存区域的使用量。

### 3.4 慢SQL检查

**检查目标**：识别执行效率低下的SQL语句。

```bash
db2 "SELECT * FROM SYSIBMADM.TOP_DYNAMIC_SQL 
     ORDER BY TOTAL_EXEC_TIME DESC 
     FETCH FIRST 10 ROWS ONLY"
```

分析TOP 10慢SQL，评估是否需要优化或添加索引。

### 3.5 系统资源检查

```bash
top -u db2inst1                          # 查看DB2进程CPU/内存占用
iostat -dx 2 5                           # 查看磁盘I/O情况
```

### 3.6 HADR状态检查（如已配置高可用）

**检查目标**：确认HADR主备复制状态正常。

```bash
db2pd -db <数据库名> -hadr              # 查看HADR角色、状态和同步延迟
```

**正常状态**：主库角色为 `PRIMARY`，备库角色为 `STANDBY`，状态为 `PEER` 或 `CONNECTED`。

### 3.7 安全审计检查

```bash
db2 "select * from syscat.dbauth"       # 查看数据库级权限
db2 "select * from syscat.tabauth"      # 查看表级权限
```

检查是否存在异常账号或权限变更。


## 四、每月巡检

### 4.1 RUNSTATS统计信息更新

**检查目标**：确保数据库统计信息准确，优化器能生成高效执行计划。

```bash
# 查看需要更新统计信息的表
db2 "SELECT TABSCHEMA, TABNAME, STATS_TIME 
     FROM SYSCAT.TABLES 
     WHERE STATS_TIME < CURRENT TIMESTAMP - 30 DAYS"

# 对指定表执行RUNSTATS
db2 RUNSTATS ON TABLE <表名> FOR INDEXES ALL
```

建议每月对变化频繁的表执行一次RUNSTATS。

### 4.2 REORG表重组

**检查目标**：回收碎片空间，提升数据访问效率。

```bash
# 查看需要重组的表
db2 "SELECT TABSCHEMA, TABNAME, REORG_PENDING, REORG_NEEDED 
     FROM SYSIBMADM.ADMINTABINFO 
     WHERE REORG_PENDING = 'Y' OR REORG_NEEDED = 'Y'"
```

对有重组需求的表执行：
```bash
db2 REORG TABLE <表名>
```

### 4.3 完整备份验证

**检查目标**：验证备份文件的完整性和可恢复性。

```bash
# 查看所有备份历史
db2 list history backup all for <数据库名>

# 验证最新备份文件有效性
db2ckbkp -h <备份文件路径>
```

确保备份文件可读且无损坏。

### 4.4 数据库配置参数审查

```bash
db2 get db cfg for <数据库名>           # 查看数据库配置
db2 get dbm cfg                         # 查看实例配置
```

对比基线配置，审查是否有异常变更。

### 4.5 数据库增长趋势分析

```bash
db2 "SELECT TBSP_NAME, TBSP_USABLE_PAGES, TBSP_USED_PAGES,
        decimal(float(TBSP_USED_PAGES)/float(TBSP_USABLE_PAGES)*100,5,2) as USAGE_PCT 
     FROM SYSIBMADM.TBSP_UTILIZATION"
```

记录各表空间使用量，评估月度增长趋势，预判扩容需求。


## 五、巡检记录模板

### 5.1 每日巡检记录表

| 检查项         | 检查结果 | 状态（正常/异常） | 备注 |
| -------------- | -------- | ----------------- | ---- |
| 实例进程状态   |          |                   |      |
| 数据库连接状态 |          |                   |      |
| 表空间状态     |          |                   |      |
| 表空间使用率   |          |                   |      |
| 磁盘剩余空间   |          |                   |      |
| 日志状态       |          |                   |      |
| 错误日志       |          |                   |      |
| 最近备份状态   |          |                   |      |
| **巡检人**     |          | **巡检日期**      |      |

### 5.2 异常处理记录

| 发现时间 | 异常描述 | 处理措施 | 处理结果 | 处理人 |
| -------- | -------- | -------- | -------- | ------ |
|          |          |          |          |        |


## 六、常见问题快速处理

### 6.1 表空间已满

**现象**：插入/更新数据时报 `SQL0289N` 错误。

**处理**：
```bash
# 查看表空间使用情况
db2 list tablespaces show detail

# 扩容表空间（自动存储）
db2 ALTER TABLESPACE <表空间名> EXTEND (ALL <新增页数>)

# 或添加新容器
db2 ALTER TABLESPACE <表空间名> ADD (FILE '<路径>' <大小>)
```

### 6.2 连接数超限

**现象**：新连接报 `SQL1040N` 错误。

**处理**：
```bash
# 查看当前连接数
db2 list applications

# 强制终止异常连接
db2 force application (<应用句柄>)

# 或调整最大连接数
db2 update db cfg for <数据库名> using MAXAPPLS <新数值>
```

### 6.3 事务日志已满

**现象**：报 `SQL0964C` 错误。

**处理**：
```bash
# 增加辅助日志文件数
db2 update db cfg for <数据库名> using LOGSECOND <新数值>

# 或提交/回滚未完成的长事务后，执行数据库备份以释放日志空间
```


## 七、附录

### 7.1 常用DB2巡检命令速查

| 用途             | 命令                                        |
| ---------------- | ------------------------------------------- |
| 查看实例列表     | `db2ilist`                                  |
| 查看当前实例     | `db2 get instance`                          |
| 启动实例         | `db2start`                                  |
| 连接数据库       | `db2 connect to <DB_NAME>`                  |
| 查看表空间       | `db2 list tablespaces show detail`          |
| 查看表空间使用率 | `db2pd -db <DB_NAME> -tablespaces`          |
| 查看缓冲池       | `db2pd -db <DB_NAME> -bufferpools`          |
| 查看锁信息       | `db2pd -db <DB_NAME> -locks`                |
| 查看日志         | `db2pd -db <DB_NAME> -logs`                 |
| 查看HADR         | `db2pd -db <DB_NAME> -hadr`                 |
| 查看内存         | `db2mtrk -i -v`                             |
| 查看备份历史     | `db2 list history backup all for <DB_NAME>` |
| 验证备份         | `db2ckbkp -h <备份文件>`                    |
