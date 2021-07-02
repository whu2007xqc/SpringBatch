### 创建项目
创建一个基于Maven的Spring batch项目。
可以使用 [start.spring.io](https://start.spring.io/) 网站来协助创建

##### 配置数据库
由于Spring batch本身初始化和存储执行信息的需要，需要绑定一个数据库。

### 概念
* `JobInstance`：  
Job的实例
* `JobExecution`：  
JobInstance的一次执行
* `JobParameters`：  
Job的参数，用于Job的运行。且可以标识一个Job，如果参数相同且执行COMPLETE，则不能再执行。
* `Step`：  
步骤。一个Job由一个或多个Step。有两种类型，tasklet为简单的Step，chunk-based（reader、processor、writer）可以执行更为复杂的逻辑。  
* `StepExecution`：每次Step执行的时候都会创建一个StepExecution  
* `ExecutionContext`：  
执行上下文，以KV的方式被框架持久化。Job和每一个Step都至少有一个ExecutionContext。Step的执行上下文会在Step的每个commit点时保存，而Job的执行上下文会在两个StepExecution之间保存。  
* `JobRepository`*：  
访问数据库。  
* `JobLauncher`：  
Job的执行入口，加载Job和参数执行。  
 

### 语法
```java
    public Job jobDemoJob(){
        return jobBuilderFactory.get("jobDemoJOb")
                .start(step1())
                .on("COMPLETED").to(step2()) //step1完成后执行step2
                .from(step2()).on("COMPLETE").to(step3()) //step2满足条件后执行step3
                .from(step3()).end() //执行完成step3后结束
                .build();
    }
```
用from-on-to能无限循环吗？

##### Flow
Flow是Step的集合，可以在Job内或Job之间复用。

##### Split
多线程并发执行
```java
    public Flow flowDemo(){
        return new FlowBuilder<Flow>("flowDemo")
                .start(step2())
                .next(step3())
                .build();
    }

    public Job jobDemoJob2(){
        return jobBuilderFactory.get("jobDemoJob2")
                .start(step1())
                .split(new SimpleAsyncTaskExecutor()).add(flowDemo(),flowDemo())
                .end()
                .build();
    }
```
step1和flowDemo在不同的线程并发执行

### Decider决策器
from-on-to就是一种简单的决策器。
定义Decider，`.next(myDecider()).from(myDecider()).on(...).to(xxxStep())`

### 嵌套Job
父Job与子Job
```java
private Step childJob(JobRepository repository,...){

JobStepBuilder(new StepBuilder(""))
.job(childJob1)
.launcher(...)
.repository(...)
.transactionManager(...)
.build(); 
}
```
指定launch父job，不launch子job
在properties文件中指定launch的job：
`spring.batch.job.names=xxxJob`

### Listener监听器
一种控制Job作业流的方式

JobExecutionListener
StepExecutionListener
ChunkListener
ItemReadListener, ItemProcessListener, ItemWriteListener

before/after/error方法



ListItemReader

### Reader
ItemReader
JdbcPagingItemReader
```java
@Bean
@StepScope
public JdbcPagingItemReader<Customer> dbJdbcDemoReader(){
    JdbcPagingItemReader reader = new JdbcPagingItemReader();

    reader.setDataSource(this.dataSource);
    reader.setFetchSize(100);
    reader.setRowMapper((rs,rowNum)->{
        return Customer.builder().id(rs.getLong("id"))...build();
    });

    MysqlPagingQueryProvider queryProvider = new MysqlPagingQueryProvider ();
    queryProvider.setSelectClause("id, firstName, lastName, birthdate");
    queryProvider.setFromClause("from Customer");
    Map<String, Order> sortKeys = new HashMap<>(1);
    sortKeys.put("id", Order.ASCENDING);

    reader.setQueryProvider(queryProvider);
    return reader;
}

```
###### 异常处理
通过ItemStream来管理Step的状态，通过访问上下文（ExecutionContext）的方式来维护（Map键值对），因此通过上下文使得重启一个Step变得可行。
当执行出问题时，会将最近的状态更新到JobRepository，下次job再次执行时，就会从最近的状态开始执行。

对于ItemSteam接口，open()方法会在step开始时，从DB向上下文中填充一些Value；update()在每个transaction或step执行完成后来更新上下文；close()在所有数据处理完成之后调用。

因此在update()中将当前已经执行完成了的数据条数等信息记录到上下文中，由Spring batch持久化到BATCH_STEP_EXECUTION_CONTEXT表中。

### Writer

### 错误处理
通过.faultTolerant()来获得一个容错器
##### retry
##### skip

### JobLauncher

### JobOperator
JobOperator是JobLauncher的封装，提供了更多的功能。

jobOperator.start(jobName, parameters)

Spring没有自动创建JobOperator，所以需要自己创建：
```java
SimpleJobOperator so = new SimpleJobOperator();
so.setJobLauncher(jobLauncher);
so.setJobParametersConverter(new DefaultJobParametersConverter());
so.setJobRegistry();
so.setJobExplorer();
so.setJobRegistry();
```

### 作业调度器  
@EnableScheduling：使用Spring的Schedule功能
@Scheduled：在一个方法上提供调度

```java
//每隔5秒执行
@Scheduled(fixedDelay = 5000)
public void scheduler(){
    jobOperator.startNextInstance("jobName");
}

@Bean
public Job jobName(){
    return jobBuilderFacory.get("jobName")
               .incrementer(myJobIncrementor)
               .start(jobScheduledDemoStep())
               .build();
}
```

### 其他  
[关于@EnableBatchProcessing注解](https://blog.csdn.net/Chris___/article/details/103352103)
