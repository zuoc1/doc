# 一. Oracle

## 1. 创建用户并赋予权限

```sql
-- --创建泵目录和用户，并赋予最高权限
create tablespace BDCDJ datafile 'D:\Oracle\BDCDJ.DBF' size 500M autoextend on next 100m maxsize unlimited;

create user BDCDJ identified by BDCDJ default tablespace BDCDJ ACCOUNT UNLOCK;

GRANT DBA,CONNECT,RESOURCE,ALTER ANY ROLE,ALTER ANY TABLE,ALTER USER,CREATE ROLE,CREATE TABLE,CREATE USER,DROP ANY ROLE,
DROP ANY TABLE,DROP USER,GRANT ANY ROLE TO BDCDJ;

CREATE DIRECTORY WHSJ AS 'D:\Oracle';
GRANT READ,WRITE ON DIRECTORY WHSJ TO BDCDJ;

-- --每个权限单独赋予
GRANT SELECT ON  "SYS"."ALL_COL_COMMENTS" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."ALL_TABLES" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_DATA_FILES" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_FREE_SPACE" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_INDEXES" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_IND_COLUMNS" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_ROLES" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_ROLE_PRIVS" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_TABLESPACES" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_TAB_COLUMNS" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_TAB_PRIVS" TO BDCDJ WITH GRANT OPTION;
GRANT SELECT ON  "SYS"."DBA_USERS" TO BDCDJ WITH GRANT OPTION;
```

### 1.1 修改用户密码有效时间

Oracle 11g R2，静默安装后用户的密码有效期默认设置为180天，180天后密码将失效，oracle会提示要修改密码。

解决思路一：定期修改数据库用户密码，对于已经过期的账户修改下密码即可解锁。

解决思路二：将数据库密码设置为永久有效，需要管理员权限。

```sql
-- 1、查看用户的proifle是哪个，一般是default：
SELECT username,PROFILE FROM dba_users;

-- 2、查看指定概要文件（如default）的密码有效期设置：
SELECT * FROM dba_profiles s WHERE s.profile='DEFAULT' AND resource_name='PASSWORD_LIFE_TIME'; 

-- 3、将密码有效期由默认的180天修改成“无限制”：修改之后不需要重启动数据库，会立即生效。
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

### 1.2 设置客户端连接数

个人使用不用设置，企业使用需要设置足够的连接数。最佳配置：sessions=(1.1*process+5)。

```sql
-- 查看
SQL> show parameter processes
SQL> show parameter sessions

-- 修改完重启数据库
alter system set processes=300 scope=spfile;
alter system set sessions=335 scope=spfile;

-- 查询当前连接数
select * from v$process;
select * from v$session;

-- 当前那些用户正在使用数据
select a.osuser,a.username,b.cpu_time/b.executions/1000000||'s',b.sql_fulltext,a.machine from v$session a,v$sqlarea b where a.sql_address=b.address order by b.cpu_time/b.executions desc;
```

### 1.3 设置最大远程连接数

```sql
-- open_links：每个session最多允许的dblink数量
-- open_links_per_instance：每个实例最多允许的dblink个数
SQL> show parameter open_links
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
open_links                           integer     4
open_links_per_instance              integer     4

-- 重启后生效
alter system set open_links=50 scope=spfile;
alter system set open_links_per_instance=50 scope=spfile;
```

### 1.4 解锁重试密码太多的用户

一般数据库默认是10次尝试失败后锁住用户。

```sql
alter user scott account unlock;

-- 设置为无限次
alter profile default limit FAILED_LOGIN_ATTEMPTS unlimited;
```

### 1.5 创建低权限用户

```sql
-- 创建用户名和表空间
create tablespace BDCDJ_BL datafile 'D:\ProgramFiles\Oracle\WHSJ\BDCDJ_BL.DBF' size 500M autoextend on next 100m maxsize unlimited;

create user BDCDJ_BL identified by BDCDJ_BL default tablespace BDCDJ_BL ACCOUNT UNLOCK;

GRANT CONNECT,RESOURCE,CREATE TABLE TO BDCDJ_BL;

GRANT READ,WRITE ON DIRECTORY WHSJ TO BDCDJ_BL;

-- 赋表权限
GRANT select,update ON bdc_regn_fwsyq TO BDCDJ_BL;
GRANT select,update ON bdc_regn_tdsyq TO BDCDJ_BL;
GRANT select,update ON bdc_regn_yg TO BDCDJ_BL;
GRANT select,update ON bdc_regn_dy TO BDCDJ_BL;
GRANT select,update ON bdc_regn_cf TO BDCDJ_BL;
GRANT select,update ON bdc_regn_yy TO BDCDJ_BL;
GRANT select,update ON bdc_regn_zb TO BDCDJ_BL;

GRANT select,update ON bdc_qlr TO BDCDJ_BL;
GRANT select,update ON bdc_qlrlb TO BDCDJ_BL;

GRANT select,update ON bdc_xm TO BDCDJ_BL;
GRANT select,update ON bdc_zrz TO BDCDJ_BL;
GRANT select,update ON bdc_ljz TO BDCDJ_BL;
GRANT select,update ON bdc_h TO BDCDJ_BL;
GRANT select,update ON bdc_zdjbxx TO BDCDJ_BL;
GRANT select,update ON oa2_sysdic TO BDCDJ_BL;

-- 取别名
create or replace synonym BDCDJ_BL.bdc_regn_fwsyq for BDCDJ.bdc_regn_fwsyq;
create or replace synonym BDCDJ_BL.bdc_regn_tdsyq for BDCDJ.bdc_regn_tdsyq;
create or replace synonym BDCDJ_BL.bdc_regn_yg for BDCDJ.bdc_regn_yg;
create or replace synonym BDCDJ_BL.bdc_regn_dy for BDCDJ.bdc_regn_dy;
create or replace synonym BDCDJ_BL.bdc_regn_cf for BDCDJ.bdc_regn_cf;
create or replace synonym BDCDJ_BL.bdc_regn_yy for BDCDJ.bdc_regn_yy;
create or replace synonym BDCDJ_BL.bdc_regn_zb for BDCDJ.bdc_regn_zb;

create or replace synonym BDCDJ_BL.bdc_qlr for BDCDJ.bdc_qlr;
create or replace synonym BDCDJ_BL.bdc_qlrlb for BDCDJ.bdc_qlrlb;

create or replace synonym BDCDJ_BL.bdc_xm for BDCDJ.bdc_xm;
create or replace synonym BDCDJ_BL.bdc_zrz for BDCDJ.bdc_zrz;
create or replace synonym BDCDJ_BL.bdc_ljz for BDCDJ.bdc_ljz;
create or replace synonym BDCDJ_BL.bdc_h for BDCDJ.bdc_h;
create or replace synonym BDCDJ_BL.bdc_zdjbxx for BDCDJ.bdc_zdjbxx;
create or replace synonym BDCDJ_BL.oa2_sysdic for BDCDJ.oa2_sysdic;
```

## 2. 删除用户

```sql
-- 一步步执行，有可能用户被占用
drop user BDCDJ cascade;
-- 删除表空间（包涵内容和数据库文件），dbf文件如没删除掉需手动删除
drop tablespace BDCDJ including contents and datafiles;

-- 查询正在使用当前用户的进程
select username, sid, serial# from v$session where username='BDCDJ';
-- 杀掉进程
alter system kill session '246,153';
```

## 3. 备份还原

- @echo off执行以后，后面所有的命令均不显示回显，包括本条命令。
- echo off执行以后，后面所有的命令均不显示回显，但本条命令是显示的。
- cls清屏

### 2.1 exp

exp跳过空表解决方法：

1. insert一行，再rollback就产生segment，不实用；

2. 设置deferred_segment_creation参数，默认是TRUE，但只对新建的表有用，已存在的空表无用；
   alter system set deferred_segment_creation=false scope=both;

3. 已存在的空表：

   select 'alter table '||table_name||' allocate extent;' from user_tables where segment_created='NO';

```powershell
@echo off
cls
exp BDCDJ/BDCDJ@ORCL file=.\BDCDJ_%date%.dmp owner=BDCDJ log=.\EXP_BDCDJ_%date%.txt
```

### 2.2 imp

```powershell
@echo off
cls
imp BDCDJ/BDCDJ@ORCL file=.\BDCDJ_2015-06-12.dmp full=y log=.\IMP_BDCDJ.txt
```

### 2.3 expdp

导出部分表加：tables=table1,table2,table3

```powershell
::设置泵目录
set DIRECTORY_DIR=D:\Oracle
::设置备份目录
set BACKUP_DIR=D:\Oracle
::设置rar执行文件路径
set RAR_CMD="C:\Program Files\WinRAR\Rar.exe"
::获取系统时间
set H=%time:~,2%
if %H% leq 9 (set H=0%H:~1,1%) else (set H=%H%)
set DATETIME=%date:~0,4%%date:~5,2%%date:~8,2%%H%%time:~3,2%%time:~6,2%

::设置数据库连接并导出
set ORACLE_DB=localhost:1521/ORCL
set ORA_USERNAME=BDCDJ
set ORA_PASSWORD=BDCDJ
expdp %ORA_USERNAME%/%ORA_PASSWORD%@%ORACLE_DB% directory=WHSJ dumpfile="%ORA_USERNAME%.dmp" logfile="%ORA_USERNAME%_EXPDP.log" schemas=%ORA_USERNAME%

::压缩并删除原有文件，-hp设置密码
%RAR_CMD% a -ep -df -hp123zcdd "%BACKUP_DIR%\%DATETIME%.rar" "%DIRECTORY_DIR%\%ORA_USERNAME%.dmp" "%DIRECTORY_DIR%\%ORA_USERNAME%_EXPDP.log"
::手动执行则加上防止命令窗口退出，设置定时任务则去掉
pause
```

### 2.4 impdp

impdp中加table_exists_action={SKIP | APPEND | TRUNCATE | REPLACE }

skip 是如果已存在表，则跳过并处理下一个对象；
append是为表增加数据；
truncate是截断表，然后为其增加新数据；
replace是删除已存在表，重新建表并追加数据。

```powershell
impdp BDCDJ/BDCDJ@ORCL directory=WHSJ dumpfile='BDCDJ.dmp' logfile='BDCDJ.log'  schemas=BDCDJ
pause

::如果导出是BDCDJ_ZF，想导入到BDCDJ中，则按这个语句导入数据库：
impdp BDCDJ/BDCDJ@orcl remap_schema=BDCDJ_ZF:BDCDJ remap_tablespace=BDCDJ_ZF:BDCDJ directory=WHSJ dumpfile=BDCDJ.dmp logfile=BDCDJ.log ignore=y
```

## 4. 字段类型

以下情形假设一个汉字占两个字节，当编码为UTF-8时一个汉字可能占3个字节时另当别论。

| 类型          | 存储汉字 | 存储ASCII | 说明                                                | 转换                                                  |
| ------------- | -------- | --------- | --------------------------------------------------- | ----------------------------------------------------- |
| NVARCHAR2(10) | 10       | 10        | 10指字符数                                          | cast('a' as nvarchar2(20))                            |
| VARCHAR2(10)  | 5        | 10        | 10指字节数                                          | cast('a' as varchar2(20))                             |
| NCHAR(10)     | 10       | 10        | 定长，效率高，空间消耗大。<br>长度不够会填充空格。  | translate('a' USING NCHAR_CS)                         |
| CHAR(10)      | 5        | 10        | 定长，效率高，空间消耗大。<br/>长度不够会填充空格。 | cast('a' as char(20))<br>translate('a' USING CHAR_CS) |

当字段内容为 `'哈ms'`时：此时a!=b，因为char(20)中有16个空格，trim(a)=b；


| 字段名 | 类型          | length(字符数) | lengthb(字节数) |
| ------ | ------------- | -------------- | --------------- |
| a      | char(20)      | 19             | 20              |
| b      | varchar2(20)  | 3              | 4               |
| c      | nvarchar2(20) | 3              | 6               |

`update table set b=a,c=a;`后结果:

| 字段名 | 类型          | length(字符数) | lengthb(字节数) |
| ------ | ------------- | -------------- | --------------- |
| a      | char(20)      | 19             | 20              |
| b      | varchar2(20)  | 19             | 20              |
| c      | nvarchar2(20) | 19             | 38              |

结论：将char赋值给varchar2和nvarchar2会带上空格，用cast和translate都没用，必须用trim()。

例子：

```sql
select length('叶德华abc') from dual;--length按字符计，汉字、英文、数字都是1个字符，故这里返回6
select lengthb('叶德华abc')  from dual;--length按字节计，我这里是UTF-8编码，汉字3个字节，英文一个字节，故这里返回12
select substr('叶德华abc',1,4) from dual;--substr按字符截取，截取到a，返回：叶德华a
select substrb('叶德华abc',1,2) from dual;--substrb按字节截取，2不足一个汉字长度，返回：两个空格
select substrb('叶德华abc',1,3) from dual;--substrb按字节截取，3刚好是一个汉字长度，返回：叶
select substrb('叶德华abc',1,4) from dual;--substrb按字节截取，4多余一个汉字少于两个汉字，返回：叶 加一个空格

select instr('日日花前长病酒','花前',1,1) from dual;--output:3
select instrb('日日花前长病酒','花前',1,1) from dual;--output:5

--返回结果：4    也就是说：在"helloworld"的第2(e)号位置开始，查找第二次出现的“l”的位置
select instr('helloworld','l',2,2) from dual;
--返回结果：4    也就是说：在"helloworld"的第3(l)号位置开始，查找第二次出现的“l”的位置
select instr('helloworld','l',3,2) from dual;
--返回结果：9    也就是说：在"helloworld"的第4(l)号位置开始，查找第二次出现的“l”的位置
select instr('helloworld','l',4,2) from dual;
--返回结果：9    也就是说：在"helloworld"的倒数第1(d)号位置开始，往回查找第一次出现的“l”的位置
select instr('helloworld','l',-1,1) from dual;
 --返回结果：4    也就是说：在"helloworld"的倒数第1(d)号位置开始，往回查找第二次出现的“l”的位置
select instr('helloworld','l',-2,2) from dual;
--返回结果：9    也就是说：在"helloworld"的第2(e)号位置开始，查找第三次出现的“l”的位置
select instr('helloworld','l',2,3) from dual;
--返回结果：3    也就是说：在"helloworld"的倒数第2(l)号位置开始，往回查找第三次出现的“l”的位置
select instr('helloworld','l',-2,3) from dual;
```

## 5. 表

表是段，但段不一定是表，段还有index段、undo段、分区之类的。具体如下：

首先，要清楚它们的概念：表是逻辑对象；段是物理存储对象。

然后，再看它们之间的关系： 

① 段的存在，并不是依赖于表的。建立一些其它逻辑对象也会会创建段，如索引、物化视图

② 一张普通表(堆组织表)对应一个段

③ 表的建立，并不意味着段的创建，如临时表（Global Temporary Table）

④ 一张表也可以创建多个段，如分区表（Partition Table）

⑤ 多个表也可以共存于一个段，如簇表（Cluster Table）


## 6. 表空间

Database是由一个或多个被称为表空间（tablespace）的逻辑存储单位构成。表空间内的逻辑存储单位为段（segment），段又可以继续划分为数据扩展（extent）。而数据扩展是由一组连续的数据块（datablock）构成。

一、大文件表空间：由一个单一的大文件构成，而不是若干个小数据文件

```sql
select * from dba_tablespaces;-- BIGFILE为YES是大文件表空间
ALTER DATABASE SET DEFAULT bigfile TABLESPACE;-- 修改成默认创建大文件表空间
CREATE BIGFILE TABLESPACE...-- 创建大文件表空间
```

二、小文件表空间：单个文件最大32G

```sql
-- 只能修改database datafile大小，而不是tablespace大小
ALTER DATABASE DATAFILE '/oracle/oradata/db/GAME.dbf' RESIZE 4000M;
```

### 6.1 表空间联机与脱机

```sql
-- 使表空间脱机
ALTER TABLESPACE game OFFLINE;
-- 使数据文件脱机
ALTER DATABASE DATAFILE 3 OFFLINE;

