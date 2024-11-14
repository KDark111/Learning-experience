# Druid配置

## 使用Druid配置动态数据源

使用Druid配置动态数据源需要注意一点

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

否则将会导致循环依赖

### 依赖需求

> ```xml
>         <!-- 阿里数据库连接池 -->
>         <dependency>
>             <groupId>com.alibaba</groupId>
>             <artifactId>druid-spring-boot-starter</artifactId>
>         </dependency>
> ```



### Application.yaml配置

> yaml文件配置数据源,但此处只是配置表,具体使用需要配合DruidConfig文件
>
> ```yaml
> # 数据源配置
> spring:
>   datasource:
>     type: com.alibaba.druid.pool.DruidDataSource
>     driverClassName: com.mysql.cj.jdbc.Driver
>     druid:
>       # 主库数据源
>       master:
>         url: jdbc:mysql://localhost:3306/ry-activiti?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
>         username: root
>         password: root
>       # 从库数据源
>       slave:
>         # 从数据源开关/默认关闭
>         enabled: false
>         url:
>         username:
>         password:
>       # 初始连接数
>       initialSize: 5
>       # 最小连接池数量
>       minIdle: 10
>       # 最大连接池数量
>       maxActive: 20
>       # 配置获取连接等待超时的时间
>       maxWait: 60000
>       # 配置连接超时时间
>       connectTimeout: 30000
>       # 配置网络超时时间
>       socketTimeout: 60000
>       # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
>       timeBetweenEvictionRunsMillis: 60000
>       # 配置一个连接在池中最小生存的时间，单位是毫秒
>       minEvictableIdleTimeMillis: 300000
>       # 配置一个连接在池中最大生存的时间，单位是毫秒
>       maxEvictableIdleTimeMillis: 900000
>       # 配置检测连接是否有效
>       validationQuery: SELECT 1 FROM DUAL
>       testWhileIdle: true
>       testOnBorrow: false
>       testOnReturn: false
>       webStatFilter:
>         enabled: true
>       statViewServlet:
>         enabled: true   
>         # 设置白名单，不填则允许所有访问
>         allow:
>         url-pattern: /druid/*
>         # 控制台管理用户名和密码
>         login-username: zwq
>         login-password: 123456
>       filter:
>         stat:
>           enabled: true
>           # 慢SQL记录
>           log-slow-sql: true
>           slow-sql-millis: 1000
>           merge-sql: true
>         wall:
>           config:
>             multi-statement-allow: true        
> ```



### DruidConfig

> ```java
> @ApiModel("druid 配置多数据源")
> @Configuration
> public class DruidConfig {
>     @Bean
>     // 将master的配置参数注入返回参数DataSource中
>     @ConfigurationProperties("spring.datasource.druid.master")
>     public DataSource masterDataSource(DruidProperties druidProperties) {
>         DruidDataSource dataSource = DruidDataSourceBuilder.create().build();
>         return druidProperties.dataSource(dataSource);
>     }
> 
>     @Bean
>     @ConfigurationProperties("spring.datasource.druid.slave")
>     // 当前缀的参数值为true时才启用此方法
>     @ConditionalOnProperty(prefix = "spring.datasource.druid.slave", name = "enabled", havingValue = "true")
>     public DataSource slaveDataSource(DruidProperties druidProperties) {
>         DruidDataSource dataSource = DruidDataSourceBuilder.create().build();
>         return druidProperties.dataSource(dataSource);
>     }
> 
>     @Bean(name = "dynamicDataSource")
>     @Primary
>     public DynamicDataSource dataSource(DataSource masterDataSource) {
>         Map<Object, Object> targetDataSources = new HashMap<>();
>         targetDataSources.put(DataSourceType.MASTER.name(), masterDataSource);
>         setDataSource(targetDataSources, DataSourceType.SLAVE.name(), "slaveDataSource");
>         return new DynamicDataSource(masterDataSource, targetDataSources);
>     }
> 
>     /**
>      * @Description 设置数据源
>      * targetDataSources 备选数据源集合
>      * @return
>      * @Author  zwq
>      * @Date  2024-11-13 17:58
>      **/
>     public void setDataSource(Map<Object, Object> targetDataSource, String sourceName, String beanName) {
>         try {
>             // SpringUtils方便在非spring管理环境中获取bean
>             DataSource dataSource = SpringUtils.getBean(beanName);
>             targetDataSource.put(sourceName, dataSource);
>         } catch (Exception e) {
> 
>         }
>     }    
>     
>     /**
>      * @Description 去除监控页面底部的广告
>      * @return
>      * @Author  zwq
>      * @Date  2024-11-13 19:12
>      **/
>     @Bean
>     @ConditionalOnProperty(name = "spring.datasource.druid.statViewServlet.enabled", havingValue = "true")
>     public FilterRegistrationBean removeDruidFilterRegistrationBean(DruidStatProperties properties) {
>         // 获取web监控页面的参数
>         DruidStatProperties.StatViewServlet config = properties.getStatViewServlet();
>         // 提取common.js的配置路径
>         String pattern = config.getUrlPattern() != null ? config.getUrlPattern() : "/druid/*";
>         String commonJsPattern = pattern.replaceAll("\\*", "js/common.js");
>         final String filePath = "support/http/resources/js/common.js";    
>         // 创建filter进行过滤
>         Filter filter = new Filter() {
>             @Override
>             public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
>                 filterChain.doFilter(servletRequest, servletResponse);
>                 // 重置缓冲区,响应头不会被重置
>                 servletResponse.resetBuffer();
>                 // 获取common.js
>                 String text = Utils.readFromResource(filePath);
>                 // 正则替换banner,除去底部的广告信息
>                 text = text.replaceAll("<a.*?banner\"></a></br>", "");
>                 text = text.replaceAll("powered.*?shrek.wang</a>", "");
>                 servletResponse.getWriter().write(text);
>             }
>         };
>         FilterRegistrationBean registrationBean = new FilterRegistrationBean();
>         registrationBean.setFilter(filter);
>         registrationBean.addUrlPatterns(commonJsPattern);
>         return registrationBean;
>     }
> }        
> ```



