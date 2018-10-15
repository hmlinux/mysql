# MySQL双主多从集群配置文档

MySQL主主复制、主从复制

## MySQL2主1从集群配置

### 1. 服务器环境

|IP            |`Server_id`|MySQL Version  |Role      |`MySQL_HOME`    |`Datadir`         |Alias
|:-------------|:----------|:--------------|:---------|:---------------|:-----------------|:----
|10.19.53.222  |1          |MySQL 5.7.23   |`Master`  |/opt/app/mysql  |/data/mysql/mysql | M1
|10.19.91.16   |2          |MySQL 5.7.23   |`Master`  |/opt/app/mysql  |/data/mysql/mysql | M2
|10.19.36.184  |1101       |MySQL 5.7.23   |`Slave`   |/opt/app/mysql  |/data/mysql/mysql | S1

### 2. 编译安装MySQL

下载MySQL5.7源码包
```
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-boost-5.7.23.tar.gz
```

安装依赖包
```bash
yum install -y gcc gcc-c++ make automake zlib-devel ncurses ncurses-devel bison openssl-devel
```

安装Cmake
```
wget https://cmake.org/files/v3.12/cmake-3.12.3.tar.gz

tar -zxf cmake-3.12.3.tar.gz
cd cmake-3.12.3
./bootstrap --prefix=/usr/local/cmake
gmake -j4 && gmake install
```

#### 编译安装MySQL5.7
```
mkdir /opt/app/mysql
mkdir /data/mysql/mysql -p

groupadd mysql
useradd -r -g mysql -s /sbin/nologin mysql -M
chown -R mysql:mysql /data/mysql

tar -zxf mysql-boost-5.7.23.tar.gz
cd mysql-5.7.23/

/usr/local/cmake/bin/cmake . \
-DCMAKE_INSTALL_PREFIX=/opt/app/mysql \
-DMYSQL_DATADIR=/data/mysql/mysql \
-DSYSCONFDIR=/etc \
-DMYSQL_TCP_PORT=3306 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITH_ARCHIVE_STORAGE_ENGINE=1 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DDEFAULT_CHARSET=utf8 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_EXTRA_CHARSETS:STRING=all \
-DWITH_SSL=yes \
-DWITH_EMBEDDED_SERVER=1 \
-DWITH_DEBUG=0 \
-DWITH_BOOST=boost

gmake -j4 && gmake install

mkdir /data/mysql/binlog
chown -R mysql:mysql /data/mysql/binlog

mkdir /opt/app/mysql/logs
chown -R mysql:mysql /opt/app/mysql/logs

cd /opt/app/mysql/
cp support-files/mysql.server /etc/init.d/mysqld
chmod 755 /etc/init.d/mysqld
chkconfig mysqld on
systemctl daemon-reload

cat >> /etc/profile.d/mysql.sh << EOF
export PATH=$PATH:/opt/app/mysql/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/app/mysql/lib
EOF
source /etc/profile.d/mysql.sh
```

#### 主1节点`my.cnf`二进制日志配置
```
log_bin = /data/mysql/binlog/mysql-bin
relay_log = /data/mysql/binlog/mysql-relay-bin
server_id = 1
log_slave_updates = 1
relay_log_purge = 1
binlog_cache_size = 2M
binlog_format = MIXED
sync_binlog = 1
expire_logs_days = 30
```

#### 主2节点`my.cnf`二进制日志配置
```
log_bin = /data/mysql/binlog/mysql-bin
relay_log = /data/mysql/binlog/mysql-relay-bin
server_id = 2
log_slave_updates = 1
relay_log_purge = 1
binlog_cache_size = 2M
binlog_format = MIXED
sync_binlog = 1
expire_logs_days = 30
```

#### 从`slave`节点`my.cnf`二进制日志配置
```
relay_log = /data/mysql/binlog/mysql-relay-bin
server_id = 1101
read_only = 1
```

安装数据库
```bash
mysqld --initialize --user=mysql --basedir=/opt/app/mysql --datadir=/data/mysql/mysql

mysql_ssl_rsa_setup --datadir=/data/mysql/mysql
chown -R mysql:mysql /data/mysql
```

启动数据库
```
service mysqld start
/sbin/chkconfig mysqld on
```

数据库初始化配置，配置数据库root密码
```
mysql_secure_installation
```

### 3. 创建REPLICATION用户(主上创建)

In 10.19.53.222 (M1)
```
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '741616710#Repl' ;
FLUSH PRIVILEGES ;
```

In 10.19.91.16 (M2)
```
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '741616710#Repl' ;
FLUSH PRIVILEGES ;
```

### 4. 启动数据库二进制日志复制(从上配置)

In 10.19.53.222 (M1)
```
STOP SLAVE;
CHANGE MASTER TO master_host='10.19.91.16', master_user='repl', master_password='741616710#Repl' ;
START SLAVE;
```

In 10.19.91.16 (M2)
```
STOP SLAVE;
CHANGE MASTER TO master_host='10.19.53.222', master_user='repl', master_password='741616710#Repl' ;
START SLAVE;
```

In 10.19.108.107 (S1)
```
STOP SLAVE;
CHANGE MASTER TO master_host='10.19.91.16', master_user='repl', master_password='741616710#Repl' ;
START SLAVE;
```

### 5. 查看主从状态

M1
```
mysql> show variables like 'hostname';
+---------------+--------------+
| Variable_name | Value        |
+---------------+--------------+
| hostname      | 10-19-53-222 |
+---------------+--------------+
1 row in set (0.00 sec)

mysql> show slave hosts;
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|         2 |      | 3306 |         1 | 6e008dcb-cc60-11e8-afe1-525400a84742 |
+-----------+------+------+-----------+--------------------------------------+
1 row in set (0.00 sec)
```

M2
```
mysql> show variables like 'hostname';
+---------------+-------------+
| Variable_name | Value       |
+---------------+-------------+
| hostname      | 10-19-91-16 |
+---------------+-------------+
1 row in set (0.01 sec)

mysql> show slave hosts;
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|      1101 |      | 3306 |         2 | 9e6e47a5-cc63-11e8-910a-525400ec5030 |
|         1 |      | 3306 |         2 | 551d0f00-cc5e-11e8-8284-525400202abb |
+-----------+------+------+-----------+--------------------------------------+
2 rows in set (0.00 sec)
```

S1
```
mysql> show variables like 'hostname';
+---------------+--------------+
| Variable_name | Value        |
+---------------+--------------+
| hostname      | 10-19-36-184 |
+---------------+--------------+
1 row in set (0.01 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.19.91.16
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 589
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 802
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

### 创建普通用户

```
grant all on *.* to huangming@'10.19.%' identified by 'password';
flush privileges;
```
