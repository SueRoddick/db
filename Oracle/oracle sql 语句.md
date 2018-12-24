##### 1、删除重复记录
 查找表中多余的重复记录，重复记录是根据单个字段（Id）来判断

select * from 表 where Id in (select Id from 表 group byId having count(Id) > 1)

 删除表中多余的重复记录，重复记录是根据单个字段（Id）来判断，只留有rowid最小的记录

DELETE from 表 WHERE (id) IN ( SELECT id FROM 表 GROUP BY id HAVING COUNT(id) > 1) AND ROWID NOT IN (SELECT MIN(ROWID) FROM 表 GROUP BY id HAVING COUNT(*) > 1);

 查找表中多余的重复记录（多个字段）

select * from 表 a where (a.Id,a.seq) in(select Id,seq from 表 group by Id,seq having count(*) > 1)

 删除表中多余的重复记录（多个字段），只留有rowid最小的记录

delete from 表 a where (a.Id,a.seq) in (select Id,seq from 表 group by Id,seq having count(*) > 1) and rowid not in (select min(rowid) from 表 group by Id,seq having count(*)>1)

 查找表中多余的重复记录（多个字段），不包含rowid最小的记录

select * from 表 a where (a.Id,a.seq) in (select Id,seq from 表 group by Id,seq having count(*) > 1) and rowid not in (select min(rowid) from 表 group by Id,seq having count(*)>1)
delete from 表 a where (a.Id,a.seq) in (select Id,seq from 表 group by Id,seq having count(*) > 1) and rowid not in (select min(rowid) from 表 group by Id,seq having count(*)>1)


delete from BT_SCALE_MAIN a
 where (a.intime) in (select intime from BT_SCALE_MAIN group by intime having count(intime) > 1) 
 and rowid not in (select min(rowid) from BT_SCALE_MAIN group by intime having count(intime)>1)

##### 2、导出导入数据
第一种：删除用户
--删除用户
drop user scts cascade;

--创建用户并授权（根据实际权限）
create user ylglxt identified by "ylglxt";
grant create session to ylglxt;
grant connect,resource,imp_full_database,exp_full_database,dba to ylglxt;


-- 导入导出直接在命令行里执行（不需要登录oracle用户）
exp scts/scts owner=scts file=scts_2016-07-26.dmp
imp scts3/scts3 file=scts2016-07-31_qiu.dmp full=y

第二种：直接导入导出
导出
exp lsreport/lsreport@129db file=E:\导出.dmp 

①完全导出 full=y 

②用户1和用户2的表导出 owner=(用户1,用户2)

③将表1和表2导出 tables=(table1,table2)

④将表1中字段1以“00” 开头的数据导出 tables=(table1) query=\"where 字段1 like '00%'\"

导入
imp ylglxt/ylglxt@orcl fromuser=lsreport touser=ylglxt file=e:\20160425data.dmp
fromuser=lsreport 导出用户名
touser=ylglxt 导入用户名

##### 3、日期

delete from BT_SCALE_MAIN a where to_date(a.intime , 'yyyy-mm-dd hh24:mi:ss') <(sysdate-4)

输出当月的所有天数
SELECT TRUNC (SYSDATE, 'MM') + ROWNUM - 1 FROM DUAL CONNECT BY ROWNUM <= TO_NUMBER (TO_CHAR (LAST_DAY (SYSDATE), 'dd'))

select to_char(sysdate,'yyyy-mm-dd hh24:mi:ss') as nowTime from dual; 

##### 4、查询前N条记录
①查询前10条记录

SELECT *　FROM torderdetail a WHERE ROWNUM <= 10

②查询10到20条（先排序后取数据）

SELECT * FROM (SELECT a.*, ROWNUM rn FROM torderdetail a)　WHERE rn >= 10 AND rn <= 20

③对于分组后取最近的10条纪录，则是rownum无法实现的，这时只有row_number可以实现，row_number() over(partition by 分组字段 order by 排序字段)就能实现分组后编号，比如说要取近一个月的每天最后10个订单纪录

