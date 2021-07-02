# 使用MyBatisPagingItemReader

### 背景
3500万数据采取了分页Select（一次10万）和Insert（一次1万）之后，JVM内存不足的问题解决了，但又遇到了数据库连接超时的问题，当链接空闲时间超过10分钟就会被杀掉，因此需要提高查询效率。

### 解决方法
##### 采用Transactional注解  
想在每一次每次batch insert时，采用Transactional注解，传播属性为
`propagation = Propagation.REQUIRES_NEW`  
达到每次事务都能够重新获取数据库连接，来避免单个连接用时过长的问题。

因为在Spring batch框架之中，Step上会有事务，因此再Step内的方法上再加事务是不起作用的。

##### 采用Spring batch的分区读
采用Spring batch自身的事务管理机制，结合reader-processor-writter的处理框架，来分批read和write数据。  
* Spring batch**分区**的主要接口
  [全网最详细SpringBatch批处理读取分区(Paratition)文件讲解](https://blog.csdn.net/TreeShu321/article/details/110679574)
* Spring batch分区的实例
[Spring批处理分区分片](https://www.jdon.com/springboot/spring-batch-partition.html)
[Spring Batch分区示例](https://blog.csdn.net/cyan20115/article/details/106551735)
* Mybatis的**分页**读
[mybatis-spring](https://mybatis.org/spring/zh/batch.html)

##### 分区分页处理失败时的处理方式  
如果报出异常，可以在Reader中配置skip来忽略报错。  
https://blog.csdn.net/topdeveloperr/article/details/84337956

##### Mybatis插入数据库慢的问题
https://blog.csdn.net/zl1zl2zl3/article/details/105007492
https://www.jianshu.com/p/0f4b7bc4d22c

### 我的代码实现
##### Job入口主程序
```java
    @Bean("amlTranJob")
    public Job amlTranJob(@Qualifier(AML_TRAN_PRE_MASTER_STEP) Step amlTranPreMasterStep,
                          @Qualifier(AML_MAN_TRAN_AND_DISPUTE_STEP) Step amlManTranAndDisputeStep,
                          @Qualifier(AML_TRAN_STEP) Step amlTranStep, CommonJobListener commonJobListener) {
        LOGGER.accessLogger().infoLog(() -> StructureLogSupplierBean.builder().functionName("amlTranJob").reserved("amlTranJob begin").build());
        return jobBuilderFactory.get(TaskManagementJobNameMapperEnum.AML_TRAN_JOB.getJobName())
                .incrementer(new RunIdIncrementer()).start(amlManTranAndDisputeStep).next(amlTranPreMasterStep)
                .next(amlTranStep).listener(commonJobListener).build();
    }


    @Bean(AML_TRAN_PRE_MASTER_STEP)
    @JobScope
    public Step amlTranPreMasterStep(CommonPartitioner commonPartitioner,
                                     @Qualifier("masterAmlTranPreHandler") TaskExecutorPartitionHandler partitionHandler,
                                     @Qualifier(AML_TRAN_PRE_STEP) Step amlTranPreStep){

        return stepBuilderFactory.get(AML_TRAN_PRE_MASTER_STEP)
                .partitioner(AML_TRAN_PRE_STEP, commonPartitioner)
                .partitionHandler(partitionHandler)
                .build();
    }


    @Bean(AML_TRAN_PRE_STEP)
    public Step amlTranPreStep(@Qualifier("amlTranPreMybatisReader") MyBatisPagingItemReader<SatAmlTranResponsePO> amlTranPreReader,
                               @Qualifier("amlTranPreProcessor") AmlTranPreProcessor amlTranPreProcessor,
                               @Qualifier("amlTranPreWriter") AmlTranPreWriter amlTranPreWriter) {
        LOGGER.accessLogger().infoLog(() -> StructureLogSupplierBean.builder().functionName(AML_TRAN_PRE_STEP).reserved("amlTranPreStep begin").build());
        return stepBuilderFactory.get(AML_TRAN_PRE_STEP)
                .<SatAmlTranResponsePO, SatAmlTranResponsePO>chunk(CommConstant.MYSQL_BATCH_INSERT_SIZE)
                .reader(amlTranPreReader)
                .processor(amlTranPreProcessor)
                .writer(amlTranPreWriter)
                .build();
    }
```
##### 定义partitionHandler
注意这里的`handler.setGridSize`方法，原本是用来规定分区数量的，但这里用来传递表的数据量，分区数量改为在Partitioner里面计算。
```java
@Component("amlTranPrePartitionerHandler")
@JobScope
public class AmlTranPrePartitionerHandler {

    @Value(REPORT_DATA_DATE_PARAM)
    private String dataDate;

    @Autowired
    private AmlTranRepository repository;

    @Bean("masterAmlTranPreHandler")
    @JobScope
    public TaskExecutorPartitionHandler masterAmlTranPreHandler(@Qualifier("amlTranPreStep") Step amlTranPreStep) throws Exception {
        int dataCount = repository.countForPage(dataDate);

        TaskExecutorPartitionHandler handler = new TaskExecutorPartitionHandler();
        handler.setGridSize(dataCount);
        handler.setTaskExecutor(new SimpleAsyncTaskExecutor());
        handler.setStep(amlTranPreStep);
        handler.afterPropertiesSet();

        return handler;
    }
}
```
##### 定义分区partitioner
在网上的许多帖子中，需要定义分区的开始和结束的位移，用于确定每个并发在查询时数据的边界，不至于查出重复的数据。但在对于一个一天只更新一次的表，不会更新索引也不会更新数据，因此这个位移完全可以使用LIMIT来代替，只要提前算好LIMIT的参数即可，这样就不要求表一定要有自增主键，也不需要对自增主键进行计算分区了。但有几点需要注意：
1. 定义的分区数据量必须能被分页数据量整除，否则上一个分区的最后一个分页会和下一个分区的第一个分页的数据有重复
2. 必须设定每个分区的最大分页数量或Item最大数量，否则会导致每个分区都读从beginIndex往后的所有数据
3. 因为Limit越往后读越慢，如果有条件应当尽量使用自增长主键来代替limit。

```java
@Component(SpringBatchReportConstant.COMMON_PARTITIONER)
public class CommonPartitioner implements Partitioner {

    @Override
    @LoggerInErrorFormat(callType = CommConstantError.INTERNAL, errorCode = ErrorCodeNewEnum.STEP_PARTITION_ERROR,errorMessage = "CommonPartition failed.")
    public Map<String, ExecutionContext> partition(int dataCount) {

        if(CommConstant.PARTITION_DATA_SIZE % CommConstant.REPORT_PAGE_SIZE != 0){
            throw new ReportException(ErrorCodeNewEnum.STEP_PARTITION_ERROR.buildErrorCode(), "PARTITION_DATA_SIZE must be divisible by REPORT_PAGE_SIZE.");
        }

        int gridSize = (int)Math.ceil((float)dataCount/ CommConstant.PARTITION_DATA_SIZE);
        int pageNum;

        Map<String, ExecutionContext> result = new HashMap<>();
        for (int i = 1; i <= gridSize; i++) {
            pageNum = (int)Math.ceil((float)(dataCount - CommConstant.PARTITION_DATA_SIZE * (i-1)) / CommConstant.REPORT_PAGE_SIZE);
            ExecutionContext execContext = new ExecutionContext();
            execContext.putLong("indexRange",(long) CommConstant.PARTITION_DATA_SIZE * (i-1));
            execContext.putInt("partitionNum", i);
            execContext.putInt("queryCount",CommConstant.PARTITION_PAGE_NUM > pageNum ? pageNum : CommConstant.PARTITION_PAGE_NUM);
            result.put("partition" + i, execContext);
        }
        return result;
    }
}
```
需要注意`queryCount`参数的计算方法，该参数代表的是一个分区的分页总数，在Reader中来对分页进行循环。  
因为我们定义的分区大小和表实际的数据量会有差别，当前者远大于后者时，会出现分页空跑的情况。  
比如表实际只有100条数据，`select ... from ${tableName} limit 100, 10;`得出的结果为空，但是根据limit的特性，还是会查全表，导致资源和时间浪费。


##### 定义Reader
```java
@Configuration
@RefreshScope
public class AmlTranPreReader {

    @Bean("amlTranPreMybatisReader")
    @StepScope
    public MyBatisPagingItemReader<SatAmlTranResponsePO> amlTranPreReader(
            @Value("#{stepExecutionContext['indexRange']}") Long indexRange,
            @Value(REPORT_DATA_DATE_PARAM) String dataDate,
            @Qualifier("opsSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {

        MyBatisPagingItemReader<SatAmlTranResponsePO> reader = new MyBatisPagingItemReader();
        reader.setName("amlTranPreReader");
        reader.setQueryId("com.mastercard.cnswitch.ods.report.infrastructure.tunnel.database.dao.ops.OpsAmlDictDao.listAmlOnlineFinclTranData");
        reader.setSqlSessionFactory(sqlSessionFactory);
        reader.setPageSize(CommConstant.REPORT_PAGE_SIZE);
        reader.setMaxItemCount(CommConstant.PARTITION_DATA_SIZE);
        Map<String, Object> paramMap = Maps.newHashMapWithExpectedSize(2);
        String tableNameAcq= ShardingUtil.calcRptShardingTable(DateTimeUtils.str2Date(dataDate.replace("-",""),DateTimeUtils.DATE_PATTERN_EIGHT_DIGIT), ShardingTableEnum.OPS_BATCLR_SNGL_FINCL_TRAN);
        paramMap.put("indexRange", indexRange);
        paramMap.put("tbOpsBatclrSnglFinclTran", tableNameAcq);
        reader.setParameterValues(paramMap);

        return reader;
    }
}
```
* 采用的是MyBatisPagingItemReader，可以实现分页读取。
* 在其源码中定义了`_page`,`_pagesize`,`_skiprows`等参数，需要将其加入到SQL语句中才能起作用，见下文doReadPage方法。
* 注意，`reader.setMaxItemCount(CommConstant.PARTITION_DATA_SIZE)`设置了每个分区处理的数据量的上限，这个与LIMIT的使用方式相关。

```java
    protected void doReadPage() {
        Map<String, Object> parameters = new HashMap();
        if (this.parameterValues != null) {
            parameters.putAll(this.parameterValues);
        }

        parameters.put("_page", this.getPage());
        parameters.put("_pagesize", this.getPageSize());
        parameters.put("_skiprows", this.getPage() * this.getPageSize());
        if (this.results == null) {
            this.results = new CopyOnWriteArrayList();
        } else {
            this.results.clear();
        }

        this.results.addAll(this.sqlSessionTemplate.selectList(this.queryId, parameters));
    }

    protected void doJumpToPage(int itemIndex) {
    }
```
通过在Reader中指定pageSize，来循环读取数据，**当一页读取的数据量小于pageSize时，结束循环**（这是一个坑，参见下文【只用LIMIT实现分区加分页】）。

##### 定义Processor
```java
@Component("amlTranPreProcessor")
@StepScope
@RefreshScope
public class AmlTranPreProcessor implements ItemProcessor<SatAmlTranResponsePO, SatAmlTranResponsePO> {

    private final String[] groupTranType = {"22000001","21000002","21000003","21010010","21010011","21000005","21000006","21000007","21000008"};
    private final List<String> filterList = Arrays.asList(groupTranType.clone());

    @Override
    public SatAmlTranResponsePO process(SatAmlTranResponsePO item) {
        if(filterList.contains(item.getTranType()))
            return item;
        return null;
    }
}
```
虽然在Reader与Writter中是以List来读取和写入数据的，在ItemProcessor中定义的是`SatAmlTranResponsePO`，可以在Processor中进行PO与DO的转换、数据加工等。

##### 定义Writer
```java
@StepScope
@Component("amlTranPreWriter")
public class AmlTranPreWriter implements ItemWriter<SatAmlTranResponsePO> {

    @Autowired
    private AmlTranRepository repository;

    @Override
    public void write(List<? extends SatAmlTranResponsePO> list) throws Exception {
        if(!CollectionUtils.isEmpty(list) && null != list.get(0).getId()){
            repository.batchInsertByPO((List<SatAmlTranResponsePO>) list);
        }
    }
}
```


### 查询SQL

通过mybatis先计算出LIMIT的参数，对事实表限制查询数量，将where条件放在外层。
```xml
<select id="listAmlOnlineFinclTranData" resultMap="SatAmlResultMap" resultType="com.mastercard.cnswitch.ods.report.infrastructure.tunnel.database.dataobject.SatAmlTranResponsePO"
            parameterType="java.util.HashMap">
        <bind name="beginIndex" value="indexRange+_skiprows"/>
    select ...
    from (select a.*
                from ${tbOpsBatclrSnglFinclTran} a
                LIMIT #{beginIndex},#{_pagesize}
        ) t
        left join ops_dw_additional_data_112 addi on t.uuid = addi.uuid
        left outer join (select data_type,data_code,data_val_en from ops_cmps_data_dictionary where data_type = 'MCC') d1 on t.mcc = d1.data_code
        <!-- where t.group_tran_type in ('22000001','21000002','21000003','21010010','21010011','21000005','21000006','21000007','21000008') -->
</select>
```

### 问题
观察上文的SQL，因为${tbOpsBatclrSnglFinclTran} 表是大数据量的表，如果先where筛选再limit会导致每个分页查询都很慢，总时间不可接受。
如果将where条件放在limit之外，会导致查询出来的数据条数不到一个pageSize，这会导致框架认为已经查完所有数据，终止循环。
对于这种情况，在SQL中不做where筛选，将数据筛选放在processor中处理可以解决。
但是对于需要做group by汇总的情况，查询出的数据还是小于pageSize，因此只能放弃该框架，自己实现分页循环读取（“参考Spring batch分区分页2”）。
