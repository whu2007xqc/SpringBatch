根据[官方文档](https://docs.spring.io/spring-batch/docs/current/reference/html/index-single.html#javaConfig) 所说，需要实现`BatchConfigurer`接口。
最简单的方法是继承`DefaultBatchConfigurer`，然后重写createJobRepository方法或get方法。  

```java
@Configuration
public class CustomerBatchConfigurer extends DefaultBatchConfigurer {
    
    @Resource(name = "metaDruidDataSource")
    private DruidDataSource metaDataSource;
    
    @Resource(name = "metaTransactionManager")
    DataSourceTransactionManager dataSourceTransactionManager;
    
    @Override
    public JobRepository createJobRepository() throws Exception {
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(metaDataSource);
        factory.setTransactionManager(dataSourceTransactionManager);
        factory.setTablePrefix("BATCH_");
        factory.afterPropertiesSet();
        return factory.getObject();
    }
    
    @Override
    public PlatformTransactionManager getTransactionManager() { return dataSourceTransactionManager; }
    
    @Override
    public JobExplorer createJobExplorer() throws Exception {
        JobExplorerFactoryBean factory = new JobExplorerFactoryBean();
        factory.setDataSource(metaDataSource);
        factory.setTablePrefix("BATCH_");
        factory.afterPropertiesSet();
        return factory.getObject();
    }
}
```

### 源码中JobRepository等对象的初始化
在SimpleBatchConfiguration类中通过initialize方法对JobRepository等对象进行初始化。  
其中获取Configurer通过的是getConfigurer方法，此方法内会判断用户是否自定义了Congfigurer，如果是则返回用户自定的，否则使用DefaultBatchConfigurer。  