SELECT * FROM (SELECT a.*, ROW_NUMBER () OVER (PARTITION BY TRUNC (order_date) ORDER BY order_date DESC) rn FROM torderdetail a)　WHERE rn <= 10

④输出当月的所有天数

SELECT TRUNC (SYSDATE, 'MM') + ROWNUM - 1 FROM DUAL CONNECT BY ROWNUM <= TO_NUMBER (TO_CHAR (LAST_DAY (SYSDATE), 'dd'))

##### 5、删除重复数据，只留一条数据
delete from bt_dbjh_tc 
where   planid in (select planid  from bt_dbjh_tc group by planid  having count(planid) > 1) 
and   rowid  not in (select min(rowid ) from bt_dbjh_tc group by planid having count(planid)>1)

##### 6、oracle账户过期修改
1、查看用户的proifle是哪个，一般是default：

①SELECT username,PROFILE FROM dba_users;

②查看指定概要文件（如default）的密码有效期设置： 
SELECT * FROM dba_profiles s WHERE s.profile='DEFAULT' AND resource_name='PASSWORD_LIFE_TIME';

③修改为无限期；修改之后不需要重启动数据库，会立即生效。
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;

④ 修改后，还没有被提示ORA-28002警告的帐户不会再碰到同样的提示；  已经被提示的帐户必须再改一次密码
alter user smsc identified by <原来的密码> ----不用换新密码

##### 7、oracle授权
系统权限授权命令：grant connect, resource, dba to 用户名1 [,用户名2]...;

系统权限传递：增加WITH ADMIN OPTION选项，则得到的权限可以传递。

 grant connect, resorce to user50 with admin option; //可以传递所获权限。
 
 系统权限回收：系统权限只能由DBA用户回收
 
  Revoke connect, resource from user50;

实体权限分类：select, update, insert, alter, index, delete, all //all包括所有权限
```
授予用户dba权限：
grant connect,resource,dba to lsreport  ---一般不用，一般用户都不要授予dba权限

user01:
SQL> grant select, update, insert on product to user02;
SQL> grant all on product to user02;
```
将表的操作权限授予全体用户：

SQL> grant all on product to public; // public表示是所有的用户，这里的all权限不包括drop。

实体权限传递(with grant option)：

user01:

SQL> grant select, update on product to user02 with grant option; // user02得到权限，并可以传递。

实体权限回收：

user01:

SQL>Revoke select, update on product from user02; //传递的权限将全部丢失。

##### 8、从一个表的字段更新另一个表的字段
```
merge into  LS_FLOW_RECORD_SHIFT   
using LS_FLOW_RECORD 
on(  LS_FLOW_RECORD_SHIFT.Recordid=LS_FLOW_RECORD.id)
when matched then
update set LS_FLOW_RECORD_SHIFT.flowid=LS_FLOW_RECORD.flowid;
```
##### 9 查看用户空间大小
```
①select tablespace_name ,sum(bytes) / 1024 / 1024 as MB　from dba_data_files group by tablespace_name;

②select tablespace_name,file_name from dba_data_files;

③select b.file_name 物理文件名,
b.tablespace_name 表空间,
b.bytes/1024/1024 大小M,
(b.bytes-sum(nvl(a.bytes,0)))/1024/1024 已使用M,
substr((b.bytes-sum(nvl(a.bytes,0)))/(b.bytes)*100,1,5) 利用率
from dba_free_space a,dba_data_files b
where a.file_id=b.file_id
group by b.tablespace_name,b.file_name,b.bytes
order by b.tablespace_name;
```
④各个实例所占空间
```
select *
  from (select owner || '.' || tablespace_name name, sum(b) g
          from (select owner,
                       t.segment_name,
                       t.partition_name,
                       round(bytes / 1024 / 1024 / 1024, 2) b,
                       tablespace_name
                 from dba_segments t)
        where owner not in
              ('SYS', 'OUTLN', 'SYSTEM', 'TSMSYS', 'DBSNMP', 'WMSYS')
         group by owner || '.' || tablespace_name)
 order by name
```
##### 10、oracle复制表数据，复制表结构 

