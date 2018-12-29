---
title: springboot 配置Druid多数据源
date: 2018-11-22 18:21:09

categories: 数据库
tags: 
  - Druid
---
在单数据源的情况下，Spring Boot的配置非常简单，只需要在application.yml文件中配置连接参数即可。但是往往随着业务量发展，我们通常会进行数据库拆分或是引入其他数据库，从而我们需要配置多个数据源，
#### 配置文件
```yaml
spring:
  autoconfigure:
    ## 多数据源环境下必须排除掉 DataSourceAutoConfiguration，否则会导致循环依赖报错
    exclude:
    - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
  datasource:
    ## 以 'spring.datasource' 和 'spring.datasource.druid' 开头的属性会作为公共配置，注入到每一个数据源
    driver-class-name: org.postgresql.Driver
    username: 123
    password: 123
    druid:
      ## 多数据源的标识，若该属性存在则为多数据源环境，不存在则为单数据源环境
      data-sources:
        ### master 数据源的配置，以下为 master 数据源独有的配置
        master:
          url: jdbc:postgresql://127.0.0.1:333
          username: hello_pivot
          password: hello_pivot
        ### slave 数据源的配置，以下为 slave 数据源独有的配置
        slave:
          url: jdbc:postgresql://127.0.0.1:333
          username: hello_pivot
          password: hello_pivot
```
#### 主数据源配置
```java
import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceBuilder;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.SqlSessionTemplate;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;

import javax.sql.DataSource;

/**
 * @Author: jesse
 * @Date: 2018/11/12
 * @Time: 6:56 PM
 */
@Configuration
@MapperScan(basePackages = {"com.hellobike.pivot.ehr.dal.mapper"}, sqlSessionTemplateRef = "masterSqlSessionTemplate")
public class MasterDataSourceConfig {
    @Bean(name = "masterDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.druid.data-sources.master")
    @Primary
    public DruidDataSource setDataSource() {
        return DruidDataSourceBuilder.create().build();
    }
    @Bean(name = "masterTransactionManager")
    @Primary
    public DataSourceTransactionManager setTransactionManager(@Qualifier("masterDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "masterSqlSessionFactory")
    @Primary
    public SqlSessionFactory setSqlSessionFactory(@Qualifier("masterDataSource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        try {
            bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mybatis/mapper/**/*.xml"));
            return bean.getObject();
        } catch (Exception e) {
            e.printStackTrace();
            LogUtils.ERROR.error("masterSqlSessionFactory create fail",e);
            throw new RuntimeException(e);
        }
    }

    @Bean(name = "masterSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate setSqlSessionTemplate(@Qualifier("masterSqlSessionFactory") SqlSessionFactory sqlSessionFactory)  {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}     
```
**通过不同数据源扫描不同目录下mapper.xml实现**