-- 使表空间联机
ALTER TABLESPACE game ONLINE;
-- 使数据文件联机
ALTER DATABASE DATAFILE 3 ONLINE;

-- 使表空间只读
ALTER TABLESPACE game READ ONLY;
-- 使表空间可读写
ALTER TABLESPACE game READ WRITE;
-- 意外删除了数据文件
ALTER TABLESPACE game OFFLINE FOR RECOVER;
```

### 6.2 缩小数据文件

某个表空间A曾经使用了很大的空间，现在A上面已经没有多少数据，而硬盘空间吃紧。能否缩小表空间A的大小，释放一些空间出来。
通过enable row movement，然后 shrink(缩小) space。最后resize表空间A所在的数据文件，就可以了。最后也许还需要重建索引，因为row movement后，rowid将发生变化。

#### 6.2.1 创建测试表空间和用户

```sql
create tablespace test datafile ‘D:\Oracle\test.dbf’ size 10m autoextend on;

create user test1 idenitfied by test1
default tablespace test;
grant connect,resource to test1;
```

查看TEST表空间大小

```sql
select b.name,sum(a.bytes/1024/1024) "MB" from v$datafile a,v$tablespace b where a.ts#=b.ts# and b.name='TEST' group by b.name;
NAME                                   MB
------------------------------ ----------
TEST                                  10
```

#### 6.2.2 用test1用户登录，并创建测试表

```sql
conn test1/test1;

create table a as select * from all_objects;
insert into a select * from a;
insert into a select * from a;
insert into a select * from a;--多次插入
```

再次查询

```sql
select b.name,sum(a.bytes/1024/1024) "MB" from v$datafile a,v$tablespace b where a.ts#=b.ts# and b.name='TEST' group by b.name;
TABLESPACE_NAME                        MB
------------------------------ ----------
TEST                                    144
```

查看object_id=52880的数据行的rowid

```sql
select rowid,object_id from a where object_id=52880;
ROWID               OBJECT_ID
------------------ ----------
AAAM7gAAGAAAAFOAAa      52880
AAAM7gAAGAAAAMXABA      52880
AAAM7gAAGAAAAaqAAn      52880
AAAM7gAAGAAAAo4AAL      52880
AAAM7gAAGAAABkvAAL      52880
```

#### 6.2.3 删除表a中的部分内容

查看ROW_MOVEMENT是否开启

```sql
select owner,table_name,row_movement from dba_tables where table_name='A';
OWNER                          TABLE_NAME                     ROW_MOVEMENT
------------------------------ ------------------------------ ------------
TEST1                          A                              DISABLED      
```

查询表的总行数和所占空间大小

```sql
select count(*) from a;
  COUNT(*)
----------
   1304864

select owner,segment_name,segment_type,bytes/1024/1024 "MB" from dba_segments where segment_name='A';
OWNER    SEGMENT_NAME     SEGMENT_TYPE         MB
--------     ------------              ------------ ----------
TEST1             A                         TABLE               144
```

删除部分内容

```sql
delete from a where rownum<100000;
commit;
delete from a where rownum<100000;
commit;
delete from a where rownum<900000;
commit;
```

再次查询表所占空间大小，发现并没有改变

```sql
select owner,segment_name,segment_type,bytes/1024/1024 "MB" from dba_segments where segment_name='A';
OWNER    SEGMENT_NAME     SEGMENT_TYPE         MB
--------     ------------              ------------ ----------
TEST1             A                         TABLE               144
```

#### 6.2.4 启用row movement，并shrink space

```sql
conn / as sysdba
alter table test1.a enble row movement;
alter table test1.a shrink space;
```

通过允许行移动，并缩减空间后，重新查看得知A表现在占用表空间为22.75MB。

```sql
select owner,segment_name,segment_type,bytes/1024/1024 "MB" from dba_segments where segment_name='A';
OWNER    SEGMENT_NAME     SEGMENT_TYPE         MB
--------     ------------              ------------ ----------
TEST1             A                         TABLE               22.75
```

再次查询object_id=52880的行的rowid，可以看到，shrink space后rowid发生了变化。所以，当进行了row movement，并 shrink space后，需要重建索引。以便索引能指向正确的数据行。

```sql
select rowid,object_id from a where object_id=52880 order by 1;
ROWID               OBJECT_ID
------------------ ----------
AAAM7gAAGAAAAAUAAC      52880
AAAM7gAAGAAAAAUAAv      52880
AAAM7gAAGAAAAAUAA8      52880
AAAM7gAAGAAAAAVAAY      52880
AAAM7gAAGAAAAAVAA0      52880
```

由于表空间TEST只有一个数据文件，而且表空间TEST中只有一个对象A，现在A所占空间为22.75M，那么我们可以将test.dbf resize为30MB。这样，表空间TEST就只占用30MB的大小了。当表中存在很多对象时，需要的表空间需要计算得到。

```sql
-- 查询最后一块的id
SELECT MAX(block_id) FROM dba_extents WHERE tablespace_name = 'TEST';-- 433536
-- 计算最小需要的大小
show parameter db_block_size;-- 8192
SELECT 433536*8192/1024/1024 FROM dual;-- 3387
-- 修改数据文件大小
alter database datafile 'D:\Oracle\test.dbf' resize 30m;
```

再次查询表空间大小

```sql
select b.name,sum(a.bytes/1024/1024) "MB" from v$datafile a,v$tablespace b where a.ts#=b.ts# and b.name='TEST' group by b.name;
NAME                                   MB
------------------------------ ----------
TEST                                  30
```

#### 6.2.5 补充

对于oracle的表，数据库在日常使用过程中，不断的insert，delete，update操作，导致表和索引出现碎片是在所难免的事情，碎片多了，sql的执行效率自然就差了，道理很简单，高水位线（HWL）下的许多数据块都是无数据的，但全表扫描的时候要扫描到高水位线的数据块，也就是说oracle要做许多的无用功！

delete并不会降低高水位线，虽然删除了数据内容，但并不会释放空间。只有启用row movement，并 shrink space后，才能降低高水位线。另外，即使在没有启用row movement的情况下，truncate也会降低高水位线。不过truncate会将表中所有数据全部清除。

```sql
-- 生成所有表的shrink space语句
select 'alter table '||table_name||' enable row movement;' from user_tables order by table_name;
select 'alter table '||table_name||' shrink space;' from user_tables order by table_name;
```

以前的技术：在9i的时候，一个很成熟的碎片整理技术。这个move确实整理碎片的效率很高，但是美中不足的是不移动高水位。而且还要重建索引。

```sql
--高水位以下合并碎片，不移动高水位
alter table xxx move;
--高水位以下合并碎片，同时压缩表，不移动高水位
alter table xxx move compress;
```

人工减少表空间：几张大表是日报表，数据增删改查比较大，碎片也很多，其中实际占用的空间肯定比高水位的空间小。人工过程很复杂很耗时间。

```sql
1.把需要的数据放到一个临时表，create table temp_t1 nologging Select /*+parallel(tab n)*/ * from t1 where .....
2.彻底删了原表，drop table t1 pruge;
3.还原表,ALTER TABLE oldname temp_t1 TO t1 ;
4.重建索引，create index index_name on t1(...);
```

### 6.3 其它查询

```sql
-- 查看表空间的数据文件路径
select t1.name,t2.name from v$tablespace t1,v$datafile t2 where t1.ts#=t2.ts#;

-- 查表空间使用情况
SELECT a.tablespace_name "表空间名",b.total "表空间大小(M)",a.free "表空间剩余大小(M)",b.total-a.free "表空间使用大小(M)",round((total-free)/total,4)*100 "使用率%" 
FROM (SELECT tablespace_name,SUM(bytes)/(1024*1024) free FROM dba_free_space GROUP BY tablespace_name) a,
     (SELECT tablespace_name,SUM(bytes)/(1024*1024) total FROM dba_data_files GROUP BY tablespace_name) b
     WHERE a.tablespace_name = b.tablespace_name;

-- 查看用户对应的表空间
SELECT distinct a.owner,a.tablespace_name FROM dba_tables a,dba_users b where a.owner=b.username and a.tablespace_name is not null and b.account_status='OPEN' order by a.owner;
```

### 6.4 重建临时表空间，解决临时表空间过大的问题

```sql
-- 0.查看目前默认的临时表空间
select * from database_properties where property_name='DEFAULT_TEMP_TABLESPACE';

-- 1.创建中转临时表空间
create temporary tablespace temp1 tempfile '/u01/app/oradata/ARPDB/temp03.dbf' size 512m reuse autoextend on next 1m maxsize unlimited;

-- 2.改变缺省临时表空间为刚刚创建的新临时表空间temp1
alter database default temporary tablespace temp1;

-- 3.删除原临时表空间
drop tablespace temp including contents and datafiles;

-- 如果删除表空间的时候，hang住的话，可以使用下列语句：
-- 先把运行在temp临时表空间的sql语句kill掉，这样的sql语句多为排序的语句
Select se.username,se.sid,se.serial#,su.extents,su.blocks*to_number(rtrim(p.value))as Space,tablespace,segtype,sql_text
from v$sort_usage su,v$parameter p,v$session se,v$sql s
where p.name='db_block_size' and su.session_addr=se.saddr and s.hash_value=su.sqlhash
and s.address=su.sqladdr
order by se.username,se.sid;
-- 查询出来之后，kill掉这些sql语句：假如某一条运行的sql语句的SID为524，serial#为778
alter system kill session '71,58031';

-- 4.重建临时表空间
create temporary tablespace temp tempfile '/u01/app/oradata/ARPDB/temp01.dbf' size 512m reuse autoextend on next 1m maxsize unlimited;

-- 5.重置缺省临时表空间为新建的temp表空间
alter database default temporary tablespace temp;

-- 6.删除中转用临时表空间
drop tablespace temp1 including contents and datafiles;
```

### 6.5 迁移表的表空间

```sql
-- 把tablename表从spaceOne移动到spaceTwo中
alter table spaceOne.tablename move tablespace spaceTwo;
```

备注一：
当前的用户必须对spaceTwo、spaceOne都有操作权限才可以。
备注二：
其实如果对两个表空间都有权限的话，可以通过
`create spaceTwo.tablename as select * from spaceOne.tablename;`
之后再删除spaceOne中tablename表的间接方式也能实现。

### 6.6 批量迁移不在当前表空间中的数据

有些用户的数据会跨两个表空间，导致恢复备份时要建多个表空间，不然会报错。次语句解决次问题。

```sql
DECLARE
  v_user varchar2(100):='BDCDJ_TJ';--所在用户
  v_space varchar2(100):='BDCDJ_TJ';--旧表空间
  v_flag int:=0;--0:输出不执行 1:输出并执行
  a_count INT:=0;
  b_count INT:=0;
  c_count INT:=0;
  CURSOR a_mysql IS
    SELECT 'alter index '||owner||'.'||index_name||' rebuild tablespace '||v_space mysql FROM dba_indexes WHERE owner=v_user and
    tablespace_name!=v_space and index_type<>'LOB';
  CURSOR b_mysql IS
    SELECT 'alter table '||owner||'.'||table_name||' move tablespace '||v_space mysql FROM dba_tables WHERE owner=v_user and 
    tablespace_name!=v_space;
  CURSOR c_mysql IS
    SELECT 'alter table '||owner||'.'||table_name||' move tablespace '||v_space||' lob('||column_name||') store as(tablespace '||
    v_space||')' mysql FROM dba_lobs WHERE owner=v_user and tablespace_name!=v_space;
BEGIN
  DBMS_OUTPUT.put_line('迁移索引');
  FOR r_mysql IN a_mysql LOOP
    DBMS_OUTPUT.put_line(r_mysql.mysql);
    IF v_flag=1 THEN
      EXECUTE IMMEDIATE r_mysql.mysql;
    END IF;
    a_count:=a_count+1;
  END LOOP;
  DBMS_OUTPUT.put_line('a_count:'||a_count);
  
  DBMS_OUTPUT.put_line('迁移表');
  FOR r_mysql IN b_mysql LOOP
    DBMS_OUTPUT.put_line(r_mysql.mysql);
    IF v_flag=1 THEN
      EXECUTE IMMEDIATE r_mysql.mysql;
    END IF;
    b_count:=b_count+1;
  END LOOP;
  DBMS_OUTPUT.put_line('b_count:'||b_count);
  
  DBMS_OUTPUT.put_line('迁移lob字段');
  FOR r_mysql IN c_mysql LOOP
    DBMS_OUTPUT.put_line(r_mysql.mysql);
    IF v_flag=1 THEN
      EXECUTE IMMEDIATE r_mysql.mysql;
    END IF;
    c_count:=c_count+1;
  END LOOP;
  DBMS_OUTPUT.put_line('c_count:'||c_count);
END;
```

## 7. 创建修改profile

```sql
--1. 创建profile文件
create profile profile_test1 limit
failed_login_attempts 3
password_lock_time 1
password_life_time unlimited
password_reuse_time 90
sessions_per_user unlimited
cpu_per_session unlimited
cpu_per_call 1000
connect_time 30
logical_reads_per_session default
logical_reads_per_call 1000
composite_limit 6
private_sga 128k;

--2. 给用户指定profile
alter user 用户名 profile 文件名;

--3. 修改profile
alter profile [资源文件名] limit [资源名] unlimited;

--4. 删除profile（已分配的profile，删除时必须加cascade进行联级删除，该用户自动变为default）
drop profile [资源文件名] [CASCADE] 
```

修改空闲自动断开时间

```sql
select * from dba_profiles;

alter profile MONITORING_PROFILE LIMIT IDLE_TIME UNLIMITED;
alter profile MONITORING_PROFILE LIMIT CONNECT_TIME UNLIMITED;
```

## 8 系统表

```sql
-- 查询列信息，包涵列注释
select * from user_tab_columns;