### DynamicDataSource

> ```java
> @ApiModel("动态数据源")
> public class DynamicDataSource extends AbstractRoutingDataSource
> {
>     public DynamicDataSource(DataSource defaultTargetDataSource, Map<Object, Object> targetDataSources)
>     {
>         super.setDefaultTargetDataSource(defaultTargetDataSource);
>         super.setTargetDataSources(targetDataSources);
>         super.afterPropertiesSet();
>     }
> 
>     @Override
>     protected Object determineCurrentLookupKey()
>     {
>         return DynamicDataSourceContextHolder.getDataSourceType();
>     }
> }
> ```



### DynamicDataSourceContextHolder

> ```java
> @ApiModel("数据源切换处理")
> public class DynamicDataSourceContextHolder {
>     public static final Logger log = LoggerFactory.getLogger(DynamicDataSourceContextHolder.class);
> 
>     /**
>      * 使用ThreadLocal维护变量，ThreadLocal为每个使用该变量的线程提供独立的变量副本，
>      * 所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。
>      */
>     private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();
> 
>     /**
>      * 设置数据源的变量
>      */
>     public static void setDataSourceType(String dsType)
>     {
>         log.info("切换到{}数据源", dsType);
>         CONTEXT_HOLDER.set(dsType);
>     }
> 
>     /**
>      * 获得数据源的变量
>      */
>     public static String getDataSourceType()
>     {
>         return CONTEXT_HOLDER.get();
>     }
> 
>     /**
>      * 清空数据源变量
>      */
>     public static void clearDataSourceType()
>     {
>         CONTEXT_HOLDER.remove();
>     }
> }
> ```



### DruidProperties

> 获取yaml文件中的Druid配置参数,并配置给数据源
>
> ```java
> @ApiModel("druid 配置属性")
> @Configuration
> public class DruidProperties {
>     @Value("${spring.datasource.druid.initialSize}")
>     private int initialSize;
> 
>     @Value("${spring.datasource.druid.minIdle}")
>     private int minIdle;
> 
>     @Value("${spring.datasource.druid.maxActive}")
>     private int maxActive;
> 
>     @Value("${spring.datasource.druid.maxWait}")
>     private int maxWait;
> 
>     @Value("${spring.datasource.druid.connectTimeout}")
>     private int connectTimeout;
> 
>     @Value("${spring.datasource.druid.socketTimeout}")
>     private int socketTimeout;
> 
>     @Value("${spring.datasource.druid.timeBetweenEvictionRunsMillis}")
>     private int timeBetweenEvictionRunsMillis;
> 
>     @Value("${spring.datasource.druid.minEvictableIdleTimeMillis}")
>     private int minEvictableIdleTimeMillis;
> 
>     @Value("${spring.datasource.druid.maxEvictableIdleTimeMillis}")
>     private int maxEvictableIdleTimeMillis;
> 
>     @Value("${spring.datasource.druid.validationQuery}")
>     private String validationQuery;
> 
>     @Value("${spring.datasource.druid.testWhileIdle}")
>     private boolean testWhileIdle;
> 
>     @Value("${spring.datasource.druid.testOnBorrow}")
>     private boolean testOnBorrow;
> 
>     @Value("${spring.datasource.druid.testOnReturn}")
>     private boolean testOnReturn;    
> 
>     public DruidDataSource dataSource(DruidDataSource datasource) {
>         /** 配置初始化大小、最小、最大 */
>         datasource.setInitialSize(initialSize);
>         datasource.setMaxActive(maxActive);
>         datasource.setMinIdle(minIdle);
> 
>         /** 配置获取连接等待超时的时间 */
>         datasource.setMaxWait(maxWait);
> 
>         /** 配置驱动连接超时时间，检测数据库建立连接的超时时间，单位是毫秒 */
>         datasource.setConnectTimeout(connectTimeout);
> 
>         /** 配置网络超时时间，等待数据库操作完成的网络超时时间，单位是毫秒 */
>         datasource.setSocketTimeout(socketTimeout);
> 
>         /** 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 */
>         datasource.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
> 
>         /** 配置一个连接在池中最小、最大生存的时间，单位是毫秒 */
>         datasource.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
>         datasource.setMaxEvictableIdleTimeMillis(maxEvictableIdleTimeMillis);    
> 
>         /**
>          * 用来检测连接是否有效的sql，要求是一个查询语句，常用select 'x'。如果validationQuery为null，testOnBorrow、testOnReturn、testWhileIdle都不会起作用。
>          */
>         datasource.setValidationQuery(validationQuery);
>         /** 建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效。 */
>         datasource.setTestWhileIdle(testWhileIdle);
>         /** 申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。 */
>         datasource.setTestOnBorrow(testOnBorrow);
>         /** 归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。 */
>         datasource.setTestOnReturn(testOnReturn);
>         return datasource;
>     }
> }        
> ```



