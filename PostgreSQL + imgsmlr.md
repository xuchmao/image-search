# 说明
* 简单介绍如何使用PostgreSQL搭建一个简单的相似图片搜索服务

## 方案介绍
* 使用PostgreSQL + imgsmlr插件计算图片特征值
* 计算目标图片特征值，使用PostgreSQL进行向量查询
* pg imgsmlr使用Haar小波变换提取图片特征值, 图片搜索效果比常见的感知哈希方式效果要好一些

## 性能
* 充分利用cpu + 内存，发挥硬件性能
* 数据库内存参数、并行参数调优
* 使用pg_prewarm插件进行数据预热，减少磁盘IO
* 图片特征值建立区表，使用dblink模拟并行查询，将单表查询改为分区表并行查询，充分利用每一个cpu核心
* 硬件配置cpu 16核 32G内存, 单机一秒即可在一亿图片中完成搜索

## linux 源码安装PostgreSQL

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
pg_ctl -D ${PGDATA} -l logfile start

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
port = 5432

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

## 编译imgsmlr插件
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

## 数据库表结构
```
-- 创建数据库 --
create database image with encoding='utf8';

-- 使用数据库 --
\c image;

-- 创建插件 --
create extension if not exists dblink;

create extension if not exists imgsmlr;

create extension if not exists pg_prewarm;

-- 创建图片特征值表 --
create table image_sig (
  image_id int primary key,
  sig signature not null
)
partition by hash (image_id);

-- image_sig字段注释 --
comment on table image_sig is '图片特征值表';
comment on column image_sig.image_id is '图片ID';
comment on column image_sig.sig is '图片signature';

-- 创建image_sig表sig字段索引
create index idx_img_sig on image_sig using gist(sig);

-- 创建image_sig分区表 --
do language plpgsql $$
declare
  i int;
begin
  for i in 0..15
  loop
    execute format('create table image_sig_%s partition of image_sig for values WITH (MODULUS 16, REMAINDER %s)', i, i);
  end loop;
end;
$$;

-- dblink 连接函数 --
create or replace function conn(
  name,   -- dblink名字
  text    -- 连接串,URL
) returns void as $$
declare
begin
  perform dblink_connect_u($1, $2);
  return;
exception when others then
  return;
end;
$$ language plpgsql strict;

-- 并行查询相似图像函数 根据需要设置conn_url中的user port --
create or replace function parallel_image_search(
  v_sig signature,  -- 图像特征值
  v_pt_limit int -- 每个分区返回数量
)
returns setof record as
$$
declare
  pt_num int;
  conn_url text := 'dbname=image user=foo port=5432'; -- dblink连接
  conn_name text := 'd_conn_';
  sql text;
  ts1 timestamp;
  t timestamp;
begin
  pt_num := 16 -1;

  t := clock_timestamp();
  for i in 0..pt_num loop
    ts1 := clock_timestamp();
    perform conn(conn_name||i, conn_url);
    perform image_id from dblink_get_result(conn_name||i, false) as t(image_id int);
    sql := format('select * from image_sig_%s order by sig <-> %L limit %s', i, v_sig, v_pt_limit);
    perform dblink_send_query(conn_name||i, sql);
  end loop;

  raise notice 'conn time cost - %', clock_timestamp()-t;
  ts1 := clock_timestamp();
  for i in 0..pt_num loop
    return query
    select
      image_id, sig
    from
      dblink_get_result(conn_name||i, false)
    as t(image_id int, sig signature);
  end loop;

  raise notice 'query time cost - %', clock_timestamp()-ts1;
  return;
end;
$$ language plpgsql strict;

-- image_sig表数据预热函数 --
create or replace function prewarm_image_sig()
returns void as $$
declare
begin
  raise notice 'prewarm image_sig start @ %', now()::text;
  for i in 0..15 loop
    execute format('select pg_prewarm(''image_sig_%s_sig_idx'', ''buffer'')', i);
    raise notice 'prewarm idx image_sig_%_sig_idx done', i;
    execute format('select pg_prewarm(''image_sig_%s'', ''buffer'')', i);
    raise notice 'prewarm idx image_sig_% done', i;
  end loop;
  raise notice 'prewarm image_sig end @ %', now()::text;
  return;
end;
$$ language plpgsql strict;

-- 性能测试测试SQL --
create or replace function get_rand_img_sig(int)
returns signature as $$
  select ('('||rtrim(ltrim(array(select (random()*$1)::float4 from generate_series(1,16))::text,'{'),'}')||')')::signature;
$$ language sql strict stable;

create or replace function explain_image_sig_query(
  v_max int default 1,
  v_limt int default 1
) returns void as $$
declare
  sig text;
  t_sql text;
begin
  for i in 0..v_max-1 loop
    sig := get_rand_img_sig(10);
    t_sql:= format('explain (analyze,verbose,timing,costs,buffers)
  select
    image_id
  from parallel_image_search(''%s''::signature, %s) as t(image_id int, sig signature)
  order by sig <-> ''%s''::signature
  limit 1;', sig, v_limt, sig);
    raise notice '%', t_sql;
    execute t_sql;
  end loop;
  return;
end;
$$ language plpgsql strict;

```

## 配置数据库重启脚本

* 在${PGDATA}创建bin目录，然后在bin目录创建数据库重启脚本db_restart.sh, 内容如下
```
#! /bin/bash

pg_ctl restart
sleep 3
psql -p 1921 image -c "select prewarm_image_sig()"
```

## 配置服务器重启数据库初始化脚本

* 在${PGDATA}/bin目录创建预热image_sig表sql文件, 命名为db_image_init.sql, 内容如下
```
\c image;
select prewarm_image_sig();
```

* 在/etc/init.d/postgresql文件添加以下内容
```
-- 设置psql命令位置 端口号 -- 
PSQL="$prefix/bin/psql"
PORT=1921

-- 在start/restart/reload位置添加以下命令 --
sleep 5
su - $PGUSER -c "$PSQL -p $PORT -f $PGDATA/bin/db_image_init.sql >> $PGDATA/pg_log/db_image_init.log 2>&1 &"
```

## 注意事项
* 业务开发，上传图片到postgreSQL图片库后，需要预热对应image_sig对应分区表和索引。分区编号可通过执行 explain select image_id from image_sig where image_id = :image_id 获取。

## 参考文档
* [PostgreSQL 相似搜索插件介绍大汇总](https://github.com/digoal/blog/blob/master/201809/20180904_01.md)
* [PostgreSQL 在视频、图片去重，图像搜索业务中的应用](https://github.com/digoal/blog/blob/master/201611/20161126_01.md)
* [PostgreSQL 11 相似图像搜索插件 imgsmlr 性能测试与优化 1 - 单机单表 (4亿图像)](https://github.com/digoal/blog/blob/master/201809/20180904_02.md)
* [PostgreSQL 11 相似图像搜索插件 imgsmlr 性能测试与优化 2 - 单机分区表 (dblink 异步调用并行) (4亿图像)](https://github.com/digoal/blog/blob/master/201809/20180904_03.md)