-- 查询表空间信息
select * from v$tablespace;--(v$视图):是动态性能视图，存在于controlfile中，数据库在mount状态下可以查询
select * from dba_tablespace;--(dba_数据字典):是静态视图，存在于数据库中，只能在open时查询
```

## 9. tips

### 9.1 NULL

字段为空时`like`和`not like`都查不出该记录。

优化方案1：`col like '%xx%'` 用 `instr(col,'xx')=0` 替代，缺点是只能进行简单模糊。

优化方案2：`col like '%a%b%'` 用 `nvl(col,'a') like '%a%b%'` 替代，前提是字符a没有出现在查询条件中。

优化方案3：分条件查询，`col is null or col like '%a%b%'`。

### 9.2 float不准确的问题

> float中13478.8显示的13478.8，实际存的是13478.799999999999，用to_char得出的是13478.799999999999，把float放到where后当条件极其不准确，建议使用number。

### 9.3 查看数字里面有没有其它字符

```sql
translate(dymjz,'@.0123456789','@')
```

### 9.4 获取分组序号

按col1分组按col2排序

```sql
ROW_NUMBER() OVER(PARTITION BY col1 order by col2)
```

### 9.5 创建dblink远程链接

```sql
create database link mylink
　connect to BDCDJ identified by BDCDJ
  using '(DESCRIPTION =
  (ADDRESS_LIST =
  (ADDRESS = (PROTOCOL = TCP)(HOST = 192.166.10.101)(PORT = 1521))
  )
  (CONNECT_DATA =
  (SERVER = DEDICATED)
  (SERVICE_NAME = orcl)
  )
  )';
  
create table aaamy as select * from aaamy@mylink where ajbh='BGES2015003038';
```

### 9.6 创建sequence序列

nomaxvalue：不设置最大值
nocycle：一直累加，不循环

```sql
create sequence slid_sy
minvalue 1
maxvalue 999999999999999999999999999
start with 201601010001
increment by 1
cache 20;
```

### 9.7 查询被锁住的表

```sql
-- 1.查询表锁情况
SELECT 
 S.USERNAME,
 DECODE(L.TYPE, 'TM', 'TABLE LOCK', 'TX', 'ROW LOCK', NULL) LOCK_LEVEL,
 O.OWNER,
 O.OBJECT_NAME,
 O.OBJECT_TYPE,
 S.SID,
 S.SERIAL#,
 S.TERMINAL,
 S.MACHINE,
 S.PROGRAM,
 S.OSUSER
  FROM V$SESSION S, V$LOCK L, DBA_OBJECTS O
 WHERE L.SID = S.SID
   AND L.ID1 = O.OBJECT_ID(+)
   AND S.USERNAME IS NOT NULL;

SELECT owner,object_name,session_id,locked_mode from(
select b.owner,b.object_name,a.session_id,a.locked_mode 
from v$locked_object a,dba_objects b 
where b.object_id = a.object_id
)
WHERE 
--AND owner='EDALDR'-- 用户
AND object_name like'%CHAMBER%';-- 表名

-- 查询那个用户那个语句锁定表
select b.owner,b.object_name,a.locked_mode,c.username,c.osuser,c.machine,c.sid,c.serial#,c.logon_time,d.sql_text from v$locked_object a,dba_objects b,v$session c,v$sql d where b.object_id=a.object_id and a.session_id=c.sid and c.sql_hash_value=d.hash_value(+) order by c.logon_time;

-- 2.查询SESSION ID 对应的 serial
SELECT * FROM v$session WHERE sid in(62);

-- 3.kill session（需要管理员权限）
alter system kill session '600,6122'; 
```

### 9.8 rownum和rowid

- rownum也就是伪列，在创建表的时候自动有的，每个表都有伪列，作条件的时候有几个注意点：只能等于1，只能大于0，可以小于任何数；

- 如果没有enable row movement，那么在ORACLE中一行数据的rowid在其生命周期中是不变的。

- 行迁移，意思就是：一个现存的行允许改变其rowid(物理存储地址)。通常情况下，数据行在分配了空间之后，行的rowid就固定了，即使以后行长度超出预留的空间，也不会将其移动。在长数据行的情况下，表会产生行链接，对于io来说行链接是不利的，为了性能，就需要将行调整为在单个物理存储地址下能保存行的所有信息，这需要行迁移。另外在flashback的时候，由于原始版本的行占据了相应的物理地址，所以也需要行迁移。其它的，比如说在收缩段空间的情况下，由于要把所有段数据向段前挤压，大部分行都需要改变其物理地址，也需要行迁移。

### 9.9 集合查询

为了合并多个SELECT语句的结果，可以使用集合操作符UNION、UNION  ALL、INTERSECT、MINUS。这些操作符多用于数据量比较大的数据库，运行速度快，称为合并查询，也叫集合查询。应用两个集合的相减、相交和相加时，是有严格要求的：

1. 两个集合的字段必须明确(用*就不行，报错)；
2. 字段类型和顺序相同(名称可以不同)，集合1的字段1是NUMBER，字段2是VARCHAR，那么集合2的字段1必须也是NUMBER，字段2必须是VARCHAR；
3. 不能排序，如果要对结果排序，可以在集合运算后，外面再套一个查询，然后排序。

| 关键字     | 说明                      |
| ---------- | ------------------------- |
| UNION      | 并集，自动去重            |
| UNION  ALL | 并集，允许重复值          |
| INTERSECT  | 交集                      |
| MINUS      | A-B，包含在A中不包含在B中 |

### 9.10 plsql developer修改查询结果

plsql developer在查询语句中加rowid可以编辑查询结果。

```sql
--其中col1、col2能编辑，col3不能编辑，因为table2中存在同名字段，这是plsql developer的bug，原因是col3被认为来源于table2
select a.col1,a.col2,a.col3,a.rowid from table2 a,table1 b where a.id=b.id;

--修改方案：
select a.col1,a.col2,a.col3,a.rowid from table1 a,table2 b where a.id=b.id;
```

### 9.11 update

```sql
-- 方法1
UPDATE table1 a SET a.C = (SELECT b.B FROM table2 b WHERE a.A = b.A) 
WHERE EXISTS (SELECT 1 FROM table2 b WHERE a.A = b.A);

-- 方法2
MERGE INTO table1 USING table2 
ON (table1.A = table2.A)    -- 条件是 A 相同
WHEN MATCHED THEN UPDATE SET table1.C = table2.B   -- 匹配的时候，更新
```

### 9.12 telnet和ping

可以用telnet命令来测试端口号是否正常打开还是关闭状态：telnet IP 端口。

telnet不是内部或外部命令,也不是可运行的程序：打开或关闭Windows功能，勾选“Telnet客户端”。

防止别人ping：gpedit.msc打开本地组策略，用户配置--管理模板--系统--阻止访问命令提示符，开启就可以


### 9.13 多行转一行

```sql
select r1 || '*' || r1 || '=' || r1 * r1 A,
       decode(r2, '', '', r2 || '*' || r1 || '=' || r2 * r1) b,
       decode(r3, '', '', r3 || '*' || r1 || '=' || r3 * r1) C,
       decode(r4, '', '', r4 || '*' || r1 || '=' || r4 * r1) D,
       decode(r5, '', '', r5 || '*' || r1 || '=' || r5 * r1) E,
       decode(r6, '', '', r6 || '*' || r1 || '=' || r6 * r1) F,
       decode(r7, '', '', r7 || '*' || r1 || '=' || r7 * r1) G,
       decode(r8, '', '', r8 || '*' || r1 || '=' || r8 * r1) H,
       decode(r9, '', '', r9 || '*' || r1 || '=' || r9 * r1) I
from (select level r1,
             lag(level, 1) over(order by level) r2,
             lag(level, 2) over(order by level) r3,
             lag(level, 3) over(order by level) r4,
             lag(level, 4) over(order by level) r5,
             lag(level, 5) over(order by level) r6,
             lag(level, 6) over(order by level) r7,
             lag(level, 7) over(order by level) r8,
             lag(level, 8) over(order by level) r9
      from dual connect by level < 10);

select * from emp start with empno=7698 connect by mgr=prior empno;

with temp as(select * from a union select * from b);

-- 将分组内的字段拼接起来
select djh,listagg(oid,',') within group(order by oid) from BDC_REGN_TDSYQ2 group by djh;
-- 当字段类型为NVARCHAR2时要使用wm_concat(to_char(oid))，否则会出现中文乱码
select djh,wm_concat(oid) from BDC_REGN_TDSYQ2 group by djh;
```

### 9.14 计算周岁

```sql
select trunc(months_between(to_date(to_char(sysdate, 'yyyymmdd'),'yyyymmdd'),to_date('19910918','yyyymmdd'))/12) from dual;
```

### 9.15 嵌套表和可变数组

```sql
-- 嵌套表
create or replace type emp_type as object(empno number(4),ename varchar2(10),job varchar2(9),mgr number(4),hiredate date,sal number(7,2),comm number(7,2));
create or replace type emp_tab_type as table of emp_type;

create table dept_and_emp
( deptno number(2) primary key,
  dname varchar2(14),
  loc varchar2(13),
  emps emp_tab_type
)nested table emps store as emps_nt;
alter table emps_nt add constraint emps_empno_unique unique(empno);

insert into dept_and_emp values(1,'Dept 1','shandong',emp_tab_type(emp_type(1,'zhangsan','kaifa',1,sysdate-100,8000,1),emp_type(2,'lisi','kaifa',2,sysdate-200,9000,2),emp_type(3,'wangwu','guanli',3,sysdate-300,10000,1)));

-- 可变数组
create or replace type emp_type as object(empno number(4),ename varchar2(10),job varchar2(9),mgr number(4),hiredate date,sal number(7,2),comm number(7,2));
create or replace type emp_tab_type as varray(50) of emp_type;

create table dept_and_emp
( deptno number(2) primary key,
  dname varchar2(14),
  loc varchar2(13),
  emps emp_tab_type
);

insert into dept_and_emp values(1,'Dept 1','shandong',emp_tab_type(emp_type(1,'zhangsan','kaifa',1,sysdate-100,8000,1),emp_type(2,'lisi','kaifa',2,sysdate-200,9000,2),emp_type(3,'wangwu','guanli',3,sysdate-300,10000,1)));
```

### 9.16 使数据库统计分析

数据库当前统计分析的数据不准

```sql
select 'analyze table '||TABLE_NAME||' compute statistics;' from  user_tables;

-- 按用户分析
exec dbms_stats.gather_schema_stats(ownname=>'scott',options=>'gather auto',estimate_percent=>dbms_stats.auto_sample_size,degree=>6);

-- 全库分析
exec dbms_stats.gather_system_stats('start'); -- 开始
exec dbms_stats.gather_system_stats('stop'); -- 结束
exec dbms_stats.gather_system_stats('interval',interval=>N); -- 一直工作N分钟
```

### 9.17 数据库去空格

数据库中的数据有时前后会多出空格，导致查询条件出现问题，以下语句生成批量去空格的语句。

```sql
select table_name,'update '||table_name||' set '||wm_concat(column_name||'=trim('||column_name||')')||';' from SYS.USER_TAB_COLUMNS 
where data_type in('NVARCHAR2','CHAR','CLOB','VARCHAR2') group by table_name;

select 'alter table '||table_name||' modify('||column_name||' NVARCHAR2('||data_length||'));' from SYS.USER_TAB_COLUMNS 
where data_type='VARCHAR2';
```

### 9.18 修改oracle字符集（ZHS16GBK）

```sql
-- 查询字符集
select userenv(‘language’) from dual;
select * from v$nls_parameters;

-- 重启数据库到mount状态
SQL> conn /as sysdba
SQL> shutdown immediate;
SQL> startup mount

-- 修改
SQL> ALTER SYSTEM ENABLE RESTRICTED SESSION;
SQL> ALTER SYSTEM SET JOB_QUEUE_PROCESSES=0;
SQL> ALTER SYSTEM SET AQ_TM_PROCESSES=0;
SQL> alter database open;
SQL> ALTER DATABASE CHARACTER SET ZHS16GBK;
第1行出现错误：
ORA-12712: new character set must be a superset of old character set
-- 提示我们：新字符集必须为旧字符集的超集，这时我们可以跳过超集的检查做更改：
SQL> ALTER DATABASE character set INTERNAL_USE ZHS16GBK;

-- 重启到正常状态
SQL> shutdown immediate;
SQL> startup
```

## 10. 函数和存储过程

存储过程默认是定义者权限，定义者能做什么存储过程能做什么，加authid current_user是执行者模式。

打印：DBMS_OUTPUT.PUT_LINE(g.zrzh);

### 10.1 FindAllDB：从全库查找值

新的

```sql
create or replace function FindAllDB(p_colz varchar2,p_coln varchar2:='',p_tabn varchar2:='',
isLike int:=0,isName int:=0) return integer is
  tabarry varchar2(255);
  valstr varchar2(100);
  mnu integer;
--CHAR VARCHAR2 NVARCHAR2
--FLOAT NUMBER LONG
--DATE TIMESTAMP(6)
--CLOB BLOB RAW

--user_tables.num_rows不准确，需要下面语句维护成真实值
--exec dbms_stats.gather_schema_stats(ownname=>'scott',options=>'gather auto',estimate_percent=>dbms_stats.auto_sample_size,degree=>6);
begin
  if isLike=0 then--字段值完全匹配
    valstr:='='''||p_colz||'''';
  else--字段值模糊匹配
    valstr:=' like ''%'||p_colz||'%''';
  end if;
  for g in(select table_name,column_name from user_tab_columns where data_type in('CHAR','VARCHAR2','NVARCHAR2') order by table_name,column_name) loop
    tabarry:='';
    if p_tabn is null or g.table_name like '%'||p_tabn||'%' then
      if (p_coln is not null and isName=0 and g.column_name=upper(p_coln)) or--字段名完全匹配
      (p_coln is not null and isName=1 and g.column_name like '%'||upper(p_coln)||'%') or--字段名模糊匹配
      (p_coln is null or isName not in(0,1)) then--字段名不匹配
        tabarry:='select count(1) from '||g.table_name||' where '||g.column_name||valstr;
      end if;
    end if;

    if tabarry is not null then
      execute immediate tabarry into mnu;
      if mnu>0 then
        DBMS_OUTPUT.PUT_LINE(replace(tabarry,'count(1)','*')||';--'||mnu);
      end if;
    end if;
  end loop;
return 1;
end;--select FindAllDB('201408250201','ywh') from dual;
```

旧的

```sql
create or replace function FindAllDB(coln varchar2,colz varchar2,isName int:=0,isLike int:=0) return integer is
  type my_arry is table of varchar2(255) index by binary_integer;
  tabarry my_arry;
  valstr varchar2(100);
  mnu integer;
--CHAR
--VARCHAR2
--NVARCHAR2
--FLOAT
--NUMBER
--DATE
--TIMESTAMP(6)
--LONG
--CLOB
--BLOB
--RAW