①.不同用户之间的表数据复制 
对于在一个数据库上的两个用户A和B，假如需要把A下表old的数据复制到B下的new，请使用权限足够的用户登入sqlplus：

insert into B.new(select * from A.old);

如果需要加条件限制，比如复制当天的A.old数据

insert into B.new(select * from A.old where date=GMT); 

蓝色斜线处为选择条件

②.同用户表之间的数据复制?

用户B下有两个表：B.x和B.y，如果需要从表x转移数据到表y，使用用户B登陆sqlpus即可：

insert into?目标表y select * from x where log_id>'3049' --?复制数据?

注意：要示目标表y必须事先创建好

如insert into bs_log2 select * from bs_log where log_id>'3049' 

③.B.x中个别字段转移到B.y的相同字段?

--如果两个表结构一样
insert into table_name_new select * from table_name_old 

如果两个表结构不一样：
insert into y(字段1,字段2) select?字段1,字段2 from x

④.只复制表结构 加入了一个永远不可能成立的条件1=2，则此时表示的是只复制表结构，但是不复制表内容???

create table 用户名.表名 as select * from 用户名.表名 where 1=2
如create table zdsy.bs_log2 as select * from zdsy.bs_log where 1=2

⑤完全复制表(包括创建表和复制表中的记录)
create table test as select * from bs_log??--bs_log是被复制表

⑥将多个表数据插入一个表中

insert into 目标表test(字段1。。。字段n) (select?字段1.。。。。字段n) from 表  union all select?字段1.....字段n from 表
?

⑦、创建用户budget_zlgc，权限和budget相同,
（A、只复制所有表结构

B、复制所有表所有信息）

创建用户budget_zlgc，并导出budge用户数据
```
exp userid="\"sys/sys as sysdba"\" file='/backup/expdb/oa0824.dmp' log='/backup/expdb/oaex0825.log' owner=budget ignore=Y buffer=256000000
```
```
A、budget用户所有表，表结构和budget相同，（无数据）
imp userid="\"sys/sys as sysdba"\" file='/backup/expdb/oa0824.dmp' log='/backup/expdb/oa0825.log' fromuser=budget touser=budget_zlgc ignore=Y buffer=256000000 rows=n
B、budget用户所有表，表结构、数据和budget相同
imp userid="\"sys/sys as sysdba"\" file='/backup/expdb/oa0824.dmp' log='/backup/expdb/oa0825.log' fromuser=budget touser=budget_zlgc ignore=Y
```
##### 11、释放表删除数据空间
truncate table  tablename  DROP STORAGE;

 ##### 12、使用flashback（闪回）恢复误删除的数据 或 误删除的表 
1、从回收站恢复时重命名表
SQL> flashback table t2 to before drop rename to t4 ;

2、删除回收站指定的表
SQL> purge table t4;

3、清空回收站
SQL> purge recyclebin

4、删除表时不经过回收站直接删除
SQL> drop table t3 purge ;

5、从回收站看删除的表
select * from user_recyclebin 

①恢复表中误删除的记录
  前提：
  1.表的结构未改动;如果在删除后表结构发生改动则不能使用闪回；
  
  2.用户必须有足够的权限
  
直接用个例子来说明更加直观：
  表：
create table TEST1
(
 ID NUMBER(10) not null,
 CREATE_DATE DATE
)
SQL> select count(*) from test1;

COUNT(*)

 57158
SQL> delete test1 where id<1000;

已删除999行。

SQL> select count(*) from test1;

COUNT(*)

 56159
SQL>select sysdate from dual;

SYSDATE

2013-06-03 15:35:57

//开始恢复数据 

SQL> alter table test1 enable row movement;//在闪回前必须 启动行移动功能 否则会报错误：  ORA-08189: 因为未启用行移动功能, 不能闪回表

表已更改。

