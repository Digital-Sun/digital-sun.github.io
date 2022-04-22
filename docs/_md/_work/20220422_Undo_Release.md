# 故障重启导致的UNDO空间撑满案例

## 事出有因

这几天，在做一年一度的数据压缩。

正进行到迁移归档数据的阶段。使用expdp加where条件的parfile导出数据，impdp加append方式导入数据。

某天晚上下班前跑着impdp脚本，偷了个懒，结果出问题了。

首先是主观上我偷懒，没把undo扩展够，也没在impdp时禁用索引，导致导入的时候撑满了undo表空间。

其次是这impdp跑着，虚拟化坏了，整个服务器重启了。

等我第二天早上来远程登录，发现我要重新加载桌面，就知道出事儿了。

## 步步深入

登录以后，先查了一下导入的那张表，发现新数据并没有导入

又试着先查一下表空间大小，看看导入占了多少空间。
```sql
SELECT TABLESPACE_NAME name,
      ROUND(used_space *8/1024/1024,1) "Used(GB)",
      ROUND(TABLESPACE_SIZE*8/1024/1024,1) "Size(GB)",
      ROUND(used_percent,1) "Percent"
 FROM DBA_TABLESPACE_USAGE_METRICS
 ORDER BY 4 DESC;
```
结果没有返回数据，看来问题不少。

检查了一下实例状态为OPEN，又检查了一下alert.log没特殊报错。于是决定重启一下看看。
```sql
select status from v$instance;

STATUS
------
OPEN

shutdown immediate；

#结果关闭的特别慢，很典型的在等某个撑满的文件释放，或者在等某个进程释放

startup

#正常启动了
```
试着换一个视图查询一下表空间使用情况，发现undo表空间确实撑满了，是现状。

```sql
select (tablespace_name) "表空间名",     
       sum(total_size) "总空间/M",     
       sum(total_free) "剩余空间/M",     
       sum(max_continue) "最大连续空间/M",     
       round(sum(total_free) / sum(total_size) * 100) "剩余百分比/ratio"    
  from ((select tablespace_name,     
                (0) total_size,     
                round(sum(bytes) / 1024 / 1024, 2) total_free,     
                round(max(bytes) / 1024 / 1024, 2) max_continue     
           from dba_free_space     
          group by tablespace_name) union all    
        (select tablespace_name, round(sum(bytes) / 1024 / 1024, 2), 0, 0     
           from dba_data_files     
          group by tablespace_name))     
 group by tablespace_name     
 order by 5 asc;
```

按理解我怀疑是我得impdp任务中途中断，没有提交，因此UNDO不会释放。

于是我查询了一下当前的dmp作业，果然又一条not running,于是试着drop生成的dmp表，结果也是undo报错
```sql
SELECT * FROM dba_datapump_jobs;

SYS_IMPORT_FULL_01 NOT RUNNING

DROP TABLE SYSTEM.SYS_IMPORT_FULL_01;

SQL 错误 [604] [60000]: ORA-00604: 递归 SQL 级别 1 出现错误
ORA-30036: 无法按 8 扩展段 (在还原表空间 'UNDOTBS1' 中)
```
说明UNDO问题还没解决。于是思路放在检查UNDO目前到底是什么情况上。
```sql
# 检查一下UNDO设置的过期时间
SELECT * FROM v$parameter WHERE name LIKE 'undo%';

undo_retention 900s

# 试着加了2个UNDO 表空间数据文件，正常添加，还是没有释放

alter tablespace UNDOTBS1 add datafile '/u01/app/oracle/oradata/jdedb/UNDOTBS07.dbf' size 1024M autoextend on next 500M maxsize unlimited;

# 检查一下UNDO是否自动管理。设置的过期时间是否自动有修改。

select to_char(begin_time, 'DD-MON-RR HH24:MI') begin_time,
to_char(end_time, 'DD-MON-RR HH24:MI') end_time, tuned_undoretention
from v$undostat order by end_time;

# 查一下UNDO总大小
select sum(a.bytes)/1024/1024/1024 as undo_size_GB from v$datafile a, v$tablespace b, dba_tablespaces c where c.contents = 'UNDO' and c.status = 'ONLINE' and b.name = c.tablespace_name and a.ts# = b.ts#;

256G
# 查一下UNDO可以的空间大小
select sum(bytes)/1024/1024 "mb" from dba_free_space where tablespace_name ='UNDOTBS1';

null

# 查一下UNDO各个段的当前状态
select tablespace_name,status,sum(bytes/1024/1024) from dba_undo_extents group by tablespace_name,status;

TABLESPACE_NAME	STATUS	SUM(BYTES/1024/1024)
UNDOTBS1	UNEXPIRED	0.6875
UNDOTBS1	EXPIRED	0.4375
UNDOTBS1	ACTIVE	262,134.375
```
到这里，其实发现有点难受了。