--user_tables.num_rows不准确，需要下面语句维护成真实值
--exec dbms_stats.gather_schema_stats(ownname=>'scott',options=>'gather auto',estimate_percent=>dbms_stats.auto_sample_size,degree=>6);
begin
  if isLike=0 then--字段值完全匹配
    valstr:='='''||colz||'''';
  else--字段值模糊匹配
    valstr:=' like ''%'||colz||'%''';
  end if;
  if isName=0 then--字段名完全匹配
    select mysql bulk collect into tabarry from (select distinct 'select count(1) from '||a.table_name||' where '||a.column_name||valstr mysql from user_tab_columns a,user_tables b
    where a.table_name=b.table_name and a.column_name=upper(coln) /*and b.num_rows>0 */and a.data_type in('CHAR','VARCHAR2','NVARCHAR2') order by mysql);
  elsif isName=1 then--字段名模糊匹配
    select mysql bulk collect into tabarry from (select distinct 'select count(1) from '||a.table_name||' where '||a.column_name||valstr mysql from user_tab_columns a,user_tables b
    where a.table_name=b.table_name and a.column_name like '%'||upper(coln)||'%' /*and b.num_rows>0 */and a.data_type in('CHAR','VARCHAR2','NVARCHAR2') order by mysql);
  else--字段名不匹配
    select mysql bulk collect into tabarry from (select distinct 'select count(1) from '||a.table_name||' where '||a.column_name||valstr mysql from user_tab_columns a,user_tables b
    where a.table_name=b.table_name /*and b.num_rows>0 */and a.data_type in('CHAR','VARCHAR2','NVARCHAR2') order by mysql);
  end if;

  for g in 1..tabarry.count loop
    execute immediate tabarry(g) into mnu;
    if mnu>0 then
      DBMS_OUTPUT.PUT_LINE(replace(tabarry(g),'count(1)','*')||';--'||mnu);
    end if;
  end loop;
return 1;
end;--select FindAllDB('ywh','201408250201') from dual;
```

### 10.2 GetFieldList：对比两表不同字段

还可以使用dblink跨数据库对比

```sql
create or replace function GetFieldList(table1 varchar2,table2 varchar2:=null,user1 varchar2:=null,user2 varchar2:=null,prefix varchar2:=null,rownum int:=10) return varchar2 authid current_user is
  v_rs varchar2(4000):='';
  v_ls varchar2(4000):='';
  type file_arry is table of varchar2(255) index by binary_integer;
  tab1_arry file_arry;
  tab2_arry file_arry;
  num1 integer:=0;
  num2 integer:=0;
  v_flag integer;
  v_sql varchar2(500);
