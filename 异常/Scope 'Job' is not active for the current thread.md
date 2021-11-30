具体的报错信息为：  
> org.springframework.beans.factory.BeanCreationException: 
> Error creating bean with name 'scopedTarget.caseFilingDetailReportStep': 
> Scope 'Job' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton;
> nested exception is java.lang.IllegalStateException: No context holder available for job scope

### 问题解析
1. Spring的单例模式(singleton)与线程安全机制  
[Spring在SingleTon模式下的线程安全](https://blog.csdn.net/fuzhongmin05/article/details/100849867)  
要点在于Spring Bean默认为singleton的；以及为了保证线程安全，Spring使用ThreadLocalMap和ThreadLocal来保存每个线程自己的对象。  

2. Spring Batch的@JobScope和@StepScope  
[Spring Batch官网定义](https://docs.spring.io/spring-batch/docs/current/reference/html/index-single.html#step-scope)  
要点在于@JobScope和@StepScope都是懒加载的。@JobScope在Job启动时初始化，且保证在运行Job的时候只有一个Bean的实例；@StepScope在Step启动时初始化。  

所以如果Step上加了@JobScope注解，那么Step的Bean为延迟加载的。对于Start Job来说，会在Job会话中将该Step Bean初始化，而对于Stop Job来说，因为该Step Bean无法初始化导致失败。  


### 解决方法
方法1：去掉Step上的@JobScope注解，让Step的Bean随容器初始化。  
方法2：注册Job上下文，使其在创建Step Bean时能获取到Job参数，正常创建Bean。




其他人遇到相同的问题：https://blog.csdn.net/loveuserzzz/article/details/118486301