可以看到大部分UNDO的段都是ACTIVE的状态，理论上是对应的会话还没有提交完成，但是我的会话（IMPDP）已经中断了。

这里如果不释放，肯定没办法应用过期时间。

那么接下来的思路有两条

 - A 重建UNDO

 - B 想办法释放UNDO

## 抉择

先说方案A。

我判断是不能使用方案A的，原因是：

重建UNDO，即新建UNDO表空间，切换老的表空间。固然是最快的办法，但是256G的老UNDO仍然不会释放，意味着你不能drop老undo。

再新建256G+的新UNDO，磁盘空间剩余不足以支持后续工作。切老UNDO段的问题也没解决掉。

> 后来查undo managerment的官方Doc，也有此类说明

>The switch operation does not wait for transactions in the old undo tablespace to commit. If there are any pending transactions in the old undo tablespace, the old undo tablespace enters into a PENDING OFFLINE mode (status). In this mode, existing transactions can continue to execute, but undo records for new user transactions cannot be stored in this undo tablespace.

>An undo tablespace can exist in this PENDING OFFLINE mode, even after the switch operation completes successfully. A PENDING OFFLINE undo tablespace cannot be used by another instance, nor can it be dropped. Eventually, after all active transactions have committed, the undo tablespace automatically goes from the PENDING OFFLINE mode to the OFFLINE mode. From then on, the undo tablespace is available for other instances (in an Oracle Real Application Cluster environment).

于是决定在方案B上试一试探索一下

## 无奈的方案B

首先想到的是impdp的作业还是not running，还是要想办法中止。

因为drop不掉，那就使用常规方法，impdp命令attach到作业上，再中止作业。

```
impdp attach=SYS_IMPORT_FULL_01
kill_job
```

Impdp正常中止以后，查UNDO状态还是ACTIVE的，

等了15min也仍然是ACTIVE的，

试着又重启了一下还是ACTIVE的。

整不会了，开始查询网上查询资料，了解到Impdp中止后，为了保证数据库一致性，UNDO还是需要保留，并做recovery工作的。

使用SQL查询
```sql
SELECT
  usn,
  state,
  undoblockstotal "Total",
  undoblocksdone "Done",
  undoblockstotal -undoblocksdone "ToDo",
  DECODE(
    cputime,0, 'unknown',
    SYSDATE+(
              (
                (undoblockstotal-undoblocksdone)/(undoblocksdone/cputime))/86400
              )
            ) "Estimatedtime to complete"
FROM v$fast_start_transactions;

# 发现确实再recovering中，预估时间在第二天完成
 
USN	STATE	Total	Done	ToDo	Estimatedtime to complete
8	RECOVERING	6,701,467	1,312	6,700,155	21-4月 -22
```
我判断这个recovering完成以后，整个impdp的事务和恢复就彻底完成了，UNDO应该就能释放了。

考虑到时间还算能接受，就原地等待了。

## 最后

等了1个整天后，查询已经recovering完了，一查询表空间，发现UNDO已经释放了。

尝试重启，时间也正常，算是恢复了。可以说全靠Oracle自己机制给力。

如果要是磁盘空间足够，时间不够的话，使用方案A会比方案B合适些。

## 总结

提前做好规划。

出了问题不要盲目操作，先搞清楚情况以及操作的后果。


# 附录

两条评估数据大小的SQL
```sql
--table:
select owner,segment_name,tablespace_name,sum(bytes)/1024/1024/1024 gb from dba_segments where segment_name ='TABLE_NAME1' and owner='USER' group by owner,segment_name,tablespace_name;
--index:
select owner,segment_name,tablespace_name,sum(bytes)/1024/1024/1024 gb from dba_segments where segment_name in(select index_name from dba_indexes where table_name='TABLE_NAME1' and owner='USER') and owner='USER' group by rollup(owner,segment_name,tablespace_name);
```