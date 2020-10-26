### 1.在pom.xml中添加依赖

```
<!--mysql-->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.41</version>
</dependency>
<!--mybatis-->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.1</version>
</dependency>
<!--mapper-->
<dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper-spring-boot-starter</artifactId>
    <version>1.2.4</version>
</dependency>
<!--pagehelper-->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.2.3</version>
</dependency>
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.3.2</version>
    <scope>compile</scope>
    <optional>true</optional>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.9</version>
</dependency>
```
### 2.在yml中配置数据源

```
spring:
  datasource:
    name: leecx
    url: jdbc:mysql://localhost:3306/leecx?useSSL=false
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
    druid:
      initial-size: 1
      min-idle: 1
      max-active: 20
      test-on-borrow: true
mybatis:
  type-aliases-package: com.sunlei.demo.pojo
  mapper-locations: classpath:mapper/*.xml
mapper:
  mappers: com.sunlei.demo.utils.MyMapper
  not-empty: false
  identity: MYSQL
pagehelper:
  helperDialect: mysql
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```
### 3.在项目目录下新建xml文件generatorConfig.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <context id="MysqlContext" targetRuntime="MyBatis3Simple" defaultModelType="flat">
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>

        <plugin type="tk.mybatis.mapper.generator.MapperPlugin">
            <property name="mappers" value="com.sunlei.demo.utils.MyMapper"/>
        </plugin>

        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/leecx"
                        userId="root"
                        password="root">
        </jdbcConnection>

        <!-- 对于生成的pojo所在包 -->
        <javaModelGenerator targetPackage="com.sunlei.demo.pojo" targetProject="src/main/java"/>

		<!-- 对于生成的mapper所在目录 -->
        <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources"/>

		<!-- 配置mapper对应的java映射 -->
        <javaClientGenerator targetPackage="com.sunlei.demo.mapper" targetProject="src/main/java"
                             type="XMLMAPPER"/>


		<table tableName="sys_user"></table>
		 
    </context>
</generatorConfiguration>
```
### 4.新建工具类GeneratorDisplay来生成sql语句

```
package com.sunlei.demo.utils;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

import org.mybatis.generator.api.MyBatisGenerator;
import org.mybatis.generator.config.Configuration;
import org.mybatis.generator.config.xml.ConfigurationParser;
import org.mybatis.generator.internal.DefaultShellCallback;

public class GeneratorDisplay {

	public void generator() throws Exception{

		List<String> warnings = new ArrayList<String>();
		boolean overwrite = true;
		//指定 逆向工程配置文件
		File configFile = new File("generatorConfig.xml"); 
		ConfigurationParser cp = new ConfigurationParser(warnings);
		Configuration config = cp.parseConfiguration(configFile);
		DefaultShellCallback callback = new DefaultShellCallback(overwrite);
		MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config,
				callback, warnings);
		myBatisGenerator.generate(null);

	} 
	
	public static void main(String[] args) throws Exception {
		try {
			GeneratorDisplay generatorSqlmap = new GeneratorDisplay();
			generatorSqlmap.generator();
		} catch (Exception e) {
			e.printStackTrace();
		}
		
	}
}

```
