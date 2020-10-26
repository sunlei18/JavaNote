##  SpringBoot整合Freemaker
#### 1.在pom.xml中引入依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```
#### 2.在application.yml中配置Freemaker

```
spring:
    freemarker:
        template-loader-path: classpath:/templates/
        cache: false
        charset: UTF-8
        check-template-location: true
        content-type: text/html
        expose-request-attributes: true
        expose-session-attributes: true
        request-context-attribute: request
        suffix: .ftl
```
#### 3.在resources下新建resource.properties

```
com.sunlei.opensource.name=123
com.sunlei.opensource.website=www.123.com
com.sunlei.opensource.language=java
```
#### 4.新建POJO为Resource.java

```
package com.sunlei.demo.pojo;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;

@Configuration
@ConfigurationProperties(prefix="com.sunlei.opensource")
@PropertySource(value="classpath:resource.properties")
public class Resource {
	private String name;
	private String website;
	private String language;
	
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public String getWebsite() {
		return website;
	}
	public void setWebsite(String website) {
		this.website = website;
	}
	public String getLanguage() {
		return language;
	}
	public void setLanguage(String language) {
		this.language = language;
	}
}

```
#### 5.新建控制层controller

```
@RequestMapping("/index")
public String index(ModelMap map) {
    map.addAttribute("resource", resource);
    return "freemarker/index";
}
```
#### 6.新建ftl文件

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8" />
    <title></title>
</head>
<body>
FreeMarker模板引擎
<h1>${resource.name}</h1>
<h1>${resource.website}</h1>
<h1>${resource.language}</h1>
</body>
</html>
```



******
## SpringBoot整合Thymeleaf
#### 1.在pom.xml文件中引入依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
#### 2.在application.yml中配置Thymeleaf

```
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
    mode: HTML5
    encoding: UTF-8
    servlet:
      content-type: text/html
    cache: false
```
#### 3.编辑控制层代码

```
@RequestMapping("index")
public String index(ModelMap map){
    map.addAttribute("name","thymeleaf-sunlei");
    return "thymeleaf/index";
}
```

#### 4.在templates目录下新建html文件

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8" />
    <title></title>
</head>
<body>
Thymeleaf模板引擎
<h1 th:text="${name}">hello world~~~~~~~</h1>
</body>
</html>
```

******