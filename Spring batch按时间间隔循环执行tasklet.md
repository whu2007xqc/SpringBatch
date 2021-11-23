### 背景
有一个Job需要检查上游数据的状态，当发现有数据的时候成功返回，当发现没有数据的时候等待10分钟再次执行检查，检查超过5次后Job失败。


### 使用tasklet的返回状态
https://blog.csdn.net/u010502101/article/details/79377428


### 使用Thread.sleep()或wati()方法（不推荐）
```java
@Component
@StepScope
public class CheckETLOnlineTasklet implements Tasklet {

    @Value(REPORT_DATA_DATE_PARAM)
    private String dataDate;

    @Autowired
    CheckETLOnlineRepository checkETLOnlineRepository;

    @Override
    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws InterruptedException {

        boolean checkSuccess = false;
        int loopIndex = 1;

        List checkResultList;

        while(!checkSuccess){
            checkResultList = checkETLOnlineRepository.selectCheckResult(dataDate);
            if(checkResultList.size() > 0){
                return RepeatStatus.FINISHED;
            } else if(loopIndex == CommConstant.CHECK_ETL_TIMES){
                checkSuccess = true;
            }

            synchronized (this){
                this.wait(CommConstant.CHECK_ETL_INTERVAL * 1000);
            }

            loopIndex++;
        }

        throw new ReportException(ErrorCodeNewEnum.DATA_FILE_CHECK_FAIL.buildErrorCode(), "Online file and data is not ready");
    }
}
```


### ScheduledThreadPoolExecutor
Timer/Schedule的讲解  
https://blog.csdn.net/weixin_39605706/article/details/112179028  
具体实现：  
https://blog.csdn.net/u013822349/article/details/107957193  
Runnable 和 Callable：  
https://blog.csdn.net/neweastsun/article/details/86505865  

```java
@Component
@StepScope
public class CheckETLOnlineTasklet implements Tasklet {

    @Value(REPORT_DATA_DATE_PARAM)
    private String dataDate;

    @Autowired
    CheckETLOnlineRepository checkETLOnlineRepository;

    @Override
    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext)
            throws InterruptedException, ExecutionException {

        int scheduleResult = ScheduleUtils.scheduleDelayByExecuteNumber(new RunCheck(),
                CommConstant.CHECK_ETL_INTERVAL, CommConstant.CHECK_ETL_TIMES);

        if(scheduleResult == ScheduleUtils.ALL_SCHEDULE_COMPLETE){
            throw new ReportException(ErrorCodeNewEnum.DATA_FILE_CHECK_FAIL.buildErrorCode(), "Online file and data is not ready");
        } else {
            return RepeatStatus.FINISHED;
        }
    }


    class RunCheck implements Callable {
        @Override
        public Boolean call() {
            List checkResultList = checkETLOnlineRepository.selectCheckResult(dataDate);
            if(checkResultList.size() > 0){
                return ScheduleUtils.TERMINATE;
            } else {
                return ScheduleUtils.REPEAT;
            }
        }
    }

}
```

```java
public class ScheduleUtils {
    private static final int CORE_POOL_SIZE = 5;
    private static volatile ScheduledThreadPoolExecutor threadPool;
    
    //The schedule running result
    public static final int ALL_SCHEDULE_COMPLETE = 0;
    public static final int EXIT_DURING_SCHEDULE = 1;
    
    //Used to tell schedule repeat or not
    public static final Boolean REPEAT = true;
    public static final Boolean TERMINATE = false;
    
    private ScheduleUtiles(){}
    
    private static ScheduledThreadPoolExecutor getInstance(){
        if(threadPool == null || threadPool.isShutdown()) {
            synchronized(ScheduleUtils.class) {
                threadPool = new ScheduledThreadPoolExecutor(CORE_POOL_SIZE,
                        new BasicThreadFactory.Builder().namingPattern("schedule-pool-%d").daemon(true).build());
            }
        }
        return threadPool;
    }
    
    public static int scheduleDelayByExecuteNumber(Callable command, long intervalDelay, int execNum)
            throw ExecutionException, InterruptedException {
        ScheduledThreadPoolExecutor executor = ScheduleUtils.getInstance();
        
        for(int i=0; i< execNum; i++){
            ScheduledFuture isRepeat = executor.schedule(command, interalDelay, TimeUnit.SECONDS);
            if(!(Boolean) isRepeat.get()) {
                executor.remove((Runnable) isRepeat);
                return EXIT_DURING_SCHEDULE;
            }
        }
        return ALL_SCHEDULE_COMPLETE;
    }
}

```
