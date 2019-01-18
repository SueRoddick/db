#### 一、Oracle分区简介 

ORACLE的分区是一种处理超大型表、索引等的技术。分区是一种“分而治之”的技术，通过将大表和索引分成可以管理的小块，从而避免了对每个表作为一个大的、单独的对象进行管理，为大量数据提供了可伸缩的性能。分区通过将操作分配给更小的存储单元，减少了需要进行管理操作的时间，并通过增强的并行处理提高了性能，通过屏蔽故障数据的分区，还增加了可用性。 
#### 二、Oracle分区优缺点 

 优点： 
增强可用性：如果表的某个分区出现故障，表在其他分区的数据仍然可用； 
维护方便：如果表的某个分区出现故障，需要修复数据，只修复该分区即可； 
均衡I/O：可以把不同的分区映射到磁盘以平衡I/O，改善整个系统性能； 
改善查询性能：对分区对象的查询可以仅搜索自己关心的分区，提高检索速度。 
 缺点： 
分区表相关：已经存在的表没有方法可以直接转化为分区表。不过 Oracle 提供了在线重定义表的功能。

#### 三、创建和使用分区表

##### 1.范围分区（RANGE）

范围分区将数据基于范围映射到每一个分区，这个范围是你在创建分区时指定的分区键决定的。这种分区方式是最为常用的，并且分区键经常采用日期。当使用范围分区时，请考虑以下几个规则：
1）每一个分区都必须有一个VALUES LESS THEN子句，它指定了一个不包括在该分区中的上限值。分区键的任何值等于或者大于这个上限值的记录都会被加入到下一个高一些的分区中。
2）所有分区，除了第一个，都会有一个隐式的下限值，这个值就是此分区的前一个分区的上限值。
3）在最高的分区中，MAXVALUE被定义。MAXVALUE代表了一个不确定的值。这个值高于其它分区中的任何分区键的值，也可以理解为高于任何分区中指定的VALUE LESS THEN的值，同时包括空值。

例一：假设有一个CUSTOMER表，表中有数据200000行，我们将此表通过CUSTOMER_ID进行分区，每个分区存储100000行，我们将每个分区保存到单独的表空间中，这样数据文件就可以跨越多个物理磁盘。下面是创建表和分区的代码，如下：

```sql
例一：按数字类型（ID）划分
CREATE TABLE CUSTOMER
(
    CUSTOMER_ID NUMBER NOT NULL PRIMARY KEY,
    FIRST_NAME NOT NULL,
    LAST_NAME NOT NULL,
    PHONE NOT NULL,
    EMAIL VARCHAR2(80),
    STATUS CHAR(1)
)
PARTITION BY RANGE (CUSTOMER_ID)
(
    PARTITION CUS_PART1 VALUES LESS THAN (100000) TABLESPACE CUS_TS01,
    PARTITION CUS_PART2 VALUES LESS THAN (200000) TABLESPACE CUS_TS02
);

添加分区SQL：
alter table CUSTOMER add partition CUS_PART3 VALUES LESS THAN (300000);

例二：按时间划分
CREATE TABLE ORDER_ACTIVITIES 
( 
    ORDER_ID      NUMBER(7) NOT NULL, 
    ORDER_DATE    DATE, 
    TOTAL_AMOUNT NUMBER, 
    CUSTOTMER_ID NUMBER(7), 
    PAID   CHAR(1) 
) 
PARTITION BY RANGE (ORDER_DATE) 
(
  PARTITION ORD_ACT_PART01 VALUES LESS THAN (TO_DATE('01- MAY -2003','DD-MON-YYYY')) TABLESPACEORD_TS01,
  PARTITION ORD_ACT_PART02 VALUES LESS THAN (TO_DATE('01-JUN-2003','DD-MON-YYYY')) TABLESPACE ORD_TS02,
  PARTITION ORD_ACT_PART02 VALUES LESS THAN (TO_DATE('01-JUL-2003','DD-MON-YYYY')) TABLESPACE ORD_TS03
);
```

##### 2.列表分区（LIST）

该分区的特点是某列的值只有几个，基于这样的特点我们可以采用列表分区。创建一个按字段数据列表固定可枚举值分区的表。插入记录分区字段的值必须在列表中，否则不能被插入。

