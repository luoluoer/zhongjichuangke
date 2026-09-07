# <center>Ubuntu MySQL 数据库运维手册</center>

> **适用版本**：MySQL 5.7 / 8.0 on Ubuntu 18.04+
> **适用项目**：鹿角隧道、鹿角隧道东延伸段、六纵线（六横线至三环高速段）、宝山嘉陵江大桥智慧管理智能监控指挥中心项目
> **维护单位**：中机创新科技发展（重庆）有限公司


## 一、手册说明

### 1.1 目的

规范 Ubuntu 环境下 MySQL 数据库的部署、配置、日常运维和故障处理操作，保障智慧管理平台核心数据的安全性、稳定性和可恢复性。

### 1.2 适用范围

本手册适用于本项目所使用的所有 MySQL 数据库实例（含开发、测试、生产环境）的运维工作。

### 1.3 配置文件路径

| 文件       | 路径                                 |
| ---------- | ------------------------------------ |
| 主配置文件 | `/etc/mysql/my.cnf`                  |
| 服务端配置 | `/etc/mysql/mysql.conf.d/mysqld.cnf` |
| 错误日志   | `/var/log/mysql/error.log`           |
| 慢查询日志 | `/var/log/mysql/mysql-slow.log`      |
| 数据目录   | `/var/lib/mysql/`                    |
| 服务管理   | `systemctl`                          |


## 二、安装与初始化

### 2.1 系统环境准备

```bash
# 更新系统包列表并升级现有软件包
sudo apt update && sudo apt upgrade -y

# 确认系统版本
lsb_release -a
```

### 2.2 安装 MySQL

```bash
# 安装 MySQL 服务器
sudo apt install mysql-server -y

# 安装完成后检查服务状态
sudo systemctl status mysql
```

### 2.3 服务管理

```bash
# 启动 MySQL 服务
sudo systemctl start mysql

# 设置开机自启
sudo systemctl enable mysql

# 停止 MySQL 服务
sudo systemctl stop mysql

# 重启 MySQL 服务
sudo systemctl restart mysql

# 查看服务状态
sudo systemctl status mysql
```

### 2.4 安全初始化（关键步骤）

安装完成后必须立即执行安全脚本完成基础加固：

```bash
sudo mysql_secure_installation
```

按提示依次完成以下设置：
1. 设置 root 强密码（需包含大小写字母、数字、特殊字符）
2. 移除匿名用户（Remove anonymous users? → Y）
3. 禁止远程 root 登录（Disallow root login remotely? → Y）
4. 删除测试数据库（Remove test database and access to it? → Y）
5. 重载权限表（Reload privilege tables now? → Y）

### 2.5 登录验证

```bash
# 使用 root 登录（首次需用 sudo）
sudo mysql

# 或使用设置的密码登录
mysql -u root -p
```


## 三、配置与优化

### 3.1 基础配置

编辑配置文件 `/etc/mysql/mysql.conf.d/mysqld.cnf`：

```ini
[mysqld]

# === 基础设置 ===
# 绑定地址（0.0.0.0 允许所有IP访问，127.0.0.1 仅本地）
bind-address = 0.0.0.0

# 端口
port = 3306

# 字符集（必须使用 utf8mb4 以支持完整 Unicode）
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# 最大连接数（根据内存和应用调整，通常 200-1000）
max_connections = 500
```

### 3.2 性能优化参数（以 16GB 内存服务器为例）

```ini
[mysqld]

# === InnoDB 缓冲池（建议设置为物理内存的 50%-80%）===
# 16GB 内存建议 8-12GB
innodb_buffer_pool_size = 8G

# === InnoDB 日志 ===
# 建议 256M-1G，大事务/高写入可适当增大
innodb_log_file_size = 512M

# === 事务提交刷盘策略 ===
# 1=强一致（默认），2=更高吞吐（可接受最多1秒数据丢失）
innodb_flush_log_at_trx_commit = 1

# === I/O 优化 ===
# O_DIRECT 绕过 OS 页缓存，减少双缓冲
innodb_flush_method = O_DIRECT

# SSD/NVMe 建议 2000/4000 或更高
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# === 慢查询日志 ===
# 开启慢查询日志，记录执行时间超过 1 秒的 SQL
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

> **修改配置后必须重启服务生效**：`sudo systemctl restart mysql`

### 3.3 配置调优辅助工具

```bash
# 安装 MySQLTuner（分析配置并给出优化建议）
sudo apt-get install mysqltuner -y
mysqltuner --user root --ask-pass

# 或使用官方 Performance Schema 分析
mysql -u root -p -e "SELECT * FROM sys.metrics ORDER BY value DESC LIMIT 20;"
```


## 四、用户与权限管理

### 4.1 创建业务数据库和专用账号（安全最佳实践）

**禁止直接使用 root 账号连接业务应用**，必须创建最小权限的专用账号。

```sql
-- 创建数据库（指定 utf8mb4 字符集）
CREATE DATABASE appdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 创建业务账号（限制来源网段）
CREATE USER 'appuser'@'10.0.0.%' IDENTIFIED BY 'StrongPass!';

