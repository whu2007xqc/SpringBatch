### Job运行时程序被异常中断  
##### 模拟系统突然宕机的情况  
如果在本地起的服务，比如通过IDEA断点，执行到断点时Stop本地程序，IDEA仍会将程序全部执行完成再关闭。此时Job仍然可以Complete。
如果在Windows任务管理器直接杀掉IDEA的Java进程和控制台窗口主进程，此时Job就一直保持在RUNNING状态。

##### 重跑Job
如果此时通过RESTART模式想来重跑Job，Spring Batch框架会报错。  
因为Job的状态不能是STARTING,STARTED,STOPPING,UNKNOWN,COMPLETED,ABANDONED;  
且不能是RUNNING状态，即Job有开始时间但没有结束时间；  

结合以上两点，此时需要通过脚本更新Spring Batch的元数据：
```sql
update BATCH_JOB_EXECUTION 
   set status = 'STOPPED', 
       END_TIME = ifnull(END_TIME, NOW())
 where JOB_EXECUTION_ID = 14154
;

update BATCH_STEP_EXECUTION 
   set status = 'STOPPED', 
       END_TIME = ifnull(END_TIME, NOW())
 where JOB_EXECUTION_ID = 14154
```