```sql
CREATE  TABLE  ListTable
( 
    id    INT  PRIMARY  KEY , 
    name  VARCHAR (20), 
    area  VARCHAR (10) 
) 
PARTITION  BY  LIST (area) 
( 
    PARTITION  part1 VALUES ('guangdong','beijing') TABLESPACE  Part1_tb, 
    PARTITION  part2 VALUES ('shanghai','nanjing')  TABLESPACE  Part2_tb 
);
```

##### 3.哈希分区（散列分区）（HASH）

hash分区最主要的机制是根据hash算法来计算具体某条纪录应该插入到哪个分区中,hash算法中最重要的是hash函数，Oracle中如果你要使用hash分区，只需指定分区的数量即可。建议分区的数量采用2的n次方，这样可以使得各个分区间数据分布更加均匀。

```sql
CREATE TABLE HASH_TABLE 
( 
  COL NUMBER(8), 
  INF VARCHAR2(100) 
) 
PARTITION BY HASH (COL) 
( 
  PARTITION PART01 TABLESPACE HASH_TS01, 
  PARTITION PART02 TABLESPACE HASH_TS02, 
  PARTITION PART03 TABLESPACE HASH_TS03 
)

简写：
CREATE TABLE emp
(
    empno NUMBER (4),
    ename VARCHAR2 (30),
    sal   NUMBER 
)
PARTITION BY  HASH (empno) PARTITIONS 8
STORE IN (emp1,emp2,emp3,emp4,emp5,emp6,emp7,emp8);

```

##### 4.组合分区（RANGE-LIST 和 RANGE-HASH）

1）基于范围分区和列表分区，表首先按某列进行范围分区，然后再按某列进行列表分区，分区之中的分区被称为子分区。

```sql
CREATE TABLE SALES 
(
PRODUCT_ID VARCHAR2(5),
SALES_DATE DATE,
SALES_COST NUMBER(10),
STATUS VARCHAR2(20)
)
PARTITION BY RANGE(SALES_DATE) SUBPARTITION BY LIST (STATUS)
(
   PARTITION P1 VALUES LESS THAN(TO_DATE('2003-01-01','YYYY-MM-DD'))TABLESPACE rptfact2009 
  ( 
      SUBPARTITION P1SUB1 VALUES ('ACTIVE') TABLESPACE rptfact2009, 
      SUBPARTITION P1SUB2 VALUES ('INACTIVE') TABLESPACE rptfact2009 
  ), 
   PARTITION P2 VALUES LESS THAN (TO_DATE('2003-03-01','YYYY-MM-DD')) TABLESPACE rptfact2009 
  ( 
      SUBPARTITION P2SUB1 VALUES ('ACTIVE') TABLESPACE rptfact2009, 
      SUBPARTITION P2SUB2 VALUES ('INACTIVE') TABLESPACE rptfact2009 
  ) 
)
```

2）基于范围分区和散列分区，表首先按某列进行范围分区，然后再按某列进行散列分区。

```sql
create table dinya_test 
 ( 
 transaction_id number primary key, 
 item_id number(8) not null, 
 item_description varchar2(300), 
 transaction_date date 
 ) 
 partition by range(transaction_date)
 subpartition by hash(transaction_id) subpartitions 3 store in (dinya_space01,dinya_space02,dinya_space03) 
 ( 
     partition part_01 values less than(to_date(‘2006-01-01','yyyy-mm-dd')), 
     partition part_02 values less than(to_date(‘2010-01-01','yyyy-mm-dd')), 
     partition part_03 values less than(maxvalue) 
 );
```

```sql
CREATE TABLE range_hash_example(
 range_column_key int,
 hash_column_key  INT,
 DATA <span style="white-space:pre">	</span>VARCHAR2(20)
)
PARTITION BY RANGE(range_column_key)
SUBPARTITION BY HASH(hash_column_key) SUBPARTITIONS 2
(
  PARTITION part_1 VALUES LESS THAN (100000000)
  (
     SUBPARTITION part_1_sub_1,
     SUBPARTITION part_1_sub_2,
     SUBPARTITION part_1_sub_3
  ),
  PARTITION part_2 VALUES LESS THAN (200000000)
  (
     SUBPARTITION part_2_sub_1,
     SUBPARTITION part_2_sub_2
  )
);
```