-- 授予最小必要权限（仅 SELECT/INSERT/UPDATE/DELETE）
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'appuser'@'10.0.0.%';

-- 重载权限
FLUSH PRIVILEGES;
```

### 4.2 用户权限查看与回收

```sql
-- 查看所有用户及来源主机
SELECT User, Host FROM mysql.user;

-- 查看特定用户权限
SHOW GRANTS FOR 'appuser'@'10.0.0.%';

-- 回收权限
REVOKE DELETE ON appdb.* FROM 'appuser'@'10.0.0.%';

-- 删除用户
DROP USER 'appuser'@'10.0.0.%';
```

### 4.3 允许远程访问（如确需）

```bash
# 1. 修改配置文件 bind-address = 0.0.0.0
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf

# 2. 重启服务
sudo systemctl restart mysql

# 3. 配置防火墙放行 3306 端口
sudo ufw allow from any to any port 3306 proto tcp

# 4. 创建远程访问账号（限制来源IP，不建议使用 %）
CREATE USER 'appuser'@'192.168.1.%' IDENTIFIED BY 'StrongPass!';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'192.168.1.%';
FLUSH PRIVILEGES;
```


## 五、备份与恢复

### 5.1 逻辑备份（mysqldump）

#### 5.1.1 备份单个数据库

```bash
mysqldump -u root -p mydatabase > /backup/mydatabase_$(date +%Y%m%d).sql
```

#### 5.1.2 备份所有数据库

```bash
mysqldump -u root -p --all-databases --single-transaction --routines --triggers > /backup/full_backup_$(date +%Y%m%d).sql
```

#### 5.1.3 压缩备份

```bash
mysqldump -u root -p --single-transaction --routines --triggers --databases mydatabase | gzip > /backup/mydatabase_$(date +%Y%m%d).sql.gz
```

#### 5.1.4 自动备份脚本（cron 定时任务）

```bash
# 创建备份脚本 /usr/local/bin/mysql_backup.sh
#!/bin/bash
BACKUP_DIR=/backup/mysql
DATE=$(date +%Y%m%d_%H%M%S)
MYSQL_PWD="your_password"

# 全量备份
mysqldump -u root -p"$MYSQL_PWD" --all-databases --single-transaction --routines --triggers | gzip > $BACKUP_DIR/full_$DATE.sql.gz

# 保留最近 7 天备份
find $BACKUP_DIR -name "full_*.sql.gz" -mtime +7 -delete
```

```bash
# 添加 cron 任务（每日凌晨 2:00 执行）
sudo crontab -e
# 添加以下行
0 2 * * * /usr/local/bin/mysql_backup.sh
```

### 5.2 数据恢复

#### 5.2.1 恢复 SQL 文件

```bash
# 先创建目标数据库（如不存在）
mysql -u root -p -e "CREATE DATABASE mydatabase;"

# 恢复数据
mysql -u root -p mydatabase < /backup/mydatabase_backup.sql
```

#### 5.2.2 恢复压缩备份

```bash
gunzip < /backup/mydatabase_20260101.sql.gz | mysql -u root -p mydatabase
```

### 5.3 备份验证

```bash
# 验证备份文件完整性（查看前50行）
zcat /backup/mysql_20260101.sql.gz | head -n 50

# 定期抽样导入验证（建议每月执行一次恢复演练）
```


## 六、监控与巡检

### 6.1 命令行快速监控

#### 6.1.1 服务状态检查

```bash
sudo systemctl status mysql
```

#### 6.1.2 mysqladmin 状态查看

```bash
mysqladmin -u root -p status
```

输出关键指标：运行时间（Uptime）、线程数（Threads）、查询数（Queries）、连接数（Connections）等。

#### 6.1.3 登录 MySQL 查看运行状态

```sql
-- 查看服务器状态变量
SHOW STATUS LIKE 'Threads_connected';    -- 当前连接数
SHOW STATUS LIKE 'Queries';              -- 总查询数
SHOW STATUS LIKE 'Innodb_buffer_pool_%'; -- 缓冲池使用情况

-- 查看当前正在执行的查询
SHOW PROCESSLIST;

-- 查看服务器整体状态（快捷键）
\s
```

### 6.2 性能诊断工具

```bash
# MySQLTuner - 生成性能分析报告
mysqltuner --user root --ask-pass

# Percona Toolkit - 慢查询分析
pt-query-digest /var/log/mysql/mysql-slow.log
```

### 6.3 系统资源监控

```bash
# 查看进程资源占用（按 M 键按内存排序，P 键按 CPU 排序）
top
htop

# 查看磁盘 I/O
iostat -x 1

