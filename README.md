# 说明
* 简单介绍如何使用PostgreSQL搭建一个简单的相似图片搜索服务

## 方案介绍
* 使用PostgreSQL + imgsmlr插件计算图片特征值
* 提取目标图片特征值，使用PostgreSQL进行向量查询

## PostgreSQL安装
### mac os 安装 
```
brew install postgresql
```

### linux 源码安装

* 安装依赖 
``` 
sudo apt-get update
sudo apt-get install bison
sudo apt-get install flex
sudo apt-get install libreadline-gplv2-dev
sudo apt-get install zlib1g-dev
```

* 源码编译
```
sudo mkdir psql

cd psql

-- 下载源码安装包 --
wget https://ftp.postgresql.org/pub/source/v12.3/postgresql-12.3.tar.gz

tar zxvf postgresql-12.3.tar.gz

cd postgresql-12.3

-- 依次执行以下命令 -- 

-- 配置选项生成Makefile，默认安装到目录/usr/local/pgsql --
./configure 

-- echo 一下返回是否为0, 0表示无错误 --
echo $?

-- 编译 --
make 

-- 编译后再echo $? 如果为0 就可以安装了 --
echo $?

-- 安装 --
sudo make install 

-- 编译pg自带插件

-- 修改目录权限 -- 
sudo chmod 777 /usr/local/pgsql
sudo chmod 777 /usr/local/pgsql/*
sudo chmod 777 /usr/local/pgsql/share/extension

-- 编译并安装插件 --
cd ${psql源码目录}/contrib
make
sudo make install

```

* 创建存储目录 
```
-- 在数据盘创建pgsql目录 -- 
sudo mkdir pgsql
cd pgsql
sudo chmod 777 pgsql
```

* 配置环境变量
```
vim ~/.bashrc

export PATH=/usr/local/pgsql/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/pgsql/lib

-- 此处以数据目录在 /data/pgsql为例，根据实际情况做配置即可 --
export PGDATA=/data/pgsql

source ~/.bashrc
```

* 初始化数据库 
```
-- 数据目录权限设置 -- 
sudo chown -R superuser ${PGDATA}

initdb
```

* 启动数据库 
```
pg_ctl -D /data/pgsql -l logfile start

-- 关闭、重启命令 --
pg_ctl stop
pg_ctl restart
```

* 创建当前用户对应的数据库 
```
-- 启动数据库之后执行 --
createdb
```

* 创建用户和密码
```
CREATE USER foo WITH PASSWORD 'foo_pwd';
```

* 数据库参数设置 
```
cd ${PGDATA}
vim postgresql.conf

-- 日志输出 配置日志目录 --
logging_collector = on
log_directory = 'pg_log'

-- 最大连接数 --
max_connections = 1024

-- 内存参数 --
shared_buffers= 12GB
temp_buffers = 128MB
work_mem = 32MB
maintenance_work_mem = 32MB

-- 并行参数 --
max_worker_processes= 128
max_parallel_workers_per_gather= 16
max_parallel_workers= 16

-- 其他 --
max_wal_size = 1GB

-- 重启数据库 -- 
pg_ctl restart
```

* 远程访问设置 
```
-- 修改pg连接监听地址和端口 -- 
cd ${PGDATA}
vim postgresql.conf

listen_addresses = '*'
port = 1921

-- 配置远程访问策略 --
cd ${PGDATA}
vim pg_hba.conf

-- 示例,授权IP地址为192.168.100.*的主机,以用户foo加密码访问所有的数据库 --
host     all        foo             192.168.100.0/24          md5

-- 重启数据库 -- 
pg_ctl restart
```

* 设置开机自动启动 --
```
-- Ubuntu中利用 sysv-rc-conf设置开机自启动 --
sudo apt install sysv-rc-conf

cd ${psql源码目录}/contrib/start-scripts
sudo chmod a+x linux
vim linux 

-- 设置PGDATA, 启动日志位置, 启动使用的用户等 --
PGDATA="/data/pgsql"
PGLOG="$PGDATA/pg_log/auto-start.log"
PGUSER=superuser

-- 复制linux文件到/etc/init.d目录下,并更名postgresql --
sudo cp linux /etc/init.d/postgresql

-- 设置postgresql为开机自动启动 --
sudo sysv-rc-conf postgresql on
sudo sysv-rc-conf --list | grep postgresql

-- 重启服务器查看pg是否自动启动 -- 
sudo reboot
ps -ef | grep postgresql

```

### 编译imgsmlr插件
* [imgsmlr插件源码](https://github.com/postgrespro/imgsmlr)
* mac os环境make命令依赖 xcode, 编译前需确保xcode已经安装
* imgsmlr编译依赖[libgd库](https://github.com/libgd/libgd), mac os执行brew install gd安装gd, ubuntu使用apt-get install libgd2-noxpm-dev安装gd
* 下载源代码 git clone https://github.com/postgrespro/imgsmlr.git
* 修改imgsmlr_idx.c源码，添加以下宏定义, https://github.com/postgrespro/imgsmlr/issues/8
```
#ifndef FALSE
#define FALSE   (0)
#endif

#ifndef TRUE
#define TRUE    (!FALSE)
#endif
```

* mac os编译 
```
make USE_PGXS=1 CFLAGS=-I/xx/gd/include PG_LDFLAGS=-L/xx/gd/lib 
make USE_PGXS=1 CFLAGS=-I/xx/gd/include PG_LDFLAGS=-L/xx/gd/lib install   

CFLAGS PG_LDFLAGS参数用于指定gd.h文件和gd动态库所在位置
```
* linux编译 
```
make USE_PGXS=1 
make USE_PGXS=1 install
```

## 图片搜索
* 充分利用cpu + 内存，发挥硬件性能
* 数据库内存参数、并行参数调优
* 使用pg_prewarm插件进行数据预热，减少磁盘IO
* 图片特征值建立区表，利用dblink模拟并行查询，将单表查询改为分区表并行查询，充分利用每一个cpu核心

### 性能
* 硬件配置cpu 16核 32G内存, 单机一秒即可在一亿图片中完成搜索
