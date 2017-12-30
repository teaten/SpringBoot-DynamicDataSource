# Spring Boot 中使用 MyBatis 下实现多数据源动态切换，读写分离 —— 多数据源轮循

> 项目地址：[https://github.com/helloworlde/SpringBoot-DynamicDataSource/tree/roundrobin](https://github.com/helloworlde/SpringBoot-DynamicDataSource/tree/roundrobin)

> 常见错误：[https://github.com/helloworlde/SpringBoot-DynamicDataSource/blob/master/Issues.md](https://github.com/helloworlde/SpringBoot-DynamicDataSource/blob/master/Issues.md)

> 在 Spring Boot 应用中使用到了 MyBatis 作为持久层框架，添加多个数据源，实现读写分离，减少数据库的压力

> 该项目使用了一个可写数据源和多个只读数据源，为了减少数据库负载，使用轮循的方式选择只读数据源；考虑到在一个 Service 中同时会有读和写的操作，
所以本应用使用 AOP 切面通过 DAO 层的方法名切换只读数据源

> 需要注意的是，使用 DAO 层切面后不应该在 Service 类层面上加 `@Transactional` 注解，而应该添加在方法上，这也是 Spring 推荐的做法

> 动态切换数据源依赖 `configuration` 包下的4个类来实现，分别是：
> - DataSourceRoutingDataSource.java
> - DataSourceConfigurer.java
> - DynamicDataSourceContextHolder.java
> - DynamicDataSourceAspect.java

---------------------

## 创建数据库及表

- 分别创建数据库`product_master`, `product_slave_alpha`, `product_slave_beta`, `product_slave_gamma`
- 在以上数据库中分别创建表 `product`，并插入不同数据

```sql
    CREATE TABLE product_master.product(
      id INT PRIMARY KEY AUTO_INCREMENT,
      name VARCHAR(50) NOT NULL,
      price DOUBLE(10,2) NOT NULL DEFAULT 0
    );
    INSERT INTO product_master.product (name, price) VALUES('master', '1');
    
    
    CREATE TABLE product_slave_alpha.product(
      id INT PRIMARY KEY AUTO_INCREMENT,
      name VARCHAR(50) NOT NULL,
      price DOUBLE(10,2) NOT NULL DEFAULT 0
    );
    INSERT INTO product_slave_alpha.product (name, price) VALUES('slaveAlpha', '1');
    
    
    CREATE TABLE product_slave_beta.product(
      id INT PRIMARY KEY AUTO_INCREMENT,
      name VARCHAR(50) NOT NULL,
      price DOUBLE(10,2) NOT NULL DEFAULT 0
    );
    INSERT INTO product_slave_beta.product (name, price) VALUES('slaveBeta', '1');
    
    
    CREATE TABLE product_slave_gamma.product(
      id INT PRIMARY KEY AUTO_INCREMENT,
      name VARCHAR(50) NOT NULL,
      price DOUBLE(10,2) NOT NULL DEFAULT 0
    );
    INSERT INTO product_slave_gamma.product (name, price) VALUES('slaveGamma', '1');
```

## 配置数据源

- application.properties

```properties
# Master datasource config
application.server.db.master.driver-class-name=com.mysql.jdbc.Driver
application.server.db.master.url=jdbc:mysql://localhost/product_master?useSSL=false
application.server.db.master.port=3306
application.server.db.master.username=root
application.server.db.master.password=123456

# SlaveAlpha datasource config
application.server.db.slave-alpha.driver-class-name=com.mysql.jdbc.Driver
application.server.db.slave-alpha.url=jdbc:mysql://localhost/product_slave_alpha?useSSL=false
application.server.db.slave-alpha.port=3306
application.server.db.slave-alpha.username=root
application.server.db.slave-alpha.password=123456

# SlaveBeta datasource config
application.server.db.slave-beta.driver-class-name=com.mysql.jdbc.Driver
application.server.db.slave-beta.url=jdbc:mysql://localhost/product_slave_beta?useSSL=false
application.server.db.slave-beta.port=3306
application.server.db.slave-beta.username=root
application.server.db.slave-beta.password=123456

# SlaveGamma datasource config
application.server.db.slave-gamma.driver-class-name=com.mysql.jdbc.Driver
application.server.db.slave-gamma.url=jdbc:mysql://localhost/product_slave_gamma?useSSL=false
application.server.db.slave-gamma.port=3306
application.server.db.slave-gamma.username=root
application.server.db.slave-gamma.password=123456

# MyBatis config
mybatis.type-aliases-package=cn.com.hellowood.dynamicdatasource.mapper
mybatis.mapper-locations=mappers/**Mapper.xml
```

## 配置数据源

- DataSourceKey.java
```java
package cn.com.hellowood.dynamicdatasource.common;

public enum DataSourceKey {
    master,
    slaveAlpha,
    slaveBeta,
    slaveGamma
}

```

- DataSourceRoutingDataSource.java

> 该类继承自 `AbstractRoutingDataSource` 类，在访问数据库时会调用该类的 `determineCurrentLookupKey()` 
方法获取数据库实例的 key

```java
package cn.com.hellowood.dynamicdatasource.configuration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

public class DynamicRoutingDataSource extends AbstractRoutingDataSource {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Override
    protected Object determineCurrentLookupKey() {
        logger.info("Current DataSource is [{}]", DynamicDataSourceContextHolder.getDataSourceKey());
        return DynamicDataSourceContextHolder.getDataSourceKey();
    }
}

```

- DataSourceConfigurer.java

> 数据源配置类，在该类中生成多个数据源实例并将其注入到 `ApplicationContext` 中

```java
package cn.com.hellowood.dynamicdatasource.configuration;

import org.mybatis.spring.SqlSessionFactoryBean;
import org.springframework.boot.autoconfigure.jdbc.DataSourceBuilder;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class DataSourceConfigurer {

    /**
     * master DataSource
     * @Primary 注解用于标识默认使用的 DataSource Bean，因为有5个 DataSource Bean，该注解可用于 master
     * 或 slave DataSource Bean, 但不能用于 dynamicDataSource Bean, 否则会产生循环调用 
     * 
     * @ConfigurationProperties 注解用于从 application.properties 文件中读取配置，为 Bean 设置属性 
     * @return data source
     */
    @Bean("master")
    @Primary
    @ConfigurationProperties(prefix = "application.server.db.master")
    public DataSource master() {
        return DataSourceBuilder.create().build();
    }

    /**
     * Slave alpha data source.
     *
     * @return the data source
     */
    @Bean("slaveAlpha")
    @ConfigurationProperties(prefix = "application.server.db.slave-alpha")
    public DataSource slaveAlpha() {
        return DataSourceBuilder.create().build();
    }

    /**
     * Slave beta data source.
     *
     * @return the data source
     */
    @Bean("slaveBeta")
    @ConfigurationProperties(prefix = "application.server.db.slave-beta")
    public DataSource slaveBeta() {
        return DataSourceBuilder.create().build();
    }

    /**
     * Slave gamma data source.
     *
     * @return the data source
     */
    @Bean("slaveGamma")
    @ConfigurationProperties(prefix = "application.server.db.slave-gamma")
    public DataSource slaveGamma() {
        return DataSourceBuilder.create().build();
    }

    /**
     * Dynamic data source.
     *
     * @return the data source
     */
    @Bean("dynamicDataSource")
    public DataSource dynamicDataSource() {
        DynamicRoutingDataSource dynamicRoutingDataSource = new DynamicRoutingDataSource();
        Map<Object, Object> dataSourceMap = new HashMap<>(4);
        dataSourceMap.put(DataSourceKey.master.name(), master());
        dataSourceMap.put(DataSourceKey.slaveAlpha.name(), slaveAlpha());
        dataSourceMap.put(DataSourceKey.slaveBeta.name(), slaveBeta());
        dataSourceMap.put(DataSourceKey.slaveGamma.name(), slaveGamma());

        // 将 master 数据源作为默认指定的数据源
        dynamicRoutingDataSource.setDefaultTargetDataSource(master());
        // 将 master 和 slave 数据源作为指定的数据源
        dynamicRoutingDataSource.setTargetDataSources(dataSourceMap);

        // 将数据源的 key 放到数据源上下文的 key 集合中，用于切换时判断数据源是否有效
        DynamicDataSourceContextHolder.dataSourceKeys.addAll(dataSourceMap.keySet());

        // 将 Slave 数据源的 key 放在集合中，用于轮循
        DynamicDataSourceContextHolder.slaveDataSourceKeys.addAll(dataSourceMap.keySet());
        DynamicDataSourceContextHolder.slaveDataSourceKeys.remove(DataSourceKey.master.name());
        return dynamicRoutingDataSource;
    }   
    
    /**
     * 配置 SqlSessionFactoryBean
     * @ConfigurationProperties 在这里是为了将 MyBatis 的 mapper 位置和持久层接口的别名设置到 
     * Bean 的属性中，如果没有使用 *.xml 则可以不用该配置，否则将会产生 invalid bond statement 异常
     * 
     * @return the sql session factory bean
     */
    @Bean
    @ConfigurationProperties(prefix = "mybatis")
    public SqlSessionFactoryBean sqlSessionFactoryBean() {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        // 配置数据源，此处配置为关键配置，如果没有将 dynamicDataSource 作为数据源则不能实现切换
        sqlSessionFactoryBean.setDataSource(dynamicDataSource());
        return sqlSessionFactoryBean;
    }
    
    /**
     * 注入 DataSourceTransactionManager 用于事务管理
     */
    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dynamicDataSource());
    }
    
}

```

- DynamicDataSourceContextHolder.java

> 该类为数据源上下文配置，用于切换数据源

```java
package cn.com.hellowood.dynamicdatasource.configuration;


import cn.com.hellowood.dynamicdatasource.common.DataSourceKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class DynamicDataSourceContextHolder {

    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSourceContextHolder.class);

    /**
     * 用于在切换数据源时保证不会被其他线程修改
     */
    private static Lock lock = new ReentrantLock();

    /**
     * 用于轮循的计数器
     */
    private static int counter = 0;

    /**
     * Maintain variable for every thread, to avoid effect other thread
     */
    private static final ThreadLocal<String> CONTEXT_HOLDER = ThreadLocal.withInitial(DataSourceKey.master::name);


    /**
     * All DataSource List
     */
    public static List<Object> dataSourceKeys = new ArrayList<>();

    /**
     * The constant slaveDataSourceKeys.
     */
    public static List<Object> slaveDataSourceKeys = new ArrayList<>();

    /**
     * To switch DataSource
     *
     * @param key the key
     */
    public static void setDataSourceKey(String key) {
        CONTEXT_HOLDER.set(key);
    }

    /**
     * Use master data source.
     */
    public static void useMasterDataSource() {
        CONTEXT_HOLDER.set(DataSourceKey.master.name());
    }

    /**
     * 当使用只读数据源时通过轮循方式选择要使用的数据源
     */
    public static void useSlaveDataSource() {
        lock.lock();

        try {
            int datasourceKeyIndex = counter % slaveDataSourceKeys.size();
            CONTEXT_HOLDER.set(String.valueOf(slaveDataSourceKeys.get(datasourceKeyIndex)));
            counter++;
        } catch (Exception e) {
            logger.error("Switch slave datasource failed, error message is {}", e.getMessage());
            useMasterDataSource();
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    /**
     * Get current DataSource
     *
     * @return data source key
     */
    public static String getDataSourceKey() {
        return CONTEXT_HOLDER.get();
    }

    /**
     * To set DataSource as default
     */
    public static void clearDataSourceKey() {
        CONTEXT_HOLDER.remove();
    }

    /**
     * Check if give DataSource is in current DataSource list
     *
     * @param key the key
     * @return boolean boolean
     */
    public static boolean containDataSourceKey(String key) {
        return dataSourceKeys.contains(key);
    }
}


```

- DynamicDataSourceAspect.java

> 动态数据源切换的切面，切 DAO 层，通过 DAO 层方法名判断使用哪个数据源，实现数据源切换

```java
package cn.com.hellowood.dynamicdatasource.configuration;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class DynamicDataSourceAspect {
    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSourceAspect.class);

    private final String[] QUERY_PREFIX = {"select"};

    @Pointcut("execution( * cn.com.hellowood.dynamicdatasource.mapper.*.*(..))")
    public void daoAspect() {
    }

    @Before("daoAspect()")
    public void switchDataSource(JoinPoint point) {
        Boolean isQueryMethod = isQueryMethod(point.getSignature().getName());
        if (isQueryMethod) {
            DynamicDataSourceContextHolder.useSlaveDataSource();
            logger.info("Switch DataSource to [{}] in Method [{}]",
                    DynamicDataSourceContextHolder.getDataSourceKey(), point.getSignature());
        }
    }

    @After("daoAspect())")
    public void restoreDataSource(JoinPoint point) {
        DynamicDataSourceContextHolder.clearDataSourceKey();
        logger.info("Restore DataSource to [{}] in Method [{}]",
                DynamicDataSourceContextHolder.getDataSourceKey(), point.getSignature());
    }

    private Boolean isQueryMethod(String methodName) {
        for (String prefix : QUERY_PREFIX) {
            if (methodName.startsWith(prefix)) {
                return true;
            }
        }
        return false;
    }

}

```


## 配置 Product REST API 接口

- ProductController.java
   
```java
package cn.com.hellowood.dynamicdatasource.controller;

import cn.com.hellowood.dynamicdatasource.configuration.TargetDataSource;
import cn.com.hellowood.dynamicdatasource.modal.Product;
import cn.com.hellowood.dynamicdatasource.service.ProductService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/product")
public class ProduceController {

    @Autowired
    private ProductService productService;

    @GetMapping("/{id}")
    public Product getProduct(@PathVariable("id") Long productId) throws Exception {
        return productService.select(productId);
    }

    @GetMapping
    public List<Product> getAllProduct() throws Exception {
        return productService.getAllProduct();
    }

}

```

- ProductService.java
```java
package cn.com.hellowood.dynamicdatasource.service;

import cn.com.hellowood.dynamicdatasource.mapper.ProductDao;
import cn.com.hellowood.dynamicdatasource.modal.Product;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataAccessException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class ProductService {

    @Autowired
    private ProductDao productDao;

    public Product select(long productId) throws Exception {
        Product product = productDao.select(productId);
        if (product == null) {
            throw new Exception("Product:" + productId + " not found");
        }
        return product;
    }

    @Transactional(rollbackFor = DataAccessException.class)
    public boolean add(Product newProduct) throws Exception {
        Integer num = productDao.insert(newProduct);
        if (num <= 0) {
            throw new Exception("Add product failed");
        }
        return true;
    }

    public List<Product> getAllProduct() {
        return productDao.getAllProduct();
    }
}

```

- ProductDao.java

```java
package cn.com.hellowood.dynamicdatasource.mapper;

import cn.com.hellowood.dynamicdatasource.modal.Product;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

@Mapper
public interface ProductDao {
    Product select(@Param("id") long id);
    
    Integer insert(Product product);

    List<Product> getAllProduct();
}

```

- ProductMapper.xml

> 启动项目，此时访问 `/product/1` 会返回 `product_master` 数据库中 `product` 表中的所有数据，
多次访问 `/product` 会分别返回 `product_slave_alpha`、`product_slave_beta`、`product_slave_gamma` 数据库中 
`product` 表中的数据，同时也可以在看到切换数据源的 log，说明动态切换数据源是有效的

---------------

## 注意

> 在该应用中因为使用了 DAO 层的切面切换数据源，所以 `@Transactional` 注解不能加在类上，只能用于方法；有 `@Trasactional`
注解的方法无法切换数据源