SQL> FLASHBACK TABLE test1 TO TIMESTAMP to_timestamp('2013-06-03 15:35:00','yyyy-mm-dd hh24:mi:ss');

//注意：恢复时间点必须是在删除数据之前 这里是2013-06-03 15:35:57 之前就可以

闪回完成。

SQL> select count(*) from test1;

COUNT(*)

 57158
 
数据已经恢复成功，57158行 跟未删除前是一样的，如果想看被删除了哪些行（在删除后闪回前）：

SELECT * FROM test1 AS OF TIMESTAMP to_timestamp('2013-06-03 15:35:00','yyyy-mm-dd hh24:mi:ss')
MINUS
SELECT * FROM test1

②恢复被删除的表

SQL> drop table test1;
表已删除。

SQL> flashback table test1 to before drop;
闪回完成。

##### 13、 delete 删除数据恢复：（必须9i或10g以上版本支持，flashback无法恢复全文索引）

利用Oracle提供的闪回方法，如果在删除数据后还没做大量的操作（只要保证被删除数据的块没被覆写），就可以利用闪回方式直接找回删除的数据
具体步骤为：

*确定删除数据的时间（在删除数据之前的时间就行，不过最好是删除数据的时间点）

*用以下语句找出删除的数据：select * from 表名 as of timestamp to_timestamp('删除时间点','yyyy-mm-dd hh24:mi:ss')

*把删除的数据重新插入原表：

     insert into 表名 (select * from 表名 as of timestamp to_timestamp('删除时间点','yyyy-mm-dd hh24:mi:ss'));注意要保证主键不重复。

##### 14、ORACLE EBS操作某一个FORM界面，或者后台数据库操作某一个表时发现一直出于"假死"状态，可能是该表被某一用户锁定，导致其他用户无法继续操作
复制代码 代码如下:
--锁表查询SQL

SELECT object_name, owner,machine, object_type,s.sid, s.serial#
FROM gv$locked_object l, dba_objects o, gv$session s
WHERE l.object_id = o.object_id AND l.session_id = s.sid;

找到被锁定的表，解锁

复制代码 代码如下:--释放SESSION SQL:

--alter system kill session 'sid, serial#';

##### 15、Oracle表分区 

创建表及分区

```
create table DT_ELECTRICITY_MINUTEDATA
(
  rtuid     INTEGER,
  mp_id     INTEGER,
  rtudesc   NVARCHAR2(100), 
  "date"   INTEGER,
  time    INTEGER
) 

PARTITION BY RANGE("date")

 (  
     PARTITION elec_min_part201612 VALUES LESS THAN (20170101) ,     
     PARTITION elec_min_part201701 VALUES LESS THAN (20170201) , 
     PARTITION elec_min_part201702 VALUES LESS THAN (20170301) ,      
     PARTITION elec_min_part201703 VALUES LESS THAN (20170401) ,      
     PARTITION elec_min_part201704 VALUES LESS THAN (20170501) ,      
     PARTITION elec_min_part201705 VALUES LESS THAN (20170601) ,     
     PARTITION elec_min_part201706 VALUES LESS THAN (20170701) ,       
     PARTITION elec_min_part201707 VALUES LESS THAN (20170801)       
); 

 create index DT_ELEC_MINDATABACK_INDEX on DT_ELECTRICITY_MINUTEDATA (RTUID, MP_ID, "date", TIME)
  tablespace USERS
  pctfree 10
  initrans 2
  maxtrans 255
  storage
  (
    initial 64K
    next 1M
    minextents 1
    maxextents unlimited
  ) 
  ```
--ALTER TABLE DT_ELECTRICITY_MINUTEDATA EXCHANGE PARTITION elec_min_part201602 WITH TABLE DT_ELECTRICITY_MINUTEDATA_BACK;--交换分区

--alter table DT_ELECTRICITY_MIN add partition elec_minute_part201708 VALUES LESS THAN (20170901);--添加分区

