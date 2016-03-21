# pstop
一个基于MySQL performance schema的诊断工具

对比Oracle的ASH/AWR，performance_schema(以下简称为P_S)很多信息都有了，
但是一个很常用的针对于sql级别的统计信息还是不太行，如这个级别的bg，io，cpu等，
另外如果想获取某个时间段的信息还不怎么方便以及历史某个时间段的报告也不行:

为了方便即时诊断，特使用Perl脚本开发了如下工具 pstop ,本脚本兼容 MySQL 5.6和5.7。
开源地址:https://github.com/noodba

pstop包括如下16项功能:
view (default: 'sys_stats')
Possible values: 'table_io_latency', 'table_io_ops', 'file_io_latency‘, 'table_lock_latency', 'top_sql_latency',
'top_sql_exe' , 'top_sql_lock', 'top_sql_examined', 'top_sql_tmptab', 'sys_stats' ,'top_mutex_latency‘, 
'top_event','stages_latency','blocking_tree','long_op' and 'sql:DIGEST_VALUE'

--Oracle也有个类似的工具 http://www.noodba.com/?p=400


<h2>pstop典型的应用场景:</h2>
• 系统负载增加，通过slow sql或者ative sql抓取不到对应的语句 
  这种场景主要对应于一些小的sql执行频率大幅增加，可通过top_sql_exe等指标进行诊断
• 磁盘IO问题
可通过指标file_io_latency、table_io_latency、top_sql_examined、top_sql_tmptab等指标去进行诊断
• Slave延迟比较大，执行的POS不动
可通过table_io_ops、table_lock_latency等指标去进行诊断
• 系统出现很多活跃事物，对应的sql很小
可通过blocking_tree等指标进行诊断
• sys_stats可以观察整体的负载情况;sql:DIGEST_VALUE可以查询具体的SQL
• 其他的场景等等

<h2>使用步骤:</h2>

<h3>首先在管理机上安装DBI，DBD;</h3>

<h3>然后需要一个对 performance_schema  的只读用户，另外加上PROCESS权限。</h3>
具体参数说明可看帮助perl pstop.pl，常用的命令行格式如下:
perl pstop.pl -u xxx -p xxx -P xxx -h 1.1.1.1 -v table_io_ops -i 2 -n 10

<h3>其次就是调整一下instrument:</h3>

<h3>注意:增加instrument会有性能开销，请只开启需要的(默认配置下上面16项功能只有top_mutex_latency,top_event,stages_latency不可用)。</h3>

只要执行以下类似的两个语句就可以了:
<pre>
update setup_instruments set ENABLED='YES',TIMED='YES' WHERE  name like 'wait/%';
UPDATE setup_consumers
SET ENABLED = 'YES' WHERE NAME IN ( 'events_waits_current','events_waits_history',
'events_stages_current','events_stages_history','events_statements_history');
</pre>
一般情况下以上语句都是在线立即生效的，但是如memory的一些需要在启动时就设置。

<h2>输出举例:</h2>
<pre>
sys_stats:
      Time  Quests  Hcomt   RecvB   SendB   BufRRs   BufRs  BufWFs   BufWRs  DtPRs  DtPWs  LogWt   LogWRs   LogWs  LogPWs   LogWB
0111 092429  18689  11422   9.38M    8.4M    2.81M      27       0    1.32W      0      0      0       2K    1602       0   1.75M
0111 092432  22546  15591   8.66M  14.97M    3.22M       3       0    6.01W      0      0      0    5.76K    2624       0   3.88M
0111 092435  17953  11741   8.87M    9.9M     2.3M       6       0    4.43W      0      1      0    3.98K    1737       0   2.62M

top_sql_latency:
      Time   Latency |   Count   Lockt |   RowsA   RowsS   RowsE |CTmpT  SltFJ SltFRJ   SltR  SortS|                           Digest
0111 093023   55.46s |   13743    1.4s |   13743       0   13743 |    0      0      0      0      0| 8ab8cb97585df6ed56ae73d1d7d7feb5
0111 093023   26.71s |    7129 859.07ms |    7129       0    7129 |    0      0      0      0      0| 328c38113976219967af15e86df4582e
0111 093023    6.41s |    1561 85.72ms |  143909       0  143909 |    0      0      0      0      0| 1290019fcc4c58d1d9b8393347db504a
0111 093023    4.37s |      19  4.65ms |       0   15711  676688 |   18      0      0      1     18| 7f29ac044c10825bf6e4a4d219fee0ad

table_io_ops:
      Time              OP |          Fetch         Insert         Update         Delete |  TName 
0315 164620         112287 |         112287              0              0              0 |  sakila.aaa 
0315 164620          43856 |              0          43856              0              0 |  sakila.a 
</pre>
