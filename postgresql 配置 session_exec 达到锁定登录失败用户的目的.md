# postgresql 配置 session_exec 达到锁定登录失败用户的目的

## 说明

| 维度               | 说明           |
| ------------------ | -------------- |
| 系统版本           |                |
| postgresql版本     |                |
| postgresql安装方式 | 源码安装（？） |
| postgresql运行用户 | postgres       |

## 配置 session_exec

### 安装 postgresql

```bash
yum -y install gcc gcc-c++ readline-devel zlib-devel libuuid-devel uuid uuid-devel openssl openssl-devel
useradd postgres
mv postgresql-12.8.tar.gz /home/postgres
mkdir /data/app/postgresql /data/appData/postgresql /data/logs/postgresql/postgresql -p
chown -R postgres:postgres /data/app/postgresql /data/appData/postgresql /data/logs/postgresql /home/postgres
su  - postgres
tar -xf postgresql-12.8.tar.gz
cd postgresql-12.8/
./configure --prefix=/data/app/postgresql --with-ossp-uuid --with-openssl --with-zlib
make -j4 && make install
echo -e "export PGHOME=/data/app/postgresql\nexport PATH=\$PATH:\$HOME/.local/bin:\$HOME/bin:\$PGHOME/bin" >> ~/.bashrc
source ~/.bashrc
initdb -D /data/appData/postgresql/

```

### 安装 file_fdw 扩展

```bash
[postgres@localhost postgresql-12.8]$ ls
aclocal.m4  config.log     configure     contrib    doc          GNUmakefile.in  INSTALL   README
config      config.status  configure.in  COPYRIGHT  GNUmakefile  HISTORY         Makefile  src
[postgres@localhost postgresql-12.8]$ cd contrib/
[postgres@localhost contrib]$ file_fdw/
[postgres@localhost file_fdw]$ make
[postgres@localhost file_fdw]$ make install

```

### 安装 session_exec

去GitHub下载安装。（需要先将 pg_config 命令所在的文件夹加入 PATH）。

```bash
#查看 PATH
echo $PATH

#假设命令在文件夹 /usr/pgsql-14/
#使用如下命令，可以将 pg_config 暂时加入 PATH。
#配置完后可以通过 each $PATH 查看配置结果。
#生效方法：立即生效。
#有效期限：临时改变。只在当前终端窗口有效，窗口关闭后就会恢复原有的 PATH 配置。
#用户局限：仅限当前用户。
export PATH=/usr/pgsql-14/bin:$PATH
```



```bash
unzip session_exec-master.zip
cd session_exec-master/
make pg_config=/data/app/postgresql/bin/pg_config
make pg_config=/data/app/postgresql/bin/pg_config install

```

### 修改配置文件 postgresql.conf

> session_preload_libraries参数：一个或者多个要在连接开始时预载入的共享库。
> shared_preload_libraries参数：一个或者多个要在服务器启动时预载入的共享库。

```bash
logging_collector = on # 有了就不用再添加了
log_destination = 'csvlog' # 有了就不用再添加了
session_preload_libraries='session_exec'
session_exec.login_name='login'

```

### 重启数据库，让配置生效⭐⭐⭐注意需提前发公告，一般重启需要两分钟左右。

```bash
# 重启服务
service postgresql-14 restart

# 查看数据库状态
service postgresql-14 status
```

![image-20231025175034731](C:\Users\arona\AppData\Roaming\Typora\typora-user-images\image-20231025175034731.png)

### 创建外部表 postgres_log

```bash
[root@zldrugputdb155 session_exec-master]# su postgres
bash-4.2$ psql
psql (14.5)
Type "help" for help.

postgres=# 

```



