# 配置说明
## 主库
```
server-id=1 # 服务器 id, 必须每台唯一
log-bin=mysql-bin # 指定 binlog 文件名
binlog-do-db=demo # 指定要同步的数据库
```
## 从库
```
server-id=2 # 服务器id, 不能与主库相同
replicate_wild_do_table=demo.% # 指定要同步的数据库
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