--注： subpartitions 2 并不是指定subpartition的个数一定为2，实际上每个分区的子分区个数可以不同。如果不指定subpartition的具体明细，则系统按照subpartitions的值指定subpartition的个数生成子分区，名称由系统定义 。

#### 四、  有关分区表的维护操作

##### 0、在已存在且有数据的表上创建分区

  参考：https://www.cnblogs.com/25288-hf/p/6691027.html

##### 1.添加分区

以下代码给SALES表添加了一个P3分区

```sql
ALTER TABLE SALES ADD PARTITION P3 VALUES LESS THAN(TO_DATE('2003-06-01','YYYY-MM-DD'));
```

注意：以上添加的分区界限应该高于最后一个分区界限。
以下代码给SALES表的P3分区添加了一个P3SUB1子分区

```sql
ALTER TABLE SALES MODIFY PARTITION P3 ADD SUBPARTITION P3SUB1 VALUES('COMPLETE');


-- range partitioned table
ALTER TABLE range_example ADD PARTITION part04 VALUES LESS THAN (TO_DATE('2008-10-1 00:00:00','yyyy-mm-ddhh24:mi:ss'));
 
--list partitioned table
ALTER TABLE list_example ADD PARTITION part04 VALUES('TE');
 
--Adding Values for a List Partition
ALTER TABLE list_example MODIFY PARTITION part04 ADD VALUES('MIS');
 
--Dropping Values from a List Partition
ALTER TABLE list_example MODIFY PARTITION part04 DROP VALUES('MIS');
 
--hash partitioned table
ALTER TABLE hash_example ADD PARTITION part03;
 
--增加subpartition
ALTER TABLE range_hash_example MODIFY PARTITION part_1 ADD SUBPARTITION part_1_sub_4;
 
注:hash partitioned table新增partition时，现有表的中所有data都有重新计算hash值，然后重新分配到分区中，所以被重新分配的分区的indexes需要rebuild 。
```

##### 2.删除分区

```sql
ALTER TABLE SALES DROP PARTITION P3;
 
ALTER TABLE SALES DROP SUBPARTITION P4SUB1;

注意：如果删除的分区是表中唯一的分区，那么此分区将不能被删除，要想删除此分区，必须删除表。
```

##### 3.截断分区

截断某个分区是指删除某个分区中的数据，并不会删除分区，也不会删除其它分区中的数据。当表中即使只有一个分区时，也可以截断该分区。通过以下代码截断分区：

```sql
ALTER TABLE SALES TRUNCATE PARTITION P2;
```

通过以下代码截断子分区：

```sql
ALTER TABLE SALES TRUNCATE SUBPARTITION P2SUB2;
```

##### 4.合并分区

合并分区是将相邻的分区合并成一个分区，结果分区将采用较高分区的界限，值得注意的是，不能将分区合并到界限较低的分区。

```sql
ALTER TABLE SALES MERGE PARTITIONS P1,P2 INTO PARTITION P2 UPDATE INDEXES;
```

--如果省略update indexes子句的话，必须重建受影响的分区的index;

```sql
ALTER TABLE range_example MODIFY PARTITION part02 REBUILD UNUSABLE LOCAL INDEXES;
```

##### 5.拆分分区

拆分分区将一个分区拆分两个新分区，拆分后原来分区不再存在。注意不能对HASH类型的分区进行拆分。

```sql
ALTER TABLE SALES SBLIT PARTITION P2 AT(TO_DATE('2003-02-01','YYYY-MM-DD')) INTO (PARTITION P21,PARTITION P22);
```

注意：如果是RANGE类型的，使用at，LIST类型的使用values。

##### 6.接合分区（coalesce）

分区接合是针对散列分区或者*-散列子分区的，目的是减少分区数。当某个散列分区接合后，Oracle将其分区的数据分散到其它分区中。被接合的分区是由数据库选择的，接合完成后该分区会被删除，且如果没有使用UPDATE INDEX子句，本地索引和全局索引均将变成不可用，一般需要重建索引。