begin
--select GetFieldList('table1') from dual;
--select GetFieldList('table1','table2','user1','@dblink','nvl(a.tab,b.tab)',5) from dual;
  if table1 is not null then
    if user1 is null then
      for a in(select column_name from user_tab_columns where table_name=upper(table1) order by column_id) loop
        tab1_arry(tab1_arry.count+1):=a.column_name;
      end loop;
    elsif user1 like '@%' then
      v_sql:='select column_name from user_tab_columns'||user1||' where table_name=upper('''||table1||''') order by column_id';
      execute immediate v_sql bulk collect into tab1_arry;
    else
      for a in(select column_name from all_tab_columns where table_name=upper(table1) and owner=upper(user1) order by column_id) loop
        tab1_arry(tab1_arry.count+1):=a.column_name;
      end loop;
    end if;
  end if;

  if table2 is not null then
    if user2 is null then
      for a in(select column_name from user_tab_columns where table_name=upper(table2) order by column_id) loop
        tab2_arry(tab2_arry.count+1):=a.column_name;
      end loop;
    elsif user2 like '@%' then
      v_sql:='select column_name from user_tab_columns'||user2||' where table_name=upper('''||table2||''') order by column_id';
      execute immediate v_sql bulk collect into tab2_arry;
    else
      for a in(select column_name from all_tab_columns where table_name=upper(table2) and owner=upper(user2) order by column_id) loop
        tab2_arry(tab2_arry.count+1):=a.column_name;
      end loop;
    end if;
  end if;

  num1:=tab1_arry.count;
  num2:=tab2_arry.count;
  if num1=0 and num2=0 then
    return null;
  elsif num1=0 and num2!=0 then
    for a in 1..num2 loop
      if prefix is null then
        v_rs:=v_rs||tab2_arry(a)||',';
      else
        v_rs:=v_rs||replace(prefix,'tab',tab2_arry(a))||',';
      end if;
      if mod(a,rownum)=0 and a!=num2 then
        v_rs:=v_rs||chr(10);
      end if;
    end loop;
    v_rs:=num2||','||substr(v_rs,1,length(v_rs)-1);
  elsif num1!=0 and num2=0 then
    for a in 1..num1 loop
      if prefix is null then
        v_rs:=v_rs||tab1_arry(a)||',';
      else
        v_rs:=v_rs||replace(prefix,'tab',tab1_arry(a))||',';
      end if;
      if mod(a,rownum)=0 and a!=num1 then
        v_rs:=v_rs||chr(10);
      end if;
    end loop;
    v_rs:=num1||','||substr(v_rs,1,length(v_rs)-1);
  else
    num1:=0;
    num2:=0;
    for a in 1..tab1_arry.count loop
      v_flag:=0;
      for b in 1..tab2_arry.count loop
        if tab1_arry(a)=tab2_arry(b) then
          if prefix is null then
            v_rs:=v_rs||tab1_arry(a)||',';
          else
            v_rs:=v_rs||replace(prefix,'tab',tab1_arry(a))||',';
          end if;
          num1:=num1+1;
          v_flag:=1;
          if mod(num1,rownum)=0 then
            v_rs:=v_rs||chr(10);
          end if;
          exit;
        end if;
      end loop;
      if v_flag=0 then
        if prefix is null then
          v_ls:=v_ls||tab1_arry(a)||',';
        else
          v_ls:=v_ls||replace(prefix,'tab',tab1_arry(a))||',';
        end if;
        num2:=num2+1;
        if mod(num2,rownum)=0 then
          v_ls:=v_ls||chr(10);
        end if;
      end if;
    end loop;
    if v_rs like '%'||chr(10) then
      v_rs:=num1||','||substr(v_rs,1,length(v_rs)-2);
    else
      v_rs:=num1||','||substr(v_rs,1,length(v_rs)-1);
    end if;
    if num2>0 then
      if v_ls like '%'||chr(10) then
        v_ls:='表1：'||num2||','||substr(v_ls,1,length(v_ls)-2);
      else
        v_ls:='表1：'||num2||','||substr(v_ls,1,length(v_ls)-1);
      end if;
      v_rs:=v_rs||chr(10)||chr(10)||v_ls;
      v_ls:='';
      num2:=0;
    end if;

    for b in 1..tab2_arry.count loop
      v_flag:=0;
      for a in 1..tab1_arry.count loop
        if tab1_arry(a)=tab2_arry(b) then
          v_flag:=1;
          exit;
        end if;
      end loop;
      if v_flag=0 then
        if prefix is null then
          v_ls:=v_ls||tab2_arry(b)||',';
        else
          v_ls:=v_ls||replace(prefix,'tab',tab2_arry(b))||',';
        end if;
        num2:=num2+1;
        if mod(num2,rownum)=0 then
          v_ls:=v_ls||chr(10);
        end if;
      end if;
    end loop;
    if num2>0 then
      if v_ls like '%'||chr(10) then
        v_ls:='表2：'||num2||','||substr(v_ls,1,length(v_ls)-2);
      else
        v_ls:='表2：'||num2||','||substr(v_ls,1,length(v_ls)-1);
      end if;
      v_rs:=v_rs||chr(10)||chr(10)||v_ls;
    end if;
  end if;
  return v_rs;
exception when others then
  return null;
end;
```

### 10.3 split：分割字符串

```sql
create or replace type type_array as table of varchar2(4000);

create or replace function split(p_list varchar2,p_sep varchar2:=',',p_null int:=1)  return type_array pipelined is
   v_idx pls_integer;
   v_list varchar2(4000):=p_list;
   v_str varchar2(4000);
begin--默认忽略空字符
  loop
    v_idx:=instr(v_list,p_sep);
    if v_idx> 0 then
      v_str:=substr(v_list,1,v_idx-1);
      if v_str is not null or p_null!=1 then
        pipe row(v_str);
      end if;
      v_list:=substr(v_list,v_idx+length(p_sep));
    else
      if v_list is not null or p_null!=1 then
        pipe row(v_list);
      end if;
      exit;
    end if;
  end loop;
  return;
end;
```

### 10.4 字符转数字和日期

```sql
--说明：截取异常法不报错，但直接粘贴到数据库还是会报错，猜测保存时没有用转换函数，这时建议用其他方法

--方法1：判断是不是数字组成
create or replace function is_num(numvar varchar2) return int is
   v_str varchar2(1000);
begin
   v_str:=trim(numvar);
   if length(v_str)=0 then
      return 2;
   else
      v_str:=translate(v_str,'.0123456789','.');
      if v_str='.' or v_str='+.' or v_str='-.' or v_str is null then
         return 1;
      else
         return 0;
      end if;
   end if;
end;

--方法2：截取to_number的异常
create or replace function is_num(numvar varchar2) return int is
begin
  return to_number(trim(numvar));
  exception when others then
  return 0;
end;

--方法3：正则表达式
select * from 表 where not regexp_like(列名,'^[[:digit:]]+$');

--方法4：ascii值判断是否是整数
create or replace function is_num(p_str varchar2) return boolean is
k        number;
flag     boolean;
v_length number;
begin
  flag:=true;
  select length(p_str) into v_length from dual;
  for i in 1..v_length loop
    k:=ascii(substr(p_str,i,1));
    if k<48 or k>57 then--通过ASCII码判断是否0-9的数字
      flag:=false;
      exit;
    end if;
  end loop;
return flag;
end;

--转数字，格式不对不报错
create or replace function to_num(numvar varchar2) return number is
begin
  return to_number(trim(numvar));
  exception when others then
  return null;
end;

--判断是否由字母或数字组成
--select isComposition('Acw53','0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz') from dual;
create or replace function isComposition(str VARCHAR2,zc VARCHAR2) return int is
 v_one char;
 v_str varchar2(1000);
begin
 if str is null or zc is null then
   return 2;
 else
   v_one:=substr(zc,1,1);
   v_str:=translate(str,zc,v_one);
   if v_str=v_one or v_str is null then
     return 1;
   else
     return 0;
   end if;
 end if;
end;

--判断是否能转日期
create or replace function is_date(datevar varchar2,typevar varchar2:='yyyy/mm/dd hh24:mi:ss') return int is
  datetmp date;
begin
  select to_date(datevar,typevar) into datetmp from dual;
  return 1;
  exception when others then
  return 0;
end;

--转日期，格式不对不报错
create or replace function to_date(datevar varchar2,typevar varchar2:='yyyy/mm/dd hh24:mi:ss') return date is
begin
  return to_date(trim(datevar),typevar);
  exception when others then
  return null;
end;
```

### 10.5 备份所有表到临时库

```sql
create or replace procedure TBDLS(v_from varchar2,v_to varchar2,v_lx varchar2) authid current_user is
m_from varchar2(50):='BDCDJ';
m_to varchar2(50):='BDCDJ_LS';
m_lx varchar2(50):='1';--1表示同库，2表示远程连接
m_sql varchar2(300);
begin
  if v_from!='' then
    m_from:=v_from;
  end if;
  if v_to!='' then
    m_to:=v_to;
  end if;
  if v_lx!='' then
    m_lx:=v_lx;
  end if;

  for a in(select * from all_tables where owner=m_to order by table_name) loop
  begin
    m_sql:='truncate table '||a.table_name;
    EXECUTE IMMEDIATE m_sql;
    if m_lx='1' then
      m_sql:='insert into '||a.table_name||' select * from '||m_from||'.'||a.table_name;
    elsif m_lx='2' then
      m_sql:='insert into '||a.table_name||' select * from '||a.table_name||'@'||m_from;
    end if;
    EXECUTE IMMEDIATE m_sql;
  exception when others then
    continue;
  end;
  end loop;
end TBDLS;
```

### 10.6 拼接函数

```sql
create or replace type type_row as object(a varchar2(4000),b varchar2(4000));--定义行对象
create or replace type type_table as table of type_row;--定义表对象

create or replace function tab_group(p_table varchar2,p_cola varchar2:='',p_colb varchar2:='',p_where varchar2:='',
p_sep varchar2:=',',p_dis varchar2:='distinct ') return type_table pipelined is
  type col_arry is table of varchar2(4000) index by binary_integer;
  v_arry1 col_arry;
  v_arry2 col_arry;
  v_sql varchar2(800);
  v_colb varchar2(4000);
  v_num int;
  v_row type_row;
begin
  if p_cola is null then--分组字段为空
    v_sql:='select '||p_dis||p_colb||' from '||p_table||' a where ('||p_where||') and '||p_colb||' is not null order by '||p_colb;
    execute immediate v_sql bulk collect into v_arry2;

    v_num:=v_arry2.count;
    if v_num!=0 then
      v_colb:=v_arry2(1);
      for a in 2..v_arry2.count loop
        v_colb:=v_colb||p_sep||v_arry2(a);
      end loop;
      v_row:=type_row(null,v_colb);
      pipe row(v_row);
    end if;
  else--分组字段不为空
    v_sql:='select distinct nvl('||p_cola||',''aanull'') cola from '||p_table||' a where ('||p_where||') order by cola';
    execute immediate v_sql bulk collect into v_arry1;
    v_num:=v_arry1.count;
    if v_num!=0 then
      for a in 1..v_arry1.count loop
        v_sql:='select '||p_dis||p_colb||' from '||p_table||' a where ('||p_where||') and nvl('||p_cola||',''aanull'')='||v_arry1(a)||' and '||p_colb||' is not null order by '||p_colb;
        execute immediate v_sql bulk collect into v_arry2;
        v_num:=v_arry2.count;
        if v_num=0 then
          v_row:=type_row(v_arry1(a),null);
          pipe row(v_row);
        else
          v_colb:=v_arry2(1);
          for b in 2..v_arry2.count loop
            v_colb:=v_colb||p_sep||v_arry2(b);
          end loop;
          v_row:=type_row(v_arry1(a),v_colb);
          pipe row(v_row);
        end if;
      end loop;
    end if;
  end if;
  return;
end;
```

### 10.7 身份证相关

```sql
--15位转18位
create or replace function IDCARD15TO18(card varchar2) return varchar2 is
  type TIARRAY is table of integer;
  type TCARRAY is table of char(1);
  W TIARRAY:=TIARRAY(7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2);
  A TCARRAY:=TCARRAY('1', '0', 'X', '9', '8', '7', '6', '5', '4', '3', '2');
  v_card  varchar2(30);
  results varchar2(18);
  num integer:=0;
begin
  v_card:=trim(card);
  if length(v_card)!=15 then
    return card;
  end if;
  results:=substr(v_card,1,6)||'19'||substr(v_card,7);

  for i in 1.. 17 loop
    num:=num+to_number(substr(results,i,1))*W(i);
  end loop;
  num:=num mod 11;
  results:=results||A(num+1);
  return results;
exception when others then
  return card;
end;

--验证身份证是否正确(正则表达式法)
create or replace function checkidcard(p_idcard varchar2) return int is
type TIARRAY is table of integer;
type TCARRAY is table of char(1);
v_tiarry   TIARRAY:=TIARRAY(7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2);
v_tcarry   TCARRAY:=TCARRAY('1', '0', 'X', '9', '8', '7', '6', '5', '4', '3', '2');
v_card     varchar2(18);
v_areacode varchar2(200):='11,12,13,14,15,21,22,23,31,32,33,34,35,36,37,41,42,43,44,45,46,50,51,52,53,54,61,62,63,64,65,71,81,82,91,';
v_date     int;
v_regstr   varchar2(200);
v_sum      number:=0;
begin
  v_card:=replace(trim(p_idcard),'x','X');
  if instrb(v_areacode,substrb(v_card,1,2)||',')=0 then--省份代码是否正确
    return 0;
  end if;
  case lengthb(v_card) when 15 then--15位
    v_date:=to_number(substrb(v_card,7,2))+1900;
    if mod(v_date,400)=0 or (mod(v_date,100)<>0 and mod(v_date,4)=0) then--闰年
      v_regstr:='^[1-9][0-9]{5}[0-9]{2}((01|03|05|07|08|10|12)(0[1-9]|[1-2][0-9]|3[0-1])|(04|06|09|11)(0[1-9]|[1-2][0-9]|30)|02(0[1-9]|[1-2][0-9]))[0-9]{3}$';
    else
      v_regstr:='^[1-9][0-9]{5}[0-9]{2}((01|03|05|07|08|10|12)(0[1-9]|[1-2][0-9]|3[0-1])|(04|06|09|11)(0[1-9]|[1-2][0-9]|30)|02(0[1-9]|1[0-9]|2[0-8]))[0-9]{3}$';
    end if;
    if regexp_like(v_card,v_regstr) then--6位生日是否是日期
      return 1;
    else
      return 0;
    end if;
  when 18 then--18位
    v_date:=to_number(substrb(v_card,7,4));
    if mod(v_date,400)=0 or (mod(v_date,100)<>0 and mod(v_date,4)=0) then
      v_regstr:='^[1-9][0-9]{5}(19|20)[0-9]{2}((01|03|05|07|08|10|12)(0[1-9]|[1-2][0-9]|3[0-1])|(04|06|09|11)(0[1-9]|[1-2][0-9]|30)|02(0[1-9]|[1-2][0-9]))[0-9]{3}[0-9Xx]$';
    else
      v_regstr:='^[1-9][0-9]{5}(19|20)[0-9]{2}((01|03|05|07|08|10|12)(0[1-9]|[1-2][0-9]|3[0-1])|(04|06|09|11)(0[1-9]|[1-2][0-9]|30)|02(0[1-9]|1[0-9]|2[0-8]))[0-9]{3}[0-9Xx]$';
    end if;
    if regexp_like(p_idcard,v_regstr) then
      for i in 1.. 17 loop
      v_sum:=v_sum+to_number(substrb(v_card,i,1))*v_tiarry(i);
      end loop;
      v_sum:=v_sum mod 11;
      if v_tcarry(v_sum+1)=upper(substrb(v_card,18,1)) then--第18位校验码由前面17位计算得出，判断是否正确
        return 1;
      else
        return 0;
      end if;
    else
      return 0;
    end if;
  else
    return 0;--身份证号码位数不对
  end case;
exception when others then
  return 0;
end;
--验证身份证是否正确(非正则表达式法)
create or replace function checkidcard(p_idcard varchar2) return int is
type TIARRAY is table of integer;
type TCARRAY is table of char(1);
v_tiarry   TIARRAY:=TIARRAY(7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2);
v_tcarry   TCARRAY:=TCARRAY('1', '0', 'X', '9', '8', '7', '6', '5', '4', '3', '2');
v_card     varchar2(18);
v_areacode varchar2(200):='11,12,13,14,15,21,22,23,31,32,33,34,35,36,37,41,42,43,44,45,46,50,51,52,53,54,61,62,63,64,65,71,81,82,91,';
v_date     varchar2(6);
v_sum      number:=0;
-----内嵌函数
function is_num(p_str varchar2) return boolean is
k        number;
v_length number;
begin
  select length(p_str) into v_length from dual;
  for i in 1..v_length loop
    k:=ascii(substr(p_str,i,1));
    if k<48 or k>57 then--通过ASCII码判断是否0-9的数字
      return false;
    end if;
  end loop;
return true;
end;
-----内嵌函数
function is_date(p_date varchar2) return boolean is
v_year          number;
v_month         number;
v_day           number;
v_isLeapYear    boolean;
begin
  v_year :=to_number(substr(p_date,1,4));
  v_month:=to_number(substr(p_date,5,2));
  v_day  :=to_number(substr(p_date,7,2));
  --判断是否为闰年
  if (mod(v_year,400)=0) or (mod(v_year,100)<>0 and mod(v_year,4)=0) then
    v_isLeapYear:=true;
  else
    v_isLeapYear:=false;
  end if;
  --判断月份
  if v_month<1 or v_month>12 then
    return false;
  end if;
  --判断日期
  if v_month in (1,3,5,7,8,10,12) and (v_day<1 or v_day>31) then
    return false;
  end if;
  if v_month in (4,6,9,11) and (v_day<1 or v_day>30) then
    return false;
  end if;
  if v_month=2 then
    if v_isLeapYear and (v_day<1 or v_day>29) then
      return false;
    end if;
    if v_isLeapYear=false and (v_day<1 or v_day>28) then
      return false;
    end if;
  end if;
return true;
end;
begin
  v_card:=replace(trim(p_idcard),'x','X');
  if instrb(v_areacode,substrb(v_card,1,2)||',')=0 then--省份代码是否正确
    return 0;
  end if;
  case lengthb(v_card) when 15 then--15位
    if is_num(v_card)!=true then--是否数字组成
      return 0;
    end if;
    v_date:='19'||substrb(v_card,7,4);
    if is_date(v_date) then--6位生日是否是日期
      return 1;
    else
      return 0;
    end if;
  when 18 then--18位
    if is_num(substrb(v_card,1,17))!=true then--前17位是否数字组成
      return 0;
    end if;
    v_date:=substrb(v_card,7,6);
    if is_date(v_date)=false then--6位生日是否是日期
      return 0;
    end if;
    for i in 1.. 17 loop
      v_sum:=v_sum+to_number(substrb(v_card,i,1))*v_tiarry(i);
    end loop;
    v_sum:=v_sum mod 11;
    if v_tcarry(v_sum+1)=upper(substrb(v_card,18,1)) then--第18位校验码由前面17位计算得出，判断是否正确
      return 1;
    else
      return 0;
    end if;
  else
    return 0;--身份证号码位数不对
  end case;
exception when others then
  return 0;
end;
```

### 10.8 管道函数

```sql
create or replace function f_pipeline_test return type_array pipelined is
begin
  for i in 1..10 loop
    pipe row('Iteration ' || i || ' at ' || systimestamp);
    sys.dbms_lock.sleep(1);
  end loop;
  pipe row('All done!');
return;
end;

set arraysize 1
select * from table(f_pipeline_test);
```

### 10.9 游标

```sql
一、隐式游标
在PL/SQL中使用DML语句时自动创建隐式游标，其名为SQL
begin
  update student set sage=sage+10;
  if sql%FOUND then
    dbms_output.put_line('这次更新了'||sql%rowcount);
  else
    dbms_output.put_line('一行也没有更新');
  end if;
end;

二、显式游标
声明cursor stu_cur(input_sno number) is select sname from student where sno=:input_sno;
打开open stu_cur(123);
取值
1.fetch stu_cur into sno;
  while stu_cur%found loop
    update student set sage=sage+10 where current of stu_cur;--用游标更新行
    fetch stu_cur into sno;
  end loop;
2.for stu in stu_cur loop
  end loop;
3.loop
    fetch stu_cur into sno;
    exit when stu_cur%notfound;
  end loop;
4.fetch stu_cur bulk collect into ename_table;--ename_table是table类型
关闭close stu_cur;

三、动态游标
声明      TYPE <ref_cursor_name> IS REF CURSOR [RETURN <return_type>];
          ref_cur ref_cursor_name;
打开      Open ref_cur For select name from stu where no=:no;
获取记录  Fatch ref_cur InTo v_name using 13;
关闭      Close ref_cur;

四、动态执行
v_sql:='select * from emp where empno=:id';
execute immediate v_sql into emp_rec using &1;
```

## 11. 监听问题

Oracle 11g ORA-12514:TNS:监听程序当前无法识别连接描述符中请求的服务
解决过程：

1. 找到listener.ora监听文件，具体位置：D:\app\Administrator\product\11.2.0\dbhome_1\NETWORK\ADMIN\listener.ora

2. 在lisener.ora文件中找到

    ```tex
    (SID_DESC =
      (SID_NAME = CLRExtProc)
      (ORACLE_HOME = D:\app\Administrator\product\11.2.0\dbhome_1)
      (PROGRAM = extproc)
      (ENVS = "EXTPROC_DLLS=ONLY:D:\app\Administrator\product\11.2.0\dbhome_1\bin\oraclr11.dll")
    )
    ```
    
    将下面的一段内容copy进去，并适当修改。(红字部分为你的SID，其中GLOBAL_DBNAME为全局数据库名，可以与SID不同)
    
    ```tex
    (SID_DESC = 
      (GLOBAL_DBNAME = ORAC11) 
      (ORACLE_HOME = D:\app\Administrator\product\11.2.0\dbhome_1) 
      (SID_NAME = ORAC11) 
    ) 
    ```
    
3. 保存listener.ora文件，关闭并重新启动监听程序。

    lsnrctl stop   // 关闭

    lsnrctl start  // 启动

4. 此时，用正常的用户去连接双出现新的错误。

   ORA-27101: shared memory realm does not exist
5. 启动打开目录：D:\app\Administrator\admin\orac11\pfile，会发现里面有一个文件：init.ora.1052011103553，这是Oracle最后一次成功启动时备份的启动文件。
6. sqlplus /nolog，
    create spfile from pfile='D:\app\Administrator\admin\orac11\pfile\init.ora.1052011103553'
    startup  // 启动数据库。
7. 一切恢复正常。

## 12. Flashback Database闪回数据库

```sql
--查看闪回是否开启
select flashback_on from v$database;

-----开启闪回前必须开启归档模式
--查看闪回参数：show parameter recovery
--修改闪回目录：alter system set db_recovery_file_dest='/u01/app/oracle/flashback' scope=spfile;
shutdown immediate;
startup mount;
alter database flashback on;
alter database open;

--按时间闪回3分钟前
select * from AAA as of timestamp sysdate-3/1440 where id<3;
--按SCN闪回
select * from AAA as of SCN 1392651 where id<3;

--查看SCN和时间的对应关系
select scn,to_char(time_dp,'yyyy/mm/dd hh24:mi:ss') from SYS.SMON_SCN_TIME order by scn desc;
--SCN和时间的相互转换
select TO_CHAR(SCN_TO_TIMESTAMP(1393920),'yyyy/mm/dd hh24:mi:ss') from dual;
select TIMESTAMP_TO_SCN(sysdate) from dual;

--查询当前SCN
GRANT EXECUTE ON DBMS_FLASHBACK TO BMFDATA;
select DBMS_FLASHBACK.GET_SYSTEM_CHANGE_NUMBER from dual;

GRANT SELECT ON V_$DATABASE TO BMFDATA;
select CURRENT_SCN from V_$DATABASE;--需管理员权限

--查询最小SCN
select SCN_WRP*4294967296+SCN_BAS from SYS.SMON_SCN_TIME where TIME_MP=(select MIN(TIME_MP) from SYS.SMON_SCN_TIME);
```

## 13. 归档日志

```sql
一、更改Oracle为归档模式
shutdown immediate;
startup mount;
alter database archivelog;   alter database noarchivelog;
alter database open;
alter system archive log start;(启动自动归档)

查看归档模式信息：archvie log list;     select name,log_mode from v$database;

二、更改归档目录
1.查看参数db_recovery_file_dest
show parameter db_recovery;
select * from v$recovery_file_dest;

2.更改归档日志目录
语法：alter system set 参数=值 scope=spfile;
alter system set db_recovery_file_dest='D:\oracle\archivelog' scope=spfile;

三、更改归档日志大小
1.查看参数'db_recovery_file_dest_size'值
show parameter db_recov;

2.更改参数'db_recovery_file_dest_size'值大小
alter system set db_recovery_file_dest_size=30G scope=spfile;







1.启用自动归档: LOG_ARCHIVE_START=TRUE
归档模式下,日志文件组不允许被覆盖(重写),当日志文件写满之后,如果没有进行手动归档,那么系统将挂起,知道归档完成为止.这时只能读而不能写.
 
运行过程中关闭和重启归档日志进程
SQL>ARCHIVE LOG STOP
SQL>ARCHIVE LOG START
 
2.手动归档: LOG_ARCHIVE_START=FALSE
归档当前日志文件
SQL>ALTER SYSTEM ARCHIVE LOG CURRENT;
 
归档序号为052的日志文件
SQL>ALTER SYSTEM ARCHIVE LOG SEQUENCE 052;
 
归档所有日志文件
SQL>ALTER SYSTEM ARCHIVE LOG ALL;
 
改变归档日志目标
SQL>ALTER SYSTEM ARCHIVE LOG CURRENT TO '&PATH';

3.如果归档过程会消耗大量的时间,那么可以启动多个归档进程,这是个动态参数,可以用ALTER SYSTEM动态修改.
SQL>ALTER SYSTEM SET LOG_ARCHIVE_MAX_PROCESSES=10;

与归档进程有关的动态性能视图：v$bgprocess,v$archive_processes
```

## 14. 高级复制

两个数据库之间复制，有时间有需求再研究。

```sql
1.数据库是否支持高级复制功能：sys查看v$option视图，Advanced replication为TRUE则支持

2.修改db_domain：alter system set db_domain='zuotest.com.cn' scope=spfile;（scope=spfile表示数据库重启生效）
show parameter name

3.设置global_names=true：alter system set global_names=true;

4.确认两台数据库之间可以互相访问，在tnsnames.ora里设置数据库连接字符串。

5.改数据库全局名称，建公共的数据库链接。
alter database rename global_name to orcl.zuotest.com.cn;
alter database rename global_name to TXNEW.zuotest.com.cn;

create public database link TXNEW.zuotest.com.cn using 'TXNEW';
select * from global_name@TXNEW.zuotest.com.cn;
create public database link orcl.zuotest.com.cn using 'shenzhen';
select * from global_name@orcl.zuotest.com.cn;

6.建立管理数据库复制的用户repadmin，并赋权。(两个数据库都要执行如下语句)
create user repadmin identified by repadmin default tablespace users temporary tablespace temp;
execute dbms_defer_sys.register_propagator('repadmin');
grant execute any procedure to repadmin;  
execute dbms_repcat_admin.grant_admin_any_repgroup('repadmin');
execute dbms_repcat_admin.grant_admin_any_schema(username => '"REPADMIN"'); 
grant comment any table to repadmin;  
grant lock any table to repadmin;
grant select any dictionary to repadmin;

7.在数据库复制的用户repadmin下创建私有的数据库链接。
create database link TXNEW.zuotest.com.cn connect to repadmin identified by repadmin;
select * from global_name@TXNEW.zuotest.com.cn;

create database link orcl.zuotest.com.cn connect to repadmin identified by repadmin;
select * from global_name@orcl.zuotest.com.cn;

8.创建或选择实现数据库复制的用户和对象，给用户赋权，数据库对象必须有主关键字。
create user zuotest identified by zuotest default tablespace users temporary tablespace temp;  
grant connect, resource to zuotest;  
grant execute on sys.dbms_defer to zuotest;

create table dept  
(deptno number(2),  
dname varchar2(14),  
loc varchar2(13) );
alter table dept add (constraint dept_deptno_pk primary key (deptno));

create sequence dept_no increment by 1 start with 1 maxvalue 44 cycle nocache;
create sequence dept_no increment by 1 start with 45 maxvalue 99 cycle nocache;

insert into dept values (dept_no.nextval,'accounting','new york');  
insert into dept values (dept_no.nextval,'research','dallas');  
commit;
insert into dept values (dept_no.nextval,'sales','chicago');  
insert into dept values (dept_no.nextval,'operations','boston');  
commit;

9.创建要复制的组scott_mg，加入数据库对象，产生对象的复制支持
execute dbms_repcat.create_master_repgroup('zuotest_mg');
execute dbms_repcat.create_master_repobject(sname=>'zuotest',oname=>'dept', type=>'table',use_existing_object=>true,gname=>'zuotest_mg');
execute dbms_repcat.generate_replication_support('zuotest','dept','table');

select gname, master, status from dba_repgroup;  
select * from dba_repobject;

10.用repadmin身份登录ORCL数据库，创建主复制节点
execute dbms_repcat.add_master_database(gname=>'ZUOTEST_MG',master=>'TXNEW.zuotest.com.cn',use_existing_objects=>true, copy_rows=>false, propagation_mode => 'asynchronous');

11.使同步组的状态由停顿(quiesced )改为正常(normal)
execute dbms_repcat.resume_master_activity('zuotest_mg',false);
select gname, master, status from dba_repgroup;查看状态

12.创建复制数据库的时间表，我们假设用固定的时间表：10分钟复制一次。（前面两个在主节点执行）
begin  
  dbms_defer_sys.schedule_push(destination => 'TXNEW.zuotest.com.cn',
  interval => 'sysdate + 10/1440',next_date => sysdate);  
end;

begin  
  dbms_defer_sys.schedule_purge(next_date => sysdate,interval => 'sysdate + 10/1440',  
  delay_seconds => 0,rollback_segment => ''); 
end;

begin  
  dbms_defer_sys.schedule_push(destination => 'orcl.zuotest.com.cn',  
  interval => 'sysdate+10/1440',next_date => sysdate);  
end;

begin  
  dbms_defer_sys.schedule_purge(next_date => sysdate,interval => 'sysdate+10/1440', 
  delay_seconds => 0,rollback_segment => '');  
end;
```

## 15. RMAN

```sql
进入RMAN：RAMN TARGET / LOG D:\log.txt
或RMAN
  CONNECT TARGET /

1.整库备份
backup database format 'F:\oraclebackup\bak_%U';
查看list backup of database;

2.表空间备份
backup tablespace users;
查看list backup of tablespace users;

3.删除备份
delete backupset 3;
加noprompt跳过确认

4.备份数据文件
查询数据字典select file_id,file_name from dba_data_files;
查询动态性能视图V$DATABASE
backup datafile 'E:\Oracle\oradata\orcl\USERS01.DBF' format 'F:\oraclebackup\bak_%U';
或backup datafile 4 format 'F:\oraclebackup\bak_%U';
查看list backup of datafile 4;

5.备份控制文件
backup current controlfile;
任何备份可以加include current controlfile字句；备份SYSTEM会自动备份控制文件；设置configure controlfile autobackup on后做任何备份都会备份控制文件。
查看list backup of controlfile;

6.归档文件备份
backup archivelog all/until/scn/time/sequence;
执行backup时加plus archivelog会对当前Redolog归档并备份;
加delete all input参数会在备份后删除已备份的归档日志。
查看list backup of archivelog all;

7.备份初始化参数文件
备份控制文件时会自动备份初始化参数文件，不需单独备份SPFILE。初始化参数文件容易重建，其实不必特意备份。
backup spfile;

8.备份备份集
backup backupset all;   backup backupset n;
delete input参数删除已备份的备份集

9.show all;显示所有配置参数，#default表示参数未被修改
show controlfile autobackup;显示指定配置参数

10.列出所有备份list backup;
列出数据库当前所有归档list archivelog all;
列出所有无效备份list expired backup;

11.删除过期备份delete obsolete;
删除无效备份delete expired backup;--需先执行crosscheck查找无效备份（备份对应的数据文件损坏或丢失）
删除expired副本delete expired copy;
删除特定备份集delete backupset 19;
删除特定备份片段delete bdckuppiece 'F:\oraclebackup\DEMO_19.bak';
删除所有备份集delete backup;
删除特定映像副本delete database copy 'F:\oraclebackup\DEMO_19.bak';
删除所有映像副本delete copy;

12.查看7天前的数据库模式（必须连接到catalog数据库）report schema at time 'sysdate-7';
查看所有需要备份的文件report need backup;
查看指定表空间是否需要备份report need backup tablespace system;
查看过期备份report obsolete;

13.crosscheck执行检查，并将该对象标记为available(有效)和expired(无效)
crosscheck archivelog all;
crosscheck backup;

14.change修改状态，与crosscheck不同附带delete子句
将备份集修改为不可用change backupset n unavailable;
修改指定表空间备份集change backup of tablespace users unavailable;
修改指定归档文件状态change archivelog logseq=n unavailable;
删除某个归档文件change archivelog logseq=n delete;

15.非归档模式下只能关闭数据库才能进行增量备份
backup incremental level=0 database;
backup incremental level=1 datafile 5;
backup incremental level=1 tablespace users;

启用块修改跟踪alter database enable block change tracking using file'/LOACTION/TRK_FILENAME';
禁用块修改跟踪alter database disable block change tracking;
查询select status from V$BLOCK_CHANGE_TRACKING;

16.基于时间的备份保留策略configure retention policy to recovery window of n days;
基于冗余的备份保留策略configure retention policy to redundancy n;
无保留策略configure retention policy to none;
control_file_record_keep_time设置控制文件中记录最少保存时间，默认为7。用v$controlfile_record_section查看可存储记录和已存储记录数。
超出备份保留策略标记为obsolete；手工删除文件，执行crosscheck后标记为expired。

17.指定备份文件的复制份数backup copies 3 database;
run{set backup copies 2;
backup device type disk format 'D:\backup1\%U','D:\backup2\%U' tablespace users,sales;
}
设置指定设备默认备份数量configure default device type to disk;
                        configure datafile backup copies for device type disk to 2;
                        configure archivelog backup copies for decice type disk to 2;

18.设置备份集标签backup tablespace users tag tbs_usersbak;
设置备份片段大小
run{allocate channel c1 device type disk maxpiecesize=10m format 'f:\oracle\backup\bak_%U';
backup tablespace system;
}
```

## 16. 优化器

```sql
1、查看优化器
show parameter optimizer_mode;

2、修改优化器(first_rows_1000,first_rows_100,first_rows_10,first_rows_1,first_rows,all_rows,choose,rule)
系统级alter system set optimizer_mode=rule scope=both;
会话级alter session set optimizer_mode=first_rows_100;
语句级通过hints实现select /*+ rule */ * from dba_objects where rownum <= 10;

3、arraysize是客户段的一个参数，定义了一次返回到客户端的行数，当扫描了arraysize行后，停止扫描，返回数据，然后继续扫描。
查询show arraysize
暂时修改set arraysize 100
永久修改E:\Oracle\product\11.2.0\dbhome_1\sqlplus\admin\glogin.sql，末尾加set arraysize 100

查看table占用的blocks数select owner,extents,segment_name,blocks from dba_segments where segment_name='DAVE' and owner='DAVE';
                       SELECT prerid,COUNT(1) rid FROM (SELECT SUBSTR(ROWID,1,15) prerid,ROWID rid FROM dave) GROUP BY prerid;

查询extents：select extent_id,block_id,blocks from dba_extents where owner='DAVE' and segment_name='DAVE';

识别低效率的SQL语句select executions,disk_reads,buffer_gets,round((buffer_gets-disk_reads)/buffer_gets,2) hit_radio,round(disk_reads/executions,2) reads_per_run,sql_text from v$sqlarea where executions>0 and buffer_gets>0 and (buffer_gets-disk_reads)/buffer_gets<0.8;

4、设置autotrace
SET AUTOTRACE OFF            此为默认值，即关闭Autotrace 
SET AUTOTRACE ON EXPLAIN     只显示执行计划
SET AUTOTRACE ON STATISTICS  只显示执行的统计信息
SET AUTOTRACE ON             包含2,3两项内容
SET AUTOTRACE TRACEONLY      与ON相似，但不显示语句的执行结果

5、EXPLAIN PLAN的使用
EXPLAIN PLAN FOR sql语句;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('PLAN_TABLE'));
或select * from table(dbms_xplan.display);

6、PL/SQL Developer按F5查看语句执行效率
```

## 17. dbms_job

```sql
--添加job，从明天开始每天早上9点执行
declare job number;
begin
  dbms_job.submit(job,'XYBBLC;',sysdate,'trunc(sysdate)+1+9/24');
end;
--添加job，15分钟执行一次
declare job number;
begin
  dbms_job.submit(job,'XYBBLC;',sysdate,'sysdate+1/24/4');
end;

--修改job
begin
  dbms_job.change(43,null,sysdate,'sysdate+5/24/60');
end;
--停止job
begin
  dbms_job.broken(43,true);
end;
--删除job
begin
  dbms_job.remove(43);
end;
--查询job
select * from dba_jobs;
```

## 18. dbms_sql
```sql
create or replace function testzuo return integer is
       v_cursor NUMBER;
       v_stat NUMBER;
       v_row NUMBER;
       v_id NUMBER;
       v_no VARCHAR(100);
       v_date DATE;
       v_sql VARCHAR(200);
       s_id NUMBER;
       s_date DATE;
BEGIN
     s_id := 3000;
     s_date := SYSDATE;
     v_sql := 'SELECT id,qan_no,sample_date FROM "tblno" WHERE id > :sid and sample_date < :sdate';
     v_cursor := dbms_sql.open_cursor; --打开游标；
     dbms_sql.parse(v_cursor, v_sql, dbms_sql.native); --解析动态SQL语句；
     dbms_sql.bind_variable(v_cursor, ':sid', s_id); --绑定输入参数；
     dbms_sql.bind_variable(v_cursor, ':sdate', s_date);

     dbms_sql.define_column(v_cursor, 1, v_id); --定义列
     dbms_sql.define_column(v_cursor, 2, v_no, 100);
     dbms_sql.define_column(v_cursor, 3, v_date);
     v_stat := dbms_sql.execute(v_cursor); --执行动态SQL语句。
     LOOP
         EXIT WHEN dbms_sql.fetch_rows(v_cursor)<=0; --fetch_rows在结果集中移动游标，如果未抵达末尾，返回1。
         dbms_sql.column_value(v_cursor, 1, v_id); --将当前行的查询结果写入上面定义的列中。
         dbms_sql.column_value(v_cursor, 2, v_no);
         dbms_sql.column_value(v_cursor, 3, v_date);
         dbms_output.put_line(v_id || ';' || v_no || ';' || v_date);
     END LOOP;
     dbms_sql.close_cursor(v_cursor); --关闭游标。
     return 1;
END;

isAlive := dbms_session.is_session_alive( unique_sessionID );
```

## 19. 导出数据库日志

1.  运行cmd进入到 oracle的安装目录，这里以我的电脑为例： E:\app\Administrator\product\11.2.0\dbhome_1\
2.  再进入到 RDBMS\ADMIN目录。确保E:\app\Administrator\product\11.2.0\dbhome_1\RDBMS\ADMIN 下面有 awrrpt.sql这个文件。
3.  输入 sqlplus，然后以管理员身份登录。
4.  输入`@awrrpt`导出性能日志，输入`@addmrpt` 导出问题sql日志。
5.  开始按提示操作，首先是选择要生成的awr报告的类型，可以选择text或html类型。这里我们以 html类型为例，输入 html 回车。
6.  选择要生成的报告的日期是在多少天以前记录，输入1则表示要生成今天0点开始到现在之内的某个时间段的报告，输入2则表示满意生成昨天0点开始到现在的某个时间段的报告。以此类推。缺省记录最近7天，这里输入法为示例。
7.  输入天数后，界面会输出一个时间段的表格，每个时间点都对应一个snapId，间隔时间为oracle默认是1个小时，接下来输入要生成报告的时间开始点应的snap id，这里我输入3318，然后再输入结束点对应的snap id，这里输入 3320。

## 20. linux中安装oracle

```sql
0.安装虚拟机工具
tar zxvf VMwareTools.tar.gz
cd vmware-tools-distrib
./vmware-install.pl

1.创建用户组和用户(root)
查看操作系统版本：uname -a
groupadd oinstall
groupadd dba

useradd -g oinstall -G dba oracle
passwd oracle

2.创建安装目录和修改权限(root)
查看磁盘使用情况：df -h
mkdir /u01
chown -R oracle:oinstall /u01

3.修改环境变量：
su - oracle
ls -la |grep .bash_profile
vi .bash_profile

export ORACLE_SID=+ASM
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/grid
export PATH=$ORACLE_HOME/bin:$PATH

使环境变量生效：. .bash_profile或source .bash_profile

4.放宽oracle的限制（root用户）
vi /etc/sysctl.conf

fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 536870912
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586

sysctl -p使更改生效

vi /etc/security/limits.conf

oracle soft nproc 2047
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240

vi /etc/pam.d/login
session required pam_limits.so

vi /etc/profile

if [ $USER = "oracle" ]; then
    if [ $SHELL = "/bin/ksh" ]; then
        ulimit -p 16384
        ulimit -n 65536
    else
        ulimit -u 16384 -n 65536
    fi
umask 022
fi

关机：init 0

5.设置共享目录和8块2G硬盘
查看共享目录：mount
查看硬盘：fdisk -l
查看目录下文件数：ls |wc -l
查看当前路径：pwd

mkdir /mnt/iso
将镜像文件挂载到iso文件夹下mount -o loop Enterprise-R5-U4-Server-i386-dvd.iso /mnt/iso

vi /etc/yum.repos.d/myoel.repo
[dvdinfo]
name=myoel
baseurl=file:///mnt/iso/Server
enabled=1
gpgcheck=0

然后让yum主动到文件目录下搜集软件包的信息yum makecache

yum install -y compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers ksh libaio libaio-devel libXp libXp-devel libgcc libstdc++ libstdc++-devel make sysstat unixODBC unixODBC-devel

6.准备磁盘组所使用的磁盘
-依次对/dev/sdb、/dev/sdc、/dev/sdd、/dev/sde、/dev/sdf、/dev/sdg、/dev/sdh、/dev/sdi划分逻辑卷
fdisk /dev/sdb
fdisk /dev/sdc
fdisk /dev/sdd
fdisk /dev/sde
fdisk /dev/sdf
fdisk /dev/sdg
fdisk /dev/sdh
fdisk /dev/sdi

-配置udev，生成raw裸设备并修改权限
vi /etc/udev/rules.d/60-raw.rules

ACTION=="add",KERNEL=="sdb1",RUN+="/bin/raw /dev/raw/raw1 %N"
ACTION=="add",KERNEL=="sdc1",RUN+="/bin/raw /dev/raw/raw2 %N"
ACTION=="add",KERNEL=="sdd1",RUN+="/bin/raw /dev/raw/raw3 %N"
ACTION=="add",KERNEL=="sde1",RUN+="/bin/raw /dev/raw/raw4 %N"
ACTION=="add",KERNEL=="sdf1",RUN+="/bin/raw /dev/raw/raw5 %N"
ACTION=="add",KERNEL=="sdg1",RUN+="/bin/raw /dev/raw/raw6 %N"
ACTION=="add",KERNEL=="sdh1",RUN+="/bin/raw /dev/raw/raw7 %N"
ACTION=="add",KERNEL=="sdi1",RUN+="/bin/raw /dev/raw/raw8 %N"
KERNEL=="raw[1]",MODE="0660",GROUP="oinstall",OWNER="oracle"
KERNEL=="raw[2]",MODE="0660",GROUP="oinstall",OWNER="oracle"
KERNEL=="raw[3]",MODE="0660",GROUP="oinstall",OWNER="oracle"
KERNEL=="raw[4]",MODE="0660",GROUP="oinstall",OWNER="oracle"
KERNEL=="raw[5]",MODE="0660",GROUP="oinstall",OWNER="oracle"
KERNEL=="raw[6]",MODE="0660",GROUP="oinstall",OWNER="oracle"
KERNEL=="raw[7]",MODE="0660",GROUP="oinstall",OWNER="oracle"
KERNEL=="raw[8]",MODE="0660",GROUP="oinstall",OWNER="oracle"

-启动udev服务
start_udev
查看裸设备：raw -qa

7.安装grid
root登录并解压：unzip linux_11gR2_grid.zip
禁用XSerevr的访问控制：xhost +
切换oracle用户执行安装：
su - oracle
env |grep ORA          --查看环境变量
mkdir -p $ORACLE_HOME  --创建子目录
tree                   --查看子目录
./runInstaller

crs_stat -t            --查看grid服务
asmca                  --启动图形化配置工具，创建FRA磁盘组

8.安装database
su - oracle
vi .bash_profile

export ORACLE_SID=orcl
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/db_1
export PATH=$ORACLE_HOME/bin:$PATH

source .bash_profile

mkdir -p $ORACLE_HOME

exit            --退出到root
xhost +         --在linux启动GUI必须执行
su - oracle
./runInstaller  --进入相应目录并安装
```



# 二. sql server

sql server 默认的是自提交事务，也就是说每执行一条 dml 语句，此语句所做的修改都会在语句执行完毕后自动提交。
要想让 sql server 与 oracle 有同样的事务效果，可以执行 SET IMPLICIT_TRANSACTIONS ON 语句开启隐式事务。执行一条 dml 语句都会自动开启一个事务，直到用户执行 COMMIT TRAN 或 ROLLBACK TRAN 语句结束事务。
也有不同，在 sql server 中执行 ddl 语句和 dcl 语句也自动开启一个隐式事务；而 oracle 中执行 ddl 语句会自动提交之前的事务，并在 ddl 语句执行完毕自动提交 ddl 语句的事务。

## 1. 异机备份

```sql
if exists (select * from dbo.sysobjects where id = object_id(N'[dbo].[p_backupdb]') and OBJECTPROPERTY(id, N'IsProcedure') = 1) 
drop procedure [dbo].[p_backupdb] 
GO
create proc p_backupdb 
@dbname sysname='AIS20110306202234',
@bkpath nvarchar(260)='\\k3ser\landvback\',--exec master..xp_cmdshell 'net use \\计算机名\共享目录 "密码" /USER:计算机名\用户我';(\\计算机名\共享目录\备份文件名)
@bkfname nvarchar(260)='\DBNAME\_backup_\DATE\_\TIME\.BAK',
@bktype nvarchar(10)='DB',
@appendfile bit=1
as 
declare @sql varchar(8000) 
if isnull(@dbname,'')='' set @dbname=db_name() 
if isnull(@bkpath,'')='' set @bkpath=''
if isnull(@bkfname,'')='' set @bkfname='\DBNAME\_\DATE\_\TIME\.BAK' 
set @bkfname=replace(replace(replace(@bkfname,'\DBNAME\',@dbname)
,'\DATE\',convert(varchar,getdate(),112)) 
,'\TIME\',replace(convert(varchar,getdate(),108),':','')) 
set @sql='backup '+case @bktype when 'LOG' then 'log ' else 'database ' end +@dbname 
+' to disk='''+@bkpath+@bkfname 
+''' with '+case @bktype when 'DF' then 'DIFFERENTIAL,' else '' end 
+case @appendfile when 1 then 'NOINIT' else 'INIT' end 
print @sql 
exec(@sql) 
go
exec p_backupdb
```

## 2. tips

### 2.1 查询所有的表
```sql
SELECT Name FROM SysObjects Where XType='U' ORDER BY Name
```
### 2.2 修改表的模式（hbs0106为旧模式）
```sql
SELECT 'alter schema dbo transfer hbs0106.'+Name FROM SysObjects Where XType='U' ORDER BY Name
```
### 2.3 查询所有的列
```sql
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
```
### 2.4 给数据库唯一ID

```sql
--方法一：
update ZRZ set wyid=t1.rowId 
from (select zrzh,id,row_number() over (order by zrzh,id) as rowId from ZRZ) as t1 
where t1.zrzh=ZRZ.zrzh and t1.id=ZRZ.id;
--方法二：
ALTER TABLE zrz ADD MYID int;
update zrz set myid=rownum from (select myid,ROW_NUMBER() over(order by ID) rownum from zrz) zrz;
```

### 2.5 FOR XML PATH：多行转一行

```sql
SELECT B.sName,LEFT(StuList,LEN(StuList)-1) as hobby FROM (
SELECT sName,
(SELECT hobby+',' FROM student 
  WHERE sName=A.sName 
  FOR XML PATH('')) AS StuList
FROM student A 
GROUP BY sName
) B


select b.qtpkyy_id,(case when v_name='' then '' else left(v_name,len(v_name)-1) end) name from (
select qtpkyy_id,
(select name+',' from [fpb_wh0422].[dbo].[tb_other_poverty_attr]
 where qtpkyy_id=a.qtpkyy_id 
 for xml path('')) v_name 
from [fpb_wh0422].[dbo].[tb_other_poverty_attr] a 
group by qtpkyy_id) b;
```

## 3. 函数和存储过程

### 3.1 FindAllDB

```sql
USE [IDF]
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
--34     image
--35     text
--36     uniqueidentifier
--40     date
--41     time
--42     datetime2
--43     datetimeoffset
--48     tinyint
--52     smallint
--56     int
--58     smalldatetime
--59     real
--60     money
--61     datetime
--62     float
--98     sql_variant
--99     ntext
--104    bit
--106    decimal
--108    numeric
--122    smallmoney
--127    bigint
--128    hierarchyid
--129    geography
--130    hierarchyid
--165    varbinary
--167    varchar
--173    binary
--175    char
--189    timestamp
--231    nvarchar
--239    nchar
--241    xml
--256    sysname

if (exists (select * from sys.objects where name = 'FindAllDB'))
  drop proc FindAllDB
GO
CREATE PROCEDURE [dbo].[FindAllDB](@coln varchar(50),@colz varchar(50),@isName int=0,@isLike int=0)
AS
BEGIN
  DECLARE @v_tab varchar(50)
  DECLARE @v_col varchar(50)
  DECLARE @v_where nvarchar(300)
  DECLARE @v_sql nvarchar(300)
  DECLARE @v_count int
  
  if @isLike=0--字段值精准匹配
    set @v_where='='''+@colz+''''
  else--字段值模糊匹配
    set @v_where=' like ''%'+@colz+'%'''
  if @isName=0--字段名精准匹配
    DECLARE MyCur CURSOR FOR select b.name,a.name FROM syscolumns a,sysobjects b,sysindexes c where a.id=b.id and b.id=c.id and c.indid<2 and c.rows>0 and b.xtype='U' and a.xusertype in(175,239,167,231) and upper(a.name)=upper(@coln) order by b.name;
  if @isName=1--字段名模糊匹配
    DECLARE MyCur CURSOR FOR select b.name,a.name FROM syscolumns a,sysobjects b,sysindexes c where a.id=b.id and b.id=c.id and c.indid<2 and c.rows>0 and b.xtype='U' and a.xusertype in(175,239,167,231) and upper(a.name) like '%'+upper(@coln)+'%' order by b.name;
  else--字段名不匹配
    DECLARE MyCur CURSOR FOR select b.name,a.name FROM syscolumns a,sysobjects b,sysindexes c where a.id=b.id and b.id=c.id and c.indid<2 and c.rows>0 and b.xtype='U' and a.xusertype in(175,239,167,231) order by b.name;
  OPEN MyCur
  Fetch Next From MyCur Into @v_tab,@v_col
  While(@@Fetch_Status=0)
  begin
    set @v_sql='select @totalcount=count(*) from '+@v_tab+' where '+@v_col+@v_where
    exec sp_executesql @v_sql,N'@totalcount int output',@v_count output
    if @v_count!=0
      PRINT 'select * from '+@v_tab+' where '+@v_col+@v_where+';--'+convert(varchar(20),@v_count)
    Fetch Next From MyCur Into @v_tab,@v_col
  end
  Close MyCur
  Deallocate MyCur
