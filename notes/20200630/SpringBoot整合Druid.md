# SpringBoot整合Druid

## POM.xml
```xml
<!-- 整合Druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.10</version>
</dependency>
<!--自启动Druid管理后台-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.10</version>
</dependency>
```

## application.yml
```yml
#########################自定义系统配置############################
frame:
  #druid配置
  druid:
    #ip白名单（没有配置或者为空，则允许所有访问）
    allow:
    #监控页面登录用户名
    loginusername: admin
    #监控页面登录用户密码
    loginpassword: 11111
    #禁用HTML页面上的“Rest All”功能
    resetenable: false
    #ip黑名单,如果某个ip同时存在，deny优先于allow
    deny:
    #Druid访问地址
    urlmappings: /druid/*
```

## Druid配置文件映射实体
```java
package com.faep.vo;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

/**
 * 描述： Druid属性配置VO
 * 作者： Faep
 * 创建时间： 2020/6/30 9:22
 * 版本： [1.0, 2020/6/30]
 * 版权： Faep
 */
@Component
@ConfigurationProperties(prefix = "frame.druid", ignoreInvalidFields = true)
public class DruidConfigVo {
    /**
     * ip白名单（没有配置或者为空，则允许所有访问）
     */
    private String allow;
    /**
     * 监控页面登录用户名
     */
    private String loginusername;
    /**
     * 监控页面登录用户密码
     */
    private String loginpassword;
    /**
     * ip黑名单,如果某个ip同时存在，deny优先于allow
     */
    private String deny;
    /**
     * 禁用HTML页面上的“Rest All”功能
     */
    private String resetenable;
    /**
     * Druid访问地址
     */
    private String urlmappings;

    public String getAllow() {
        return allow;
    }

    public void setAllow(String allow) {
        this.allow = allow;
    }

    public String getLoginusername() {
        return loginusername;
    }

    public void setLoginusername(String loginusername) {
        this.loginusername = loginusername;
    }

    public String getLoginpassword() {
        return loginpassword;
    }

    public void setLoginpassword(String loginpassword) {
        this.loginpassword = loginpassword;
    }

    public String getDeny() {
        return deny;
    }

    public void setDeny(String deny) {
        this.deny = deny;
    }

    public String getResetenable() {
        return resetenable;
    }

    public void setResetenable(String resetenable) {
        this.resetenable = resetenable;
    }

    public String getUrlmappings() {
        return urlmappings;
    }

    public void setUrlmappings(String urlmappings) {
        this.urlmappings = urlmappings;
    }
}

```

## 根据配置初始化Druid配置
```java
package com.faep.config;

import com.alibaba.druid.support.http.StatViewServlet;
import com.faep.vo.DruidConfigVo;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

/**
 * 描述： Druid配置
 * 作者： Faep
 * 创建时间： 2020/6/30 9:01
 * 版本： [1.0, 2020/6/30]
 * 版权： Faep
 */
@Configuration
public class DruidConfig {

    @Autowired
    private DruidConfigVo druidConfigVo;

    @Bean
    public ServletRegistrationBean druidServlet() {
        ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean();
        servletRegistrationBean.setServlet(new StatViewServlet());
        servletRegistrationBean.addUrlMappings(druidConfigVo.getUrlmappings());
        Map<String, String> initParameters = new HashMap<>();
        initParameters.put("resetEnable", druidConfigVo.getResetenable()); //禁用HTML页面上的“Rest All”功能
        initParameters.put("allow", druidConfigVo.getAllow());  //ip白名单（没有配置或者为空，则允许所有访问）
        initParameters.put("loginUsername", druidConfigVo.getLoginusername());  //++监控页面登录用户名
        initParameters.put("loginPassword", druidConfigVo.getLoginpassword());  //++监控页面登录用户密码
        initParameters.put("deny", druidConfigVo.getDeny()); //ip黑名单，如果某个ip同时存在，deny优先于allow
        servletRegistrationBean.setInitParameters(initParameters);
        return servletRegistrationBean;
    }
}
```
