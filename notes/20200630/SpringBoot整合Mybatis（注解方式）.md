# SpringBoot整合Mybatis（注解方式）

## pom.xml引入相关依赖


```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>
```

## 配置数据库


```yml
server:
  port: 8099

spring:
  datasource:
    username: root
    password: 11111
    url: jdbc:mysql://localhost:3306/springboot?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=UTC
    driver-class-name: com.mysql.jdbc.Driver

## 该配置节点为独立的节点，如果将这个配置放在spring的节点下，会导致配置无法被识别
mybatis:
  mapper-locations: classpath:mapping/*.xml #注意：一定要对应mapper映射xml文件的所在路径
  type-aliases-package: com.faep.entity  # 注意：对应实体类的路径

```

## 数据库新增User表


```SQL
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user`  (
  `rowguid` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '唯一标识',
  `username` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '用户姓名',
  `loginid` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '用户登录名',
  `password` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '用户登录密码',
  `phone` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '手机号',
  `lastlogintime` datetime(0) NULL DEFAULT NULL COMMENT '最近登录时间',
  `enabled` varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '是否允许登录（未锁定）1标识可以登录',
  PRIMARY KEY (`rowguid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```

## 新增用户实体


```java
package com.faep.entity;

import java.util.Date;

/**
 * 描述： 用户实体
 * 作者： Faep
 * 创建时间： 2020/6/18 8:54
 * 版本： [1.0, 2020/6/18]
 * 版权： Faep
 */
public class User {

    String rowguid;
    String username;
    String loginid;
    String password;
    String phone;
    Date lastlogintime;
    String enabled;

    public User() {
    }

    public User(String rowguid, String username, String loginid, String password, String phone, Date lastlogintime, String enabled) {
        this.rowguid = rowguid;
        this.username = username;
        this.loginid = loginid;
        this.password = password;
        this.phone = phone;
        this.lastlogintime = lastlogintime;
        this.enabled = enabled;
    }

    public String getRowguid() {
        return rowguid;
    }

    public void setRowguid(String rowguid) {
        this.rowguid = rowguid;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getLoginid() {
        return loginid;
    }

    public void setLoginid(String loginid) {
        this.loginid = loginid;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getPhone() {
        return phone;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }

    public Date getLastlogintime() {
        return lastlogintime;
    }

    public void setLastlogintime(Date lastlogintime) {
        this.lastlogintime = lastlogintime;
    }

    public String getEnabled() {
        return enabled;
    }

    public void setEnabled(String enabled) {
        this.enabled = enabled;
    }
}

```

## 新增用户Mapper接口


```java
package com.faep.mapper;

import com.faep.entity.User;
import org.apache.ibatis.annotations.*;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * 描述： 用户Mapper
 * 作者： Faep
 * 创建时间： 2020/6/18 9:10
 * 版本： [1.0, 2020/6/18]
 * 版权： Faep
 */
@Mapper
public interface UserMapper {

    /**
     * 新增一个用户
     * @param user
     * @return
     */
    @Insert("insert into user values (#{rowguid}, #{username}, #{loginid}, #{password}, #{phone}, #{lastlogintime}, #{enabled})")
    int addNewUser(User user);

    /**
     * 删除一个用户
     * @param rowguid
     * @return
     */
    @Delete("delete from user where rowguid=#{rowguid}")
    int removeUserByGuid(String rowguid);

    /**
     * 修改用户密码
     * @param user
     * @return
     */
    @Update("update user set password=#{password} where rowguid=#{rowguid}")
    int updateUserInfo(User user);

    /**
     * 根据用户Guid查询一个用户
     * @param rowguid
     * @return
     */
    @Select("select * from user where rowguid=#{rowguid}")
    User findUserByGuid(String rowguid);

    /**
     * 查询所有用户
     * @return
     */
    @Select("select * from user")
    List<User> findAllUsers();

}

```

## 新增用户Service

### 接口

```java
package com.faep.service.api;

import com.faep.entity.User;

import java.util.List;

/**
 * 描述： 用户service接口
 * 作者： Faep
 * 创建时间： 2020/6/18 9:35
 * 版本： [1.0, 2020/6/18]
 * 版权： Faep
 */
public interface IUserService {
    /**
     * 新增一个用户
     * @param user
     * @return
     */
    int addNewUser(User user);

    /**
     * 删除一个用户
     * @param rowguid
     * @return
     */
    int removeUserByGuid(String rowguid);

    /**
     * 修改用户密码
     * @param user
     * @return
     */
    int updateUserInfo(User user);

    /**
     * 根据用户Guid查询一个用户
     * @param rowguid
     * @return
     */
    User findUserByGuid(String rowguid);

    /**
     * 查询所有用户
     * @return
     */
    List<User> findAllUsers();
}

```
### 实现类

```java
package com.faep.service.impl;

import com.faep.entity.User;
import com.faep.mapper.UserMapper;
import com.faep.service.api.IUserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * 描述： 用户service实现类
 * 作者： Faep
 * 创建时间： 2020/6/18 9:36
 * 版本： [1.0, 2020/6/18]
 * 版权： Faep
 */
@Service
public class UserService implements IUserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public int addNewUser(User user) {
        return userMapper.addNewUser(user);
    }

    @Override
    public int removeUserByGuid(String rowguid) {
        return userMapper.removeUserByGuid(rowguid);
    }

    @Override
    public int updateUserInfo(User user) {
        return userMapper.updateUserInfo(user);
    }

    @Override
    public User findUserByGuid(String rowguid) {
        return userMapper.findUserByGuid(rowguid);
    }

    @Override
    public List<User> findAllUsers() {
        return userMapper.findAllUsers();
    }
}

```

## Controller调用Service

```java
package com.faep.controller;

import com.faep.entity.User;
import com.faep.service.api.IUserService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Date;
import java.util.UUID;


/**
 * 测试Controller
 * 作者： Faep
 * 创建时间： 2020/6/16 15:16
 * 版本： [1.0, 2020/6/16]
 * 版权： Faep
 */
@RestController
@RequestMapping("/test")
public class TestController {

    private static final Logger logger = LoggerFactory.getLogger(TestController.class);

    @Autowired
    private IUserService userService;

    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String test(){
        User user = new User();
        user.setRowguid(UUID.randomUUID().toString());
        user.setUsername("Faep");
        user.setLoginid("Faep");
        user.setPassword("11111");
        user.setLastlogintime(new Date());
        user.setPhone("18888888888");
        user.setEnabled("1");
        userService.addNewUser(user);

        logger.info(LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")) + "-接口被调用");
        return "新增用户成功！";
    }

}

```

## 启动类添加mapper扫描

```java
package com.faep.application;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;

/**
 * 作者： Faep
 * 创建时间： 2020/6/16 15:13
 * 版本： [1.0, 2020/6/16]
 * 描述： SpringBoot启动类
 */
@SpringBootApplication
@ComponentScan(basePackages = {"com.faep"})
@MapperScan(basePackages = {"com.faep.mapper"})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```


## 测试
日志
```log
09:54:42.752 [http-nio-8099-exec-4] INFO  com.faep.controller.TestController - 2020-06-18 09:54:42-接口被调用
```
结果

```SQL
744396a7-f65d-4020-9aaf-fa6e93b29f30	Faep	Faep	11111	18888888888	2020-06-18 09:54:43	1
```

