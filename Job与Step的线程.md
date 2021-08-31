### 背景
项目要打印日志，有以下几个场景  
1. 接受到RPC调用的时候，此时Job没有启动
2. Job启动和结束时
3. Step启动和结束时
打印日志时需要打印JobId，因此要将JobId一直保存在内存中随时随地可取。

### 实验结果
因为Job时异步启动的，所以接受RPC调用的主程序和Job执行程序在不同的线程。因此主程序和Job都需要将JobId往MDC中保存一遍。  
对于Job和Step则分为两种情况：  
* 如果是简单的Tasklet，则Job与Step是相同的线程。Step中可以取到Job中保存的JobId。
* 如果使用了Partitioner，则Job与Partitioner、PartitionHandler是一个线程，而Step是单独的线程。