```sql
--散列分区表的散列分区接合
ALTER TABLE table_name COALESCE PARTITION;
--散列分区表的散列子分区接合
ALTER TABLE table_name MODIFY PARTITION partition_name COALESCE SUBPARTITION;
```

##### 7.重命名表分区

```sql
ALTER TABLE table_name RENAME PARTITION old_name TO new_name;
ALTER TABLE table_name RENAME SUBPARTITION old_name TO new_name;
```

##### 8.交换分区

可以将一个分区(子分区)和非分区表进行数据交换，oracle交换的方法是其实是对逻辑存储段进行交换。同样，散列|范围|列表分区可以与复合*-散列|*-范围|*-列表分区间也可以进行数据交换。当应用中需要将非分区表的数据转换进入分区表的分区时非常高效实用。使用INCLUDEING INDEXES子句可以同步将本地索引也进行交换，使用WITH VALIDATATION子句还可以实现行数据的验证。
交换分区时如果不带UPDATE INDEXES子句，则全局索引或全局索引基于的分区将变为不可用。

1）三种单级分区与非分区表的交换

```sql
ALTER TABLE table_name EXCHANGE PARTITION partition_name WITH TABLE nonpartition_name;
```

##### 9.移动分区

```sql
alter table custaddr move partition P_OTHER tablespace system;
alter table custaddr move partition P_OTHER tablespace icd_service;
```

分区移动会自动维护局部分区索引，oracle不会自动维护全局索引，所以需要我们重新rebuild分区索引，具体需要rebuild哪些索引，可以通过dba_part_indexes,dba_ind_partitions去判断。

```sql
Select index_name,status From user_indexes Where table_name='CUSTADDR';
```

##### 10.关于分区表和索引

1）本地分区索引
本地分区索引是使用了LOCAL属性创建的分区索引，其特征是索引分区的所有键均指向其基表某个 唯一分区中存储的相应行。Oracle创建本地分区索引的目的就是要确保索引也是分区管理的，而且索引的分区与表的分区是均衡的，也就是本地分区索引具有与其基表相同的分区、子分区，即分区键等同于表的分区键、分区数等同于表的分区数。
任何基表分区的增加、删除、合并、分割操作，或者散列分区增加或合并操作，Oracle会通过其自身的机制自动维护本地分区索引相应的分区，此即本地分区索引与基表的均衡性原则。
如果分区列能够形成索引列的一个子集，则本地分区索引可以是唯一索引。该限制能确保具有相同索引键的行始终映射到同一个分区，在该分区中，违反唯一性的行为能被检测到。
本地索引的优势有：
l在基表上执行除SPLIT PARTITION或ADD PARTITION 外的维护命令仅仅只有一个分区会被影响
l当分区表只有一个本地分区索引时，对分区进行维护操作的时间是与分区的大小成正比的
l本地分区索引支持分区的独立性
l只能本地分区索引支持单一分区数据的装入和卸出
l本地索引与基表的均衡性会给Oracle执行计划带来更好的性能
l表分区和各自的本地索引可以同时恢复
l分区表中的位图索引必须是本地索引，非分区表上不能建立分区位图索引
①本地前缀索引
本地前缀索引是指以索引列的左前缀来分区的，如果存在子分区则要求其子分区的分区键包含在索引键中。本地前缀索引可以是唯一索引，也可以是非唯一索引。
②本地非前缀索引
本地非前缀索引是指没有以索引列的左前缀来分区的，或者是以索引列的左前缀来分区，但子分区的分区键不在索引键中。本地非前缀索引不可以是唯一索引，除非分区键是索引键的子集（此时是前缀索引）。

2）全局分区索引
全局分区索引是指某个特定索引分区中的键可能指向存储在基表中的多个分区或子分区中的行，其创建需要使用GLOBAL属性。无论分区表是那种类型的分区，全局索引只支持按范围和散列分区两种分区方式。
全局索引往往与基表是不均衡的，如果要追求二者的均衡性，只能使用本地分区索引。全局分区的索引类型只能是b-tree索引，不能是bitmap索引。正是因为其是b-tree索引，所以无论其指向多少个分区抑或多少行，也只有一个b-tree入口，每个索引分区中会再包含指向具体表分区或子分区中的行的键。
全局分区索引必须是前缀的，不支持非前缀的。其中，前缀的意思和本地分区索引的前缀的含义相同，即“前缀索引是指以索引列的左前缀来分区的”。