END
--USE [IDF]
--GO
--exec [dbo].[FindAllDB] 'vchar','char串1'
```

# 二. mysql

## 1. tips

### 1.1 多行转一行

```sql
select entrance_name,group_concat(customer_type) from customer_entrance group by entrance_name;
```

# 三. 达梦

FindAllDB

```sql
set schema KANQ;
create or replace type TYPE_ARRAY is table of varchar(255);

create or replace function FindAllDB(coln varchar,colz varchar,isName int=0,isLike int=0) return TYPE_ARRAY pipelined  is
  tabarry TYPE_ARRAY;
  valstr varchar(100);
  mnu integer;
--INT
--INTEGER
--BIGINT
--NUMBER
--FLOAT
--CHAR
--VARCHAR
--VARCHAR2
--NVARCHAR2
--DATE
--TIMESTAMP
--DATETIME
--IMAGE
--TEXT
--CLOB
--BLOB
--BIT

--user_tables.num_rows不准确，需要维护
--字段名为DESC、TOP时与保留字冲突，需加双引号
--user_tab_columns存的字段名可能小写，但是达梦数据库不区分大小写
begin
  if isLike=0 then--字段值精准匹配
    valstr:='"='''||colz||''';';
  else--字段值模糊匹配
    valstr:='" like ''%'||colz||'%'';';
  end if;
  if isName=0 then--字段名精准匹配
    select mysql bulk collect into tabarry from (select distinct 'select count(1) from '||a.table_name||' where "'||a.column_name||valstr mysql from user_tab_columns a,user_tables b
    where a.table_name=b.table_name and upper(a.column_name)=upper(coln) /*and b.num_rows>0 */and a.data_type in('CHAR','VARCHAR','VARCHAR2','NVARCHAR2') order by mysql);
  elsif isName=1 then--字段名模糊匹配
    select mysql bulk collect into tabarry from (select distinct 'select count(1) from '||a.table_name||' where "'||a.column_name||valstr mysql from user_tab_columns a,user_tables b
    where a.table_name=b.table_name and upper(a.column_name) like '%'||upper(coln)||'%' /*and b.num_rows>0 */and a.data_type in('CHAR','VARCHAR','VARCHAR2','NVARCHAR2') order by mysql);
  else--字段名不匹配
    select mysql bulk collect into tabarry from (select distinct 'select count(1) from '||a.table_name||' where "'||a.column_name||valstr mysql from user_tab_columns a,user_tables b
    where a.table_name=b.table_name /*and b.num_rows>0 */and a.data_type in('CHAR','VARCHAR','VARCHAR2','NVARCHAR2') order by mysql);
  end if;
  
  for g in 1..tabarry.count loop
    execute immediate tabarry(g) into mnu;
    if mnu>0 then
      pipe row(replace(tabarry(g),'count(1)','*')||'--'||mnu);
    end if;
  end loop;
end;--select * from table(FindAllDB('unid','70080E080F2C227975E9C121C03CD642'));
```

# 四. 数据库相关脚本

## 4.1 bat注释

行注释1：rem+空格，dos会保留代码，但是不去执行，可做标签用。

`explorer c:`打开C盘，被注释掉了

```powershell
ipconfig
pause
rem explorer c:
pause
```

行注释2：::，此时不需要在中间加空格。

```powershell
ipconfig
::pause
explorer c:
pause
```

段、块注释：直接跳过，起到注释的作用。

```powershell
ipconfig

goto huo
pause
explorer c:
:huo

pause
```

## 4.2 拷贝语句

强制清空文件夹2中的内容，递归复制文件夹1到文件夹2

```powershell
rd/s/q C:\Users\zuo\Desktop\2
xcopy C:\Users\zuo\Desktop\1 C:\Users\zuo\Desktop\2 /s/e/i/y
```
## 4.3 forfiles

- /p Path 指定路径
- /m SearchMask 按照SearchMask搜索文件，默认是 *.*。我们想搜索rar文件可以写为 /m *.rar
- /s 如果不加此参数，只操作指定目录下这一级，反之指定目录下所有层级目录中的文件都会被操作。
- /c Command 在每个匹配的文件上运行指定的 Command，带有空格的命令字符串必须用双引号括起来。默认的 Command 是 “cmd /c echo @file”。
- /d[{+ | -}] [{MM/DD/YYYY | DD}]  根据上次修改日期选择文件。选择日期大于或等于 (+)（或者小于或等于 (-)）指定日期的文件，有绝对日期和相对日期。
- /?  显示帮助消息（简要使用说明）。禁止文件搜索/命令执行。不得与任何其它参数一起使用。

```powershell
forfiles [/p Path] [/m SearchMask] [/s] [/c Command] [/d[{+ | -}] [{MM/DD/YYYY | DD}]]
```

```powershell
rem 删除3天以上的备份
forfiles /p "%BACKUP_DIR%" /s /m *.* /d -3 /c "cmd /c del @path"
```


## 4.4 备份、停止服务后关机

BF.bat：参考上面备份脚本

BF_SHUTDOWN.bat

```powershell
call BF.bat
call SHUTDOWN.bat
```

SHUTDOWN.bat

```powershell
::停止oracle服务和监听
NET STOP OracleOraDb11g_home1TNSListener
NET STOP OracleServiceORCL
::-s代表关机 -t代表设置关机的时间,单位为秒
shutdown -s -t 0
```

## 4.5 定时删除3天前的归档日志

命令说明：

```shell
# 删除已备份过的归档，没有备份的归档，不会被删除。
DELETE ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-3';
# 删除归档，无论备份与否。
DELETE ARCHIVELOG UNTIL TIME 'sysdate-7' ; 
# 删除系统时间7天以内到现在的归档日志。
DELETE ARCHIVELOG FROM TIME 'SYSDATE-7';
```

delete_arch.txt

```powershell
run{
DELETE ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-3';
crosscheck archivelog all;
delete expired archivelog all;
}
```

delete_arch2.txt

```powershell
run{
DELETE ARCHIVELOG FROM TIME 'SYSDATE-7';
crosscheck archivelog all;
delete expired archivelog all;
}
```

delete_archive.bat，使用windows定时任务执行

```powershell
::rman是oracle的备份与恢复管理器
rman target sys/123456 @delete_arch.txt
pause
```

## 4.6 拷贝的脚本，没有分析

backup.bat

```powershell
@echo off 
forfiles /p "E:\Oracle\KANQOA\data_pump_dir" /s /m *.* /d -7 /c "cmd /c del @path" 
set backupfile=KANQOA_%date:~0,4%-%date:~5,2%-%date:~8,2%_%time:~0,2%%time:~3,2%%time:~6,2%.dmp 
set logfile=KANQOA_%date:~0,4%-%date:~5,2%-%date:~8,2%_%time:~0,2%%time:~3,2%%time:~6,2%.log 
expdp KANQOA/KANQOA@ORCL directory=data_pump_dir dumpfile=%backupfile% logfile=%logfile% schemas=KANQOA
```

backupall.bat

```powershell
@echo off
setlocal ENABLEDELAYEDEXPANSION
::读取配置文件
md %windir%\OracleAutoBackup >nul 2>nul
set configFile=%windir%\OracleAutoBackup\config.ini
set i=0
if not exist %configFile% echo.>%configFile%
for /f "delims=" %%x in (%configFile%) do (
if !i!==0 set bak_hou=%%x
if !i!==1 set bak_lot=%%x
if !i!==2 set bak_dir=%%x
if !i! gtr 2 (
set/a gup=!i!-2
set ora[!gup!]=%%x
)
set/a i+=1
)
::取默认值
if "!bak_hou!"=="" set bak_hou=3
echo !bak_hou!|findstr "^[0-9]*$">nul || set bak_hou=3
if "!bak_lot!"=="" set bak_lot=7
echo !bak_lot!|findstr "^[0-9]*$">nul || set bak_lot=7
if "!bak_dir!"=="" set bak_dir=%cd%\数据库备份
for /f "tokens=*" %%x in ("!bak_dir!") do set bak_dir=%%~fx
if not exist !bak_dir! md !val! >nul 2>nul
::去掉格式错误的数据库连接配置项
set j=0
for %%i in (1,2,3,4,5,6,7,8,9) do (
set ora[%%i]>nul 2>nul&& (
set ora_cur=
for /f "usebackq delims==. tokens=1-3" %%a in (`set ora[%%i]`) do set ora_cur=%%b
set ora[%%i]=
echo !ora_cur!|findstr "\/">nul 2>nul && echo !ora_cur!|findstr "@">nul 2>nul && (
set/a j+=1
set ora[!j!]=!ora_cur!
)
)
)
::进入管理程序
if "%1"=="" goto init

::检查exp命令是否可用
:checkexp
set resultFile=%temp%\%random%.txt
del %resultFile% /q>nul 2>nul
exp a/a@a%random% file=%temp%\%random%.dmp >nul 2>%resultFile%
if exist %resultFile% (
type %resultFile%|find "'exp' 不是内部或外部命令">nul
if !errorlevel!==0 (
del %resultFile%>nul
echo exp命令不可用！程序即将退出！
ping -n 10 127.1 >nul 2>nul
exit
)
del %resultFile%>nul
)
::1.数据库备份
title 备份进程
echo.
echo.
echo 一、正在进行备份……
for %%i in (1,2,3,4,5,6,7,8,9) do (
set ora[%%i]>nul 2>nul&& (
set ora_cur=
for /f "usebackq delims==. tokens=1-3" %%a in (`set ora[%%i]`) do set ora_cur=%%b
set ora_usr=
set ora_net=
for /f "delims=/" %%a in ('echo !ora_cur!') do set ora_usr=%%a
for /f "delims=@ tokens=2" %%a in ('echo !ora_cur!') do set ora_net=%%a
echo.
echo.
echo %%i.正在备份 !ora_usr!/******@!ora_net!……
md !bak_dir!\!ora_net!__!ora_usr!\ >nul 2>nul
set ftmr=!time: =0!
set bak_cur_dir=!bak_dir!\!ora_net!__!ora_usr!\
for /f "tokens=*" %%x in ("!bak_cur_dir!") do set bak_cur_dir=%%~fx
set bak_cur_fnm=!ora_net!__!ora_usr!__!date:~0,4!!date:~5,2!!date:~8,2!-!ftmr:~0,2!!ftmr:~3,2!
set bakfile=!bak_cur_dir!!bak_cur_fnm!.dmp
set logfile=!bak_cur_dir!!bak_cur_fnm!.log
exp !ora_cur! file="!bakfile!" log="!logfile!"
echo 如果备份成功的话，就进行压缩>nul
if exist "!bakfile!" (
pushd !bak_cur_dir!
set zipfile=
if exist "%ProgramFiles%\winrar\winrar.exe" (
echo 使用WinRAR进行压缩>nul
set zipfile=!bak_cur_fnm!.rar
"%programfiles%\winrar\winrar" a -r "!zipfile!" "!bak_cur_fnm!.dmp" "!bak_cur_fnm!.log"
) else (
echo 使用ZIP指令进行压缩>nul
set zipfile=!bak_cur_fnm!.zip
zip "!zipfile!" "!bak_cur_fnm!.dmp" "!bak_cur_fnm!.log">nul
)
if exist "!zipfile!" (
del /q "!bakfile!"
del /q "!logfile!"
)
popd
) else (
echo 如果不存在备份文件，但有日志文件，则删除日志文件>nul
if exist "!logfile!" del /q "!logfile!"
)
)
)
::2.数据库过期备份删除
echo.
echo.
echo 二、正在清除过期的备份文件……
for /f "tokens=1,2,3 delims=-" %%a in ('echo wscript.echo date-!bak_lot! ^>t~.vbs ^& cscript //nologo t~.vbs ^& del t~.vbs') do (
set y=%%a&set m=%%b&set d=%%c
if %%b lss 10 set m=0%%b
if %%c lss 10 set d=0%%c
)
set DateE=!y!-!m!-!d!
for %%i in (1,2,3,4,5,6,7,8,9) do (
set ora[%%i]>nul 2>nul&& (
set ora_cur=
for /f "usebackq delims==. tokens=1-3" %%a in (`set ora[%%i]`) do set ora_cur=%%b
set ora_usr=
set ora_net=
for /f "delims=/" %%a in ('echo !ora_cur!') do set ora_usr=%%a
for /f "delims=@ tokens=2" %%a in ('echo !ora_cur!') do set ora_net=%%a
set cur_dir=!bak_dir!\!ora_net!__!ora_usr!
for /f "tokens=*" %%x in ("!cur_dir!") do set cur_dir=%%~fx

echo 检查今天的备份成功了没有 >nul
set fnm_pre=!cur_dir!\!ora_net!__!ora_usr!__!date:~0,4!!date:~5,2!!date:~8,2!-& set today_success=0
dir !fnm_pre!*.dmp !fnm_pre!*.zip !fnm_pre!*.rar /b >nul 2>nul && set today_success=1
if "!today_success!"=="1" (
echo 判断文件夹条件是否满足 >nul
for %%a in (!cur_dir!\*.dmp,!cur_dir!\*.log,!cur_dir!\*.zip,!cur_dir!\*.rar) do (
echo 判断文件名称条件是否满足 >nul
set n=%%a&set n=!n:~-17,-9!&set n=!n:~0,4!-!n:~4,2!-!n:~6,2!
set t=%%~ta
set FileDate=!t:~0,10!
if "!n!"=="!FileDate!" (
echo 判断时间条件是否满足 >nul
if !FileDate! leq %DateE% (
echo %date:~0,10% %time:~0,8% 删除过期备份 %%a
echo %date:~0,10% %time:~0,8% 删除过期备份 %%a>>!cur_dir!\delete.log 
del /q "%%a"
)
)
)
) else (
echo %date:~0,10% %time:~0,8% [!ora_net!__!ora_usr!]因为今天的备份没有成功，暂时不删除过期文件！
echo %date:~0,10% %time:~0,8% [!ora_net!__!ora_usr!]因为今天的备份没有成功，暂时不删除过期文件！>>!cur_dir!\delete.log 
)
)
)
::3.完成退出
echo.
echo.
echo 三、本次备份操作完成，即将退出。
ping -n 10 127.1 >nul 2>nul
exit

::=================================以下是备份程序=================================
::=================================以下是管理程序=================================
:init
mode con cols=100 lines=40
title Oracle自动备份 - by zhouyou96
color 0e
::复制到 Windows 目录
copy "%~f0" "%windir%\OracleAutoBackup\OracleAutoBackup.bat" >nul 2>nul
::注册计划任务
:regtasks
sc config schedule start= auto >nul 2>nul
at|find "服务尚未启动">nul 2>nul&&(
net start schedule
if not !errorlevel!==0 (
echo Task Scheduler^(计划任务^)服务未能启动，程序即将退出！
pause>nul
goto exit
)
)
set job_tmr=!bak_hou!:00
if !bak_hou! lss 10 set job_tmr=0!bak_hou!:00
at !job_tmr! /every:1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31 %windir%\OracleAutoBackup\OracleAutoBackup.bat -backup >nul 2>nul
for /f "usebackq" %%i in (`dir %windir%\tasks\at*.job /b/o:d`) do set lastAt=%%i
del %windir%\tasks\Oracle自动备份.job >nul 2>nul
rename %windir%\tasks\!lastAt! Oracle自动备份.job
::保存配置文件
:saveconfig
echo !bak_hou!>%configFile%
echo !bak_lot!>>%configFile%
echo !bak_dir!>>%configFile%
for %%i in (1,2,3,4,5,6,7,8,9) do (
set ora[%%i]>nul 2>nul&& (
set ora_cur=
for /f "usebackq delims==. tokens=1-3" %%a in (`set ora[%%i]`) do set ora_cur=%%b
echo !ora_cur!>>%configFile%
)
)
::准备数据库配置字符串
set ora_str=
for %%i in (1,2,3,4,5,6,7,8,9) do (
set ora[%%i]>nul 2>nul&& (
set ora_cur=
for /f "usebackq delims==. tokens=1-3" %%a in (`set ora[%%i]`) do set ora_cur=%%b
set ora_usr=
set ora_net=
for /f "delims=/" %%a in ('echo !ora_cur!') do set ora_usr=%%a
for /f "delims=@ tokens=2" %%a in ('echo !ora_cur!') do set ora_net=%%a
set ora_str=!ora_str!%%i. !ora_usr!/******@!ora_net!； 
)
)
::开始
:start
cls
echo --------------------------------------------------------------------------------------------------
echo Oracle自动备份
echo 作者：zhouyou96 QQ:191458000
echo --------------------------------------------------------------------------------------------------
echo 　　使用操作系统自带的计划任务功能，每天定时运行exp命令导出指定的Oracle数据库并压缩，然后按需删除
echo 已过期的压缩的导出文件，以实现自动备份的功能。
echo 　　通常，为了便于管理，在我们公司一个oracle用户有且仅有的全权管理一个数据库，因此该用户的登陆名称
echo 其实可以视做为数据库名称。
echo.
echo 1.添加数据库：!ora_str!
echo 2.删除数据库
echo 3.设置文件夹：!bak_dir!
echo 4.几点钟备份：!bak_hou!
echo 5.删除几天前：!bak_lot!
echo 6.立即备份
echo 7.退出
echo.

::选择
:cho
set choice=
set /p choice=请选择：
if not "%choice%"=="" set choice=%choice:~0,1%
if "%choice%"=="1" goto addora
if "%choice%"=="2" goto delora
if "%choice%"=="3" goto setdir
if "%choice%"=="4" goto sethou
if "%choice%"=="5" goto setlot
if "%choice%"=="6" goto nowbak
if "%choice%"=="7" goto exit
echo.
echo =================================================================================================
echo =================================== 请选择1~7，按任意键重选！====================================
echo =================================================================================================
pause>nul
goto start

::添加数据库
:addora
set maxora=0
for %%i in (1,2,3,4,5,6,7,8,9) do (set ora[%%i]>nul 2>nul&&(set maxora=%%i))
set str_result=最多9个！ 
if not !maxora!==9 (
set/a maxora+=1
set new_ora=
set/p new_ora=请输入（用户名/密码@网络服务名）：
set str_result=格式错误！
if not "!new_ora!"=="" (
echo !new_ora!|findstr "\/">nul 2>nul && echo !new_ora!|findstr "@">nul 2>nul && (
set ora[!maxora!]=!new_ora!
set str_result=添加成功，
)
)
)
echo =================================================================================================
echo ==================================== !str_result!按任意键继续！====================================
echo =================================================================================================
pause>nul
if "!str_result!"=="添加成功，" goto saveconfig
goto start

::删除数据库
:delora
set str_result=操作错误！
set del_idx=0
set/p del_idx=请输入要删除的序数(1~9)：
if not "%del_idx%"=="" set del_idx=%del_idx:~0,1%
if "!del_idx!"=="" set del_idx=0
echo !del_idx!|findstr "^[0-9]*$">nul || set del_idx=0
if not "!del_idx!"=="0" (
set ora[!del_idx!]=
set str_result=删除成功，
)
::去掉格式错误的数据库连接配置项
if "!str_result!"=="删除成功，" (
set j=0
for %%i in (1,2,3,4,5,6,7,8,9) do (
set ora[%%i]>nul 2>nul&& (
set ora_cur=
for /f "usebackq delims==. tokens=1-3" %%a in (`set ora[%%i]`) do set ora_cur=%%b
set ora[%%i]=
echo !ora_cur!|findstr "\/">nul 2>nul && echo !ora_cur!|findstr "@">nul 2>nul && (
set/a j+=1
set ora[!j!]=!ora_cur!
)
)
)
)
echo =================================================================================================
echo ==================================== !str_result!按任意键继续！====================================
echo =================================================================================================
pause>nul
if "!str_result!"=="删除成功，" goto saveconfig
goto start
::设置文件夹
:setdir
set new_dir=
set/p new_dir=请输入备份用的文件夹：
if "!new_dir!"=="" set new_dir=%cd%\数据库备份
for /f "tokens=*" %%x in ("!new_dir!") do set new_dir=%%~fx
set bak_dir=!new_dir!
echo =================================================================================================
echo ==================================== 设置成功，按任意键继续！====================================
echo =================================================================================================
pause>nul
goto saveconfig
::几点钟备份
:sethou
set str_result=操作错误！
set new_hou=
set/p new_hou=请输入每天几点钟备份(0~23)：
echo !new_hou!|findstr "^[0-9]*$">nul || set new_hou=
if not "!new_hou!"=="" (
if !new_hou! geq 0 (
if !new_hou! leq 23 (
set bak_hou=!new_hou!
set str_result=设置成功，
)
)
)
echo =================================================================================================
echo ==================================== !str_result!按任意键继续！====================================
echo =================================================================================================
pause>nul
if "!str_result!"=="设置成功，" goto regtasks
goto start

::删除几天前
:setlot
set str_result=操作错误！
set new_lot=
set/p new_lot=请输入删除几天之前的备份(大于零)：
echo !new_lot!|findstr "^[0-9]*$">nul || set new_lot=
if not "!new_lot!"=="" (
if !new_lot! gtr 0 (
set bak_lot=!new_lot!
set str_result=设置成功，
)
)
echo =================================================================================================
echo ==================================== !str_result!按任意键继续！====================================
echo =================================================================================================
pause>nul
if "!str_result!"=="设置成功，" goto saveconfig
goto start
::现在备份
:nowbak
start %windir%\OracleAutoBackup\OracleAutoBackup.bat -backup
echo =================================================================================================
echo ==================================== 成功启动，按任意键继续！====================================
echo =================================================================================================
pause>nul
goto start

::退出程序
:exit
::pause>nul

```
