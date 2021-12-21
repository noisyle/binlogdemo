# 配置说明
## 主库
```
server-id=1 # 服务器 id, 每台服务器必须唯一
log-bin=mysql-bin # 设置 bin-log 名称
binlog-do-db=demo # 设置要同步的数据库名称
```
## 从库
```
server-id=2 # 服务器 id, 不能与主库相同
replicate-wild-do-table=demo.% # 设置要同步的数据库名称
relay-log=mysql-relay-bin # 设置 relay-log 名称
```
# 启动服务
```
docker-compose up -d
```
# 配置主库
## 使用 root 登录主库
```
docker-compose exec demo-mysql-master mysql -uroot -p demo
```
## 创建用户用于同步
```
CREATE USER 'db_sync'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'db_sync'@'%';
FLUSH PRIVILEGES;
```
## 记录日志文件和位点
```
show master status;
```
记录输出信息中的日志文件和位点信息，如下图。

![Image](Screenshot_1.png)
# 配置从库
## 使用 root 登录从库
```
docker-compose exec demo-mysql-slave mysql -uroot -p demo
```
## 设置同步
```
stop slave;
CHANGE MASTER TO master_host='demo-mysql-master',master_user='db_sync',master_password='password',master_log_file='mysql-bin.000003',master_log_pos=844;
start slave;
show slave status\G
```
命令中的日志文件和位点信息需要按照配置主库时记录的值进行填写。配置成功后输出如下信息。

![Image](Screenshot_2.png)
# 备注
- 本例中使用显式设置位点的方式开始同步，需要在开始同步前**保证主从库状态相同**。
- 同步数据库(表)的相关配置有:
  - `binlog-do-db`
  - `binlog-ignore-db`
  - `replicate-do-db`
  - `replicate-ignore-db`
  - `replicate-wild-do-table`
  - `replicate-wild-ignore-table`
- 从库使用 `replicate-wild-do-table=demo.%` 方式进行配置，可以保证跨库执行的操作也能被同步。
  ```
  // my.cnf for slave
  replicate-do-db=db2
  replicate-ignore-db=db1
  ```
  ```
  use db1;
  update db2.table1 set ......
  ```
  使用 `replicate-do-db`, `replicate-ignore-db` 进行配置时，上面这样跨数据库执行的操作不会按照配置的预期被同步或忽略。[参考: Why MySQL’s binlog-do-db option is dangerous.](https://www.percona.com/blog/2009/05/14/why-mysqls-binlog-do-db-option-is-dangerous/)
- 从库配置中如不设置 `relay-log=mysql-relay-bin` ，执行 `CHANGE MASTER` 时日志中会出现如下警告。
  > Neither --relay-log nor --relay-log-index were used; so replication may break when this MySQL server acts as a slave and has his hostname changed!!

  当前服务器作为从库，由于未设置 relay-log，MySQL 将使用主机名作为 ralay-log 名称，如果此后主机名发生变化，将导致找不到 ralay-log 从而断开同步。显式设置 relay-log 后，警告消除。