管理全局分区索引
当基表分区移动和删除（TRUNCATE、DROP、MOVE、SPLIT）时，全局索引的所有分区都受影响，也不支持分区依赖

当基表分区或子分区恢复到某个时间点时，全局索引中所有相应入口也要恢复到相同的时间点。由于索引的分区或子分区的入口可能离散分布，其它分区或子分区混合型入口不恢复，除了重建索引之外没有办法完成

以表range_example为例：

1）建立普通的索引

```sql
create index com_index_range_example_id on range_example(id);
```

2）建立本地分区索引

```sql
create index  local_index_range_example_id on range_example(id) local;
```

3）建立全局分区索引

```sql
create index gidx_range_example_id on range_example(id)
GLOBAL partition by  range(id)
(
 part_01 values less than(1000),
 part_02 values less than(MAXVALUE)
);
```

对于分区索引的删除，local index 不能指定分区名称，单独的删除分区索引。local index 对应的分区会伴随着data分区的删除而一起被删除。

global partition index 可以指定分区名称，删除某一分区。但是有一点要注意，如果该分区不为空，则会导致更高一级的索引分区被置为UNUSABLE 。

```sql
ALTER INDEX gidx_range_exampel_id drop partition part_01 ;
```

此句将导致part_02 状态为UNUSABLE



**注意：对于表分区的各种操作，一定要注意更新索引**

#### 五  相关查询

##### 1.跨分区查询

```sql
select sum( *) from
(select count(*) cn from t_table_SS PARTITION (P200709_1)
union all
select count(*) cn from t_table_SS PARTITION (P200709_2)
);
```

##### 2.查询表上有多少分区

```sql
SELECT * FROM USER_TAB_PARTITIONS WHERE TABLE_NAME='tableName';
```

##### 3.查询索引信息

````sql
select object_name,object_type,tablespace_name,sum(value)
from v$segment_statistics
where statistic_name IN ('physical reads','physical write','logical reads')and object_type='INDEX'
group by object_name,object_type,tablespace_name
order by 4 desc
 
--显示数据库所有分区表的信息：
select * from DBA_PART_TABLES
 
--显示当前用户可访问的所有分区表信息:
select * from ALL_PART_TABLES
 
--显示当前用户所有分区表的信息：
select * from USER_PART_TABLES
 
--显示表分区信息 显示数据库所有分区表的详细分区信息：
select * from DBA_TAB_PARTITIONS
 
--显示当前用户可访问的所有分区表的详细分区信息：
select * from ALL_TAB_PARTITIONS
 
--显示当前用户所有分区表的详细分区信息：
select * from USER_TAB_PARTITIONS
 
--显示子分区信息 显示数据库所有组合分区表的子分区信息：
select * from DBA_TAB_SUBPARTITIONS
 
--显示当前用户可访问的所有组合分区表的子分区信息：
select * from ALL_TAB_SUBPARTITIONS
 
--显示当前用户所有组合分区表的子分区信息：
select * from USER_TAB_SUBPARTITIONS
 
--显示分区列 显示数据库所有分区表的分区列信息：
select * from DBA_PART_KEY_COLUMNS
 
--显示当前用户可访问的所有分区表的分区列信息：
select * from ALL_PART_KEY_COLUMNS
 
--显示当前用户所有分区表的分区列信息：
select * from USER_PART_KEY_COLUMNS
 
--显示子分区列 显示数据库所有分区表的子分区列信息：
select * from DBA_SUBPART_KEY_COLUMNS
 
--显示当前用户可访问的所有分区表的子分区列信息：
select * from ALL_SUBPART_KEY_COLUMNS
 
--显示当前用户所有分区表的子分区列信息：
select * from USER_SUBPART_KEY_COLUMNS
 
--怎样查询出oracle数据库中所有的的分区表
select * from user_tables a where a.partitioned='YES'
 
--删除一个表的数据是
truncate table table_name;
 
--删除分区表一个分区的数据是
alter table table_name truncate partition p5;
````





参考：

https://www.cnblogs.com/tracy/archive/2011/05/31/2064027.html

https://blog.csdn.net/mzglzzc/article/details/46300645
