# MySQL主从结构从库重建


## 备份主库

本次采用MySQL数据库热备份工具xtrabackup进行。压缩备份用到的工具是pigz 进行压缩

- MySQL数据库版本：mysql-5.7.23
- xtrabackup备份工具版本：xtrabackup-2.4.12
- pigz压缩工具版本：pigz-2.3.4

### 非压缩备份

```shell
nohup $XTRABACKUP_HOME/bin/innobackupex --defaults-file=/etc/my.cfg --user=root --password=xxxxxx --socket=/etc/mysql/mysql.sock /usr/local/mysql > /usr/local/mysql/backup.log 2>&1 &
```

- `nohup &`  后台运行备份命令
- --defaults-file MySQL数据库配置文件路径
- --user 备份主库使用的用户 
- --password 备份主库用户的密码 
- --socket 本地备份套接字文件路径(mysql.sock)
- /usr/local/mysql 备份文件存放的路径
- /usr/local/mysql/backup.log 备份过程中的日志，可查看备份的执行情况
- 2>&1 将标准错误重定向到标准输出

### 压缩备份

这里的压缩备份直接将备份后的文件发送到目标主机上$remote_host

```shell
nohup $XTRABACKUP_HOME/bin/innobackupex --defaults-file=/etc/my.cfg --parallel=4 --user=root --password=xxxxxx --socket=/etc/mysql/mysql.sock --slave-info --no-timestamp --stream=tar /usr/local/mysql 2> /usr/local/mysql/backup.log | /iddbs/software/pigz-2.3.4/pigz -6 -p 4 | ssh $remote_host"cat - > /iddbs/tmp/backup.tar.gz" &
```

- nohup & 后台运行备份命令
- --defaults-file MySQL数据库配置文件路径
- --parallel 指定并行备份的线程数量
- --user 备份主库使用的用户
- --password 备份主库用户的密码
- --socket 本地备份套接字文件路径(mysql.sock)
- --slave-info 该参数会在备份目录下生成xtrabackup_slave_info文件，文件记录主库的binlog日志位置点。在进行数据库恢复，搭建多从库时都需要这个文件。
- --stream=tar 备份是采用的压缩方式
- /usr/local/mysql 备份文件存放的路径
- /usr/local/mysql/backup.log 备份过程中的日志，可查看备份的执行情况
- pigz -6 -p 4 -6表示压缩级别，-p表示压缩核心数
- 2>&1 将标准错误重定向到标准输出

## 停止从库MySQL服务

1.  先停止slave
```shell
stop slave;
```
2.  停止MySQL服务
```shell
$MYSQL_HOME/bin/mysqlamdin --socket=/etc/mysql/mysql.sock -uroot -pxxxxxx shutdown
```

## 恢复数据

恢复数据的时候，必须保证mysql数据库配置文件中datadir参数值存在且目录下为空，在进行数据恢复过程中，xtrabackup会自动将数据恢复到datadir参数值所在的目录下。

启动数据库的时候，必须保证server-id唯一。

### 回滚日志

将主库备份好的压缩文件进行解压后执行：

```shell
nohup $XTRABACKUP_HOME/bin/innobackupex --defaults-file=/etc/my.cfg --use-memory=8G --apply-log '压缩文件' >/tmp/apply3306.log 2>&1 &
```

### 数据恢复

```shell
nohup $XTRABACKUP_HOME/bin/innobackupex --defaults-file=/etc/my.cfg --use-memory=8G --copy-back '压缩文件' >/tmp/apply3306.log 2>&1 &
```

## 重做从库

启动数据库

```shell
$MYSQL_HOME/bin/mysqld_safe --defaults-file=/etc/my.cfg &
```

查看gtid
```shell
cat /usr/local/mysql/backup/xtrabackup_binlog_info
```
- /usr/local/mysql/backup 为压缩文件解压后的路径，可根据实际情况更改
- 在执行set global gtid_purged之前，必须执行reset master、stop slave、reset slave，否则会报错。

从库上执行：
```shell
reset master;

stop slave;

reset slave;

set global gtid_purged='71258aad-14da-11ea-8ee1-00163e0ed0aa:1-339,78599057-81e9-11e9-b25c-98039b06dd98:1-17291884,78a44d87-81e9-11e9-ae3b-98039b159cca:1-38826332,9ab6edc4-daeb-11ea-b68b-fa163e64adef:1-5,d6b50dc4-7fd2-11ea-8a35-fa163e4996e3:1-481427697';

CHANGE MASTER TO MASTER_HOST='主库IP',MASTER_PORT=mysql服务端口,MASTER_USER='主从复制账号',MASTER_PASSWORD='用户密码',MASTER_AUTO_POSITION=1;

start slave

show slave status\G
```