-- SELECT COUNT(*) FROM DT_ELECTRICITY_MINUTEDATA PARTITION (elec_min_part201601);

 /* alter table DT_ELECTRICITY_MINUTEDATA_BACK
 exchange partition elec_min_part201602
with table DT_ELECTRICITY_MINUTEDATA without validation*/

--收集表的统计信息

--exec dbms_stats.gather_table_stats('ENERGY', 'DT_ELECTRICITY_MINUTEDATA', cascade => true);

--检查重定义的合理性

--exec dbms_redefinition.can_redef_table('ENERGY', 'DT_ELECTRICITY_MINUTEDATA');

SELECT * FROM dba_part_tables where owner='ENERGY';--查看用户那些表有分区

SELECT * FROM dba_tab_partitions WHERE table_name = 'DT_ELECTRICITY_MINUTEDATA' and partition_name='ELEC_MIN_PART201707';--查找表的分区

SELECT * FROM dba_ind_partitions WHERE index_name = 'DT_ELECTRICITY_MINUTEDATA';--查找索引

 --INSERT INTO DT_ELECTRICITY_MINUTEDATA  SELECT * FROM  DT_ELECTRICITY_MINUTEDATA_BACK  


 -- ALTER TABLE DROP PARTITION --来删除分区，元数据和数据将被一并删除。-全删除

--ALTER TABLE DT_ELECTRICITY_MINUTEDATA DROP PARTITION ELEC_MINUTE_PART201708;--删除不需要的分区

--ALTER TABLE yourTable TRUNCATE PARTITION partionName1;
--语句虽简单、操作需谨慎。

#####  16、archive log 日志已满处理方法
错误提示：ORA-00257: archiver error. Connect internal only, until freed
Oracle archive log归档文件地址D:\app\Administrator\flash_recovery_area\YULANGDB\ARCHIVELOG

select * from V$RECOVERY_FILE_DEST; //查询归档日志空间大小及路径

show parameter recover; //显示归档文件路径

select * from v$flash_recovery_area_usage; --查看空间占用率

alter system set db_recovery_file_dest_size=20G;--增大归档日志空间


进入rman进行删除归档，相关命令如下：
rman target sys/lsreport@yulangdb--登录rman

RMAN>list archive log --查看archivelog 列表

RMAN>show parameter log_archive_dest;--查看archivelog日志位置

RMAN>delete expired archivelog all; //删除所有归档日志

RMAN>DELETE ARCHIVELOG ALL COMPLETED BEFORE ‘SYSDATE-7’; //保留7天的归档日志


关闭归档 
SQL> alter system set log_archive_start=false scope=spfile; #禁用自归档

SQL> shutdown immediate; //强制关闭数据库 

SQL> startup mount; //重启数据库到mount模式 

SQL> alter database noarchivelog; //修改为非归档模式 

SQL> alter database open; //打数据文件 

SQL> archive log list; //再次查看前归档模式

##### 17 查询数据库，表名，视图，列名等
```
select instance_name from v$instance;

select name,dbid from v$database;

select value from v$parameter where name='db_domain';

SELECT * FROM user_views;

select col.column_name 
from user_constraints con,  user_cons_columns col 
where con.constraint_name = col.constraint_name 
and con.constraint_type='P' 
and col.table_name = 'LS_FLOW_TYPE';


SELECT USER_TAB_COLS.TABLE_NAME   as 表名,
       USER_TAB_COLS.COLUMN_NAME  as 列名,
       USER_TAB_COLS.DATA_TYPE    as 数据类型,
       USER_TAB_COLS.DATA_LENGTH  as 长度,
       USER_TAB_COLS.NULLABLE     as 是否为空,
       USER_TAB_COLS.COLUMN_ID    as 列序号,
       user_col_comments.comments as 备注
       
  FROM USER_TAB_COLS
  
 inner join user_col_comments
    on user_col_comments.TABLE_NAME = USER_TAB_COLS.TABLE_NAME
   and user_col_comments.COLUMN_NAME = USER_TAB_COLS.COLUMN_NAME
   where USER_TAB_COLS.TABLE_NAME='TEST'
```
