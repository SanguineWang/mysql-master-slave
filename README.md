## Mysql主从集群搭建

https://juejin.cn/post/6844903615044255757

### 创建 MySQL 容器

#### 项目结构

```
mysql 
├── docker-compose.yml
├── master
│   ├── data
│   └── my.cnf
└── slave
    ├── data
    └── my.cnf
```

#### 准备配置文件

master

```yaml
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
## 设置server_id，一般设置为IP，注意要唯一
server_id=100
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，可以随便取，最好有含义（关键就是这里了）
log-bin=replicas-mysql-bin
## 为每个session分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
```

slave

```yaml
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
## 设置server_id，一般设置为IP，注意要唯一
server_id=101
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=replicas-mysql-slave1-bin
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
## relay_log配置中继日志
relay_log=replicas-mysql-relay-bin
## log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates=1
## 防止改变数据(除了特殊的线程)
read_only=1
```

#### docker-compose.yml

```yaml
version: '3.7'

services:

  mysql-master:
    image: mysql
    # 容器命名
    container_name: mysql-master
    # 环境变量，mysql要求提供，根据此密码可以进入容器mysql中
    environment:
      TZ: Asia/Shanghai
      MYSQL_DATABASE: new_ink_db
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - ./master/data/:/var/lib/mysql/
      - ./master/my.cnf:/etc/mysql/my.cnf
    ports:
      - "3306:3306"
    networks:
      - db-net

  mysql-slave:
    image: mysql
    # 容器命名为从数据库
    container_name: mysql-slave
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: new_ink_db
    volumes:
      - ./master/data/:/var/lib/mysql/
      - ./master/my.cnf:/etc/mysql/my.cnf
    # 将容器端口映射到宿主机端口，可以声明也可以不声明，一般情况下不会将后端端口直接暴露出去
    ports:
      - "3307:3306"
    networks:
      - db-net

networks:
  db-net:
    driver: bridge
```

#### 上传到服务器

```bash
$ scp -r mysql  root@{serverip}:/home/service/
```

#### 构建容器

```shell
$ docker-compose up -d
```

主数据库状态

```bash
mysql> show master status
    -> ;
+---------------------------+----------+--------------+------------------+-------------------+
| File                      | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------------------+----------+--------------+------------------+-------------------+
| replicas-mysql-bin.000004 |      704 |              | mysql            |                   |
+---------------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

#### 创建用于主从同步的账号：

```sql
create user 'slave'@'%' indentified by '123456';

grant file,select,replication slave on *.* to 'slave'@'%';
```

从数据库通过账号连接主库

```sql
CHANGE MASTER TO
    MASTER_HOST='mysql-master',
    MASTER_USER='slave',
    MASTER_PASSWORD='123456',
    MASTER_LOG_FILE='replicas-mysql-bin.000004',
    MASTER_LOG_POS=704;
```

重启slave

```
stop slave；
start slave；
```

从库状态

```bash
show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: *.*.*.*(主库地址)
                  Master_User: ****（创建的账号）
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: replicas-mysql-bin.000004
          Read_Master_Log_Pos: 704
               Relay_Log_File: replicas-mysql-relay-bin.000002
                Relay_Log_Pos: 881
        Relay_Master_Log_File: replicas-mysql-bin.000004
             Slave_IO_Running: Yes （这两行）
            Slave_SQL_Running: Yes （都是yes才算启动主从）
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 704
              Relay_Log_Space: 1099
```

