在Spring batch分区分页1中使用MyBatisPagingItemReader实现了分页查询，但最后遗留了一个问题无法解决。

需要使用ItemReader，自己实现分页查询来回避这个问题，只要在Reader中return不为null就会继续循环，否则终止循环。

### 代码实现
##### Reader
```java
@StepScope
@Component
public class InternalkeyWordDetailReader implements ItemReader<List<IntrlKeyWordRptPO>> {

    @Value("#{stepExecutionContext['indexRange']}")
    private Long indexRange;

    @Value("#{stepExecutionContext['queryCount']}")
    private Integer totalQueryCount;

    @Value("#{stepExecutionContext['partitionNum']}")
    private Integer partitionNum;

    @Value(REPORT_DATA_DATE_PARAM)
    private String dataDate;

    private volatile int page = 0;

    @Autowired
    private InternalKeyWordReportRepository repository;

    private StepExecution stepExecution;

    @BeforeStep
    public void beforeStep(final StepExecution stepExecution) {
        this.stepExecution = stepExecution;
    }

    @Override
    public List<IntrlKeyWordRptPO> read() {

        System.out.println(partitionNum + ": " + stepExecution.hashCode() + "," + stepExecution.getReadCount() + "," + page);
        if(page < totalQueryCount) {
            long beginIndex = indexRange + page * CommConstant.REPORT_PAGE_SIZE;
            List<IntrlKeyWordRptPO> list;
            if (page == totalQueryCount -1){
                list = repository.list(dataDate, beginIndex, (int)(indexRange + CommConstant.PARTITION_DATA_SIZE - beginIndex));
            } else {
                list = repository.list(dataDate, beginIndex, CommConstant.REPORT_PAGE_SIZE);
            }
            page++;
            return list;
        }
        return null;
    }
    
}
```

##### Processor
```java
@StepScope
@Component
public class InternalKeyWordProcessor implements ItemProcessor<List<IntrlKeyWordRptPO>, List<IntrlKeyWordRptPO>> {
    @Override
    public List<IntrlKeyWordRptPO> process(List<IntrlKeyWordRptPO> bizStatsDOS) {
        return bizStatsDOS;
    }
}
```

##### Writer
```java
@StepScope
@Component
public class InternalKeyWordDetailWriter implements ItemWriter<List<IntrlKeyWordRptPO>> {

    @Autowired
    private InternalKeyWordReportRepository repository;

    @Value(REPORT_DATA_DATE_PARAM)
    private String dataDate;

    @Override
    public void write(List<? extends List<IntrlKeyWordRptPO>> list) {

        // process data for insert
        List<IntrlKeyWordRptPO> intrlKeyWordRptDOList=processData((List<IntrlKeyWordRptPO>) list.get(0));

        if(!CollectionUtils.isEmpty(list)){
            // insert into pbi database
            repository.batchInsertByPO((List<IntrlKeyWordRptPO>)intrlKeyWordRptDOList);
        }
    }
}
```