### DataSourceType

> ```java
> @ApiModel("数据源")
> public enum DataSourceType {
>     /**
>      * 主库
>      */
>     MASTER,
> 
>     /**
>      * 从库
>      */
>     SLAVE
> }
> ```



### SpringUtils

> ```java
> @ApiModel("spring工具类 方便在非spring管理环境中获取bean")
> @Component
> public final class SpringUtils implements BeanFactoryPostProcessor, ApplicationContextAware
> {
>     /** Spring应用上下文环境 */
>     private static ConfigurableListableBeanFactory beanFactory;
> 
>     private static ApplicationContext applicationContext;
> 
>     @Override
>     public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException
>     {
>         SpringUtils.beanFactory = beanFactory;
>     }
> 
>     @Override
>     public void setApplicationContext(ApplicationContext applicationContext) throws BeansException
>     {
>         SpringUtils.applicationContext = applicationContext;
>     }
>     
>     /**
>      * 获取对象
>      *
>      * @param name
>      * @return Object 一个以所给名字注册的bean的实例
>      * @throws org.springframework.beans.BeansException
>      *
>      */
>     @SuppressWarnings("unchecked")
>     public static <T> T getBean(String name) throws BeansException
>     {
>         return (T) beanFactory.getBean(name);
>     }
> 
>     /**
>      * 获取类型为requiredType的对象
>      *
>      * @param clz
>      * @return
>      * @throws org.springframework.beans.BeansException
>      *
>      */
>     public static <T> T getBean(Class<T> clz) throws BeansException
>     {
>         T result = (T) beanFactory.getBean(clz);
>         return result;
>     }
> 
>     /**
>      * 如果BeanFactory包含一个与所给名称匹配的bean定义，则返回true
>      *
>      * @param name
>      * @return boolean
>      */
>     public static boolean containsBean(String name)
>     {
>         return beanFactory.containsBean(name);
>     }
> 
>     /**
>      * 判断以给定名字注册的bean定义是一个singleton还是一个prototype。 如果与给定名字相应的bean定义没有被找到，将会抛出一个异常（NoSuchBeanDefinitionException）
>      *
>      * @param name
>      * @return boolean
>      * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
>      *
>      */
>     public static boolean isSingleton(String name) throws NoSuchBeanDefinitionException
>     {
>         return beanFactory.isSingleton(name);
>     }
> 
>     /**
>      * @param name
>      * @return Class 注册对象的类型
>      * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
>      *
>      */
>     public static Class<?> getType(String name) throws NoSuchBeanDefinitionException
>     {
>         return beanFactory.getType(name);
>     }
> 
>     /**
>      * 如果给定的bean名字在bean定义中有别名，则返回这些别名
>      *
>      * @param name
>      * @return
>      * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
>      *
>      */
>     public static String[] getAliases(String name) throws NoSuchBeanDefinitionException
>     {
>         return beanFactory.getAliases(name);
>     }
> 
>     /**
>      * 获取aop代理对象
>      *
>      * @param invoker
>      * @return
>      */
>     @SuppressWarnings("unchecked")
>     public static <T> T getAopProxy(T invoker)
>     {
>         return (T) AopContext.currentProxy();
>     }
> 
>     /**
>      * 获取当前的环境配置，无配置返回null
>      *
>      * @return 当前的环境配置
>      */
>     public static String[] getActiveProfiles()
>     {
>         return applicationContext.getEnvironment().getActiveProfiles();
>     }
> 
>     /**
>      * 获取当前的环境配置，当有多个环境配置时，只获取第一个
>      *
>      * @return 当前的环境配置
>      */
>     public static String getActiveProfile()
>     {
>         final String[] activeProfiles = getActiveProfiles();
>         return StringUtils.isNotEmpty(activeProfiles) ? activeProfiles[0] : null;
>     }
> 
>     /**
>      * 获取配置文件中的值
>      *
>      * @param key 配置文件的key
>      * @return 当前的配置文件的值
>      *
>      */
>     public static String getRequiredProperty(String key)
>     {
>         return applicationContext.getEnvironment().getRequiredProperty(key);
>     }    
> }
> ```