# 查看系统整体资源
vmstat 1
```

### 6.4 第三方监控方案（推荐）

| 方案                                         | 说明                                         |
| -------------------------------------------- | -------------------------------------------- |
| **Prometheus + Grafana + mysqld_exporter**   | 采集 QPS/TPS/连接数/慢查询等指标，可视化展示 |
| **PMM（Percona Monitoring and Management）** | 开箱即用，含 QAN 慢查询分析                  |
| **mytop**                                    | 命令行实时监控，类似 top                     |

### 6.5 日志管理

```bash
# 查看错误日志最近50行
sudo tail -n 50 /var/log/mysql/error.log

# 实时跟踪错误日志
sudo tail -f /var/log/mysql/error.log

# 查看慢查询日志
sudo tail -n 50 /var/log/mysql/mysql-slow.log

# 检查日志文件大小
ls -lh /var/log/mysql/
```

### 6.6 巡检记录模板

| 检查项       | 命令/方式                                     | 正常标准                 | 结果 |
| ------------ | --------------------------------------------- | ------------------------ | ---- |
| 服务状态     | `systemctl status mysql`                      | active (running)         |      |
| 运行时间     | `mysqladmin status`                           | 持续运行                 |      |
| 当前连接数   | `SHOW STATUS LIKE 'Threads_connected'`        | < max_connections 的 80% |      |
| 缓冲池命中率 | `SHOW STATUS LIKE 'Innodb_buffer_pool_read%'` | > 95%                    |      |
| 磁盘空间     | `df -h /var/lib/mysql`                        | 剩余 > 20%               |      |
| 错误日志     | `tail -n 50 /var/log/mysql/error.log`         | 无 ERROR/SEVERE          |      |
| 慢查询       | `tail -n 20 /var/log/mysql/mysql-slow.log`    | 无 >5s 的查询            |      |
| 最近备份     | 检查备份目录                                  | 24小时内                 |      |
| **巡检人**   |                                               | **巡检日期**             |      |


## 七、常见故障排查

### 7.1 服务无法启动

```bash
# 1. 查看服务状态获取初步信息
sudo systemctl status mysql

# 2. 查看错误日志定位具体原因
sudo tail -n 50 /var/log/mysql/error.log

# 3. 常见原因及处理
# - 配置文件语法错误 → 检查 /etc/mysql/mysql.conf.d/mysqld.cnf
# - 端口冲突（3306被占用）→ sudo netstat -tulnp | grep 3306
# - 磁盘空间不足 → df -h
```

### 7.2 无法远程连接

```bash
# 1. 确认服务已启动
sudo systemctl status mysql

# 2. 检查防火墙是否放行 3306 端口
sudo ufw status

# 3. 检查 bind-address 配置（不能是 127.0.0.1）
cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep bind-address

# 4. 检查用户权限是否允许远程访问
mysql -u root -p -e "SELECT User, Host FROM mysql.user;"
```

### 7.3 密码错误/忘记 root 密码（ERROR 1045）

```bash
# 1. 停止 MySQL 服务
sudo systemctl stop mysql

# 2. 跳过权限验证启动
sudo mysqld_safe --skip-grant-tables &

# 3. 无密码登录
mysql -u root

# 4. 重置密码（MySQL 8.0+）
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '新密码';
FLUSH PRIVILEGES;

# 5. 退出并正常重启
exit
sudo systemctl restart mysql
```

### 7.4 表损坏修复

```bash
# 检查并修复所有数据库的表
sudo mysqlcheck --all-databases --auto-repair

# 修复特定数据库
sudo mysqlcheck -u root -p 数据库名 --auto-repair
```

### 7.5 磁盘空间不足

```bash
# 1. 查看磁盘使用情况
df -h

# 2. 清理无用的软件包
sudo apt autoremove

# 3. 清理 APT 缓存
sudo apt clean

# 4. 清理过期备份文件
find /backup -name "*.sql.gz" -mtime +30 -delete

# 5. 谨慎清理日志（先备份）
sudo truncate -s 0 /var/log/mysql/error.log
```


## 八、附录

### 8.1 常用命令速查

| 用途         | 命令                                            |
| ------------ | ----------------------------------------------- |
| 安装 MySQL   | `sudo apt install mysql-server -y`              |
| 启动服务     | `sudo systemctl start mysql`                    |
| 停止服务     | `sudo systemctl stop mysql`                     |
| 重启服务     | `sudo systemctl restart mysql`                  |
| 查看状态     | `sudo systemctl status mysql`                   |
| 安全初始化   | `sudo mysql_secure_installation`                |
| 登录 MySQL   | `mysql -u root -p`                              |
| 备份数据库   | `mysqldump -u root -p 库名 > 文件.sql`          |
| 恢复数据库   | `mysql -u root -p 库名 < 文件.sql`              |
| 查看错误日志 | `sudo tail -n 50 /var/log/mysql/error.log`      |
| 查看慢查询   | `sudo tail -n 50 /var/log/mysql/mysql-slow.log` |
| 查看进程列表 | `mysql -u root -p -e "SHOW PROCESSLIST;"`       |
| 查看连接数   | `mysqladmin -u root -p status`                  |
| 性能分析     | `mysqltuner --user root --ask-pass`             |