```sql
create extension file_fdw;

CREATE SERVER pglog FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE public.postgres_log(
log_time timestamp(3) with time zone,
  user_name text,
  database_name text,
  process_id integer,
  connection_from text,
  session_id text,
  session_line_num bigint,
  command_tag text,
  session_start_time timestamp with time zone,
  virtual_transaction_id text,
  transaction_id bigint,
  error_severity text,
  sql_state_code text,
  message text,
  detail text,
  hint text,
  internal_query text,
  internal_query_pos integer,
  context text,
  query text,
  query_pos integer,
  location text,
  application_name text
) SERVER pglog
OPTIONS ( program 'find /data/logs/postgresql -type f -name "*.csv" -mtime -1 -exec cat {} \;', format 'csv' );

grant SELECT on postgres_log to PUBLIC;

```

### 创建 登陆验证 函数 login

```sql
create or replace function public.login() returns void as $$
declare
res record;
failed_login_times int = 5;
failed_login int = 0;
begin
--获取数据库中所有可连接数据库的用户
for res in select rolname from pg_catalog.pg_roles where rolcanlogin= 't' and rolname !='postgres'
loop
  raise notice 'user: %!',res.rolname;
  --获取当前用户最近连续登录失败次数
  select count(*) from (select log_time,user_name,error_severity,message,detail from public.postgres_log where command_tag = 'authentication' and user_name = res.rolname and (detail is null or detail not like 'Role % does not exist.%') order by log_time desc limit failed_login_times) A WHERE A.error_severity='FATAL' into  failed_login ;
  raise notice 'failed_login_times: %! failed_login: %!',failed_login_times,failed_login;
  --用户最近密码输入错误次数达到5次或以上
  if failed_login >= failed_login_times then
    --锁定用户
    EXECUTE format('alter user %I nologin',res.rolname);
    raise notice 'Account % is locked!',res.rolname;
  end if;
end loop;
end;
$$ language plpgsql strict security definer set search_path to 'public';

```

### 验证登陆失败结果

修改 pg_hba.conf 添加对测试用户的密码策略，然后重启 pg。

```bash
host    all             test1             127.0.0.1/32          md5
```

创建测试用户

```sql
create user test1 encrypted password 'Rootmaster@777';

```

前 5 次输入错误密码，如下：

```bash
[postgres@localhost ~]$ psql -Utest1 -h127.0.0.1 -p18126 -d postgres
Password for user test1:
psql: error: FATAL:  password authentication failed for user "test1"
[postgres@localhost ~]$ psql -Utest1 -h127.0.0.1 -p18126 -d postgres
Password for user test1:
psql: error: FATAL:  password authentication failed for user "test1"
[postgres@localhost ~]$ psql -Utest1 -h127.0.0.1 -p18126 -d postgres
Password for user test1:
psql: error: FATAL:  password authentication failed for user "test1"
[postgres@localhost ~]$ psql -Utest1 -h127.0.0.1 -p18126 -d postgres
Password for user test1:
psql: error: FATAL:  password authentication failed for user "test1"
[postgres@localhost ~]$ psql -Utest1 -h127.0.0.1 -p18126 -d postgres
Password for user test1:

```

查询 test1 锁定状态，已经不可登录。

```bash
[postgres@localhost session_exec-master]$ psql -Upostgres -h127.0.0.1 -p18126 -d postgres
NOTICE:  user: test1!
NOTICE:  failed_login_times: 5! failed_login: 5!
NOTICE:  Account test1 is locked!
psql (12.8)
Type "help" for help.

postgres=# select rolcanlogin from pg_catalog.pg_roles where rolname='test1';
 rolcanlogin
-------------
 f
(1 row)

```

test1 再次登录，提示如下：

```bash
[postgres@localhost session_exec-master]$ psql -Utest1 -h127.0.0.1 -p18126 -d postgres
Password for user test1:
psql: error: FATAL:  role "test1" is not permitted to log in

```

具体的用户，解锁命令如下：

```bash
[postgres@localhost ~]$ psql -Upostgres -h127.0.0.1 -p18126 -d testdb1
psql (12.8)
Type "help" for help.

testdb1=# alter user test1 login;
ALTER ROLE

```



药品：10.34.200.155

费用：10.34.200.34

数据处理平台：10.34.200.60（make过程中少了文件，仅完成压缩包解压）

基础服务：10.34.200.75



## 问题与改进