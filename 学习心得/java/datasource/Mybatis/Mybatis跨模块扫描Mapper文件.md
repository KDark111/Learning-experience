# Mybatis

## Mybatis跨模块扫描Mapper文件

### 所需依赖

> ```xml
>      <!--sql-->
>      <dependency>
>          <groupId>mysql</groupId>
>          <artifactId>mysql-connector-java</artifactId>
>          <version>${mysql.version}</version>
>      </dependency>
> 
>      <dependency>
>          <groupId>com.baomidou</groupId>
>          <artifactId>mybatis-plus-boot-starter</artifactId>
>      </dependency>
> ```



### application.yaml配置

> ```yaml
> mybatis-plus:
> # 指定类型别名（必须放在前面，否则会无法读取到）
> type-aliases-package: com.zwq.**.domain
> # mapper扫描位置
> mapper-locations: classpath*:mapper/**/*Mapper.xml
> configuration:
>  # 驼峰命名
>  map-underscore-to-camel-case: true
>  # 开启缓存
>  cache-enabled: true
>  # 开启自动生成主键
>  use-generated-keys: true
>  # 默认执行器类型
>  default-executor-type: simple
>  # 指定日志实现类
>  log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
> global-config:
>  db-config:
>    db-type: mysql
>    # 1 代表已删除，不配置默认是1，也可修改配置
>    logic-delete-value: 1
>    # 0 代表未删除，不配置默认是0，也可修改配置
>    logic-not-delete-value: 0
>    # 逻辑删除的表字段
>    logic-delete-field: is_deleted
>    # 使用雪花算法生成id
>    id-type: assign_id
> ```



### 其余需要注意的地方

> Mapper.xml文件需要放在各个模块resource下的mapper/**/*Mapper.xml路径中
>
> Mapper文件和ServiceImpl文件需要分别使用@Mapper和@Servcie注解,使其被spring容器管理
