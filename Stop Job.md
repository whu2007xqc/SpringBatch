### 何时Job被停止
在Spring Batch框架获得Stop指令之后，会在代码控制权回到Spring Batch框架手中时停止。  
如果代码处于执行业务代码的期间，Job的状态会保持为STOPPING。  
Spring Batch获得控制权的时机包括Tasklet、Reader、Processor、Writer执行完成之后。  

### 通过其他线程停止Job
核心代码如下：  
```java
private SingleResponse stopJob(JobExecutionRequestDO requestDO) {
    ...
    JobSynchronizationManager.register(jobExecution);
    Boolean stopResult = jobOperator.stop(jobExecutionId);
    ...
}
```
注意必须注册Job上下文`JobSynchronizationManager.register(jobExecution);`，因为时通过其他线程来停止Job，该线程是不具备Job上下文的，导致无法获得Job和Step等信息。  
具体可见[Scope 'Job' is not active for the current thread](https://github.com/whu2007xqc/SpringBatch/blob/main/%E5%BC%82%E5%B8%B8/Scope%20'Job'%20is%20not%20active%20for%20the%20current%20thread.md)

### 其他
通过jobOperator.stop方法发送Stop指令给Spring Batch，会将Job的状态置为STOPPING，并立刻返回结果stopResult。此时可以根据该结果做其他的业务处理。  

Spring Batch框架获得控制权之后，会检查Job的状态是否是STOPPING，若是则将其置为STOPPED，然后依次将其下的Step置为STOPPED。

Job被STOPPED之后，仍然会执行Job Listener的afterJob方法，因此可以确切的知道何时Job被Stop掉，并追加一些业务处理。Step也是同理。

### 异常情况
如果一个Job在STARTED状态之中，遇到了服务器宕机了，重启应用之后再去Stop该Job会产生什么结果？  
在应用重启之后，Stop指令依然可以成功发送给Spring Batch，此时Job的状态会被修改为STOPPING，但因为运行Job的线程已经不存在了，所以Job无法被置为STOPPED，Step也只能是Started。

