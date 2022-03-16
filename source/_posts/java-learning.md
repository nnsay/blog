---
title: Java小白速学手册
date: 2022-03-16 17:08:34
tags:
- Code
- Java
excerpt: SpringBoot+MybatisPlus+JUnit
---
# 1. 创建项目

- 安装依赖和CLI

	```bash
  brew install java11
  brew install gradle@6
  brew tap spring-io/tap
  brew install spring-boot
  ```

- 创建新项目

  ```bash
  spring help init
  spring init --list
  spring init -d=web --build=gradle -g=neuralgalaxy.com -n point --package-name=com.neuralgalaxy.point point.zip
  unzip point.zip -d ./point
  ```
  
- 启动项目

  ```bash
  cd ./point
  ./gradlew build
  ```

	其中spring-boot CLI最用也是用的 [spring initializr](https://start.spring.io/) 进行项目模板的创建的. 详细的学习课程可查看附录课程SpringBoot2零基础入门.
	
- 代码练习: 

  - [Restful Service](https://spring.io/guides/gs/rest-service/)
  - [Spring Boot Gradle Plugin Reference Guide](https://docs.spring.io/spring-boot/docs/2.6.4/gradle-plugin/reference/htmlsingle/)


# 2. 使用数据库

使用MyBatisPlus进行数据开发, MyBatisPlush对Mybatis只做加强不做修改, 即可支持原来的xml方式执行SQL, 也支持ORM的方式操作数据库

- [MyBatisPlush官网](https://baomidou.com/)

- Maven+SpringBoot+MyBatisPlush+MySQL+JUnit的方式讲解如何通过MyBatisPlush操作数据

- 开启数据库日志

  ```yaml
  # dao层日志打开
  logging:
    level:
      com:
        neuralgalaxy:
          apps:
            dao: trace
  ```

- 代码练习: [2022版MyBatisPlus教程](https://b23.tv/wQxw0Pe) :官网在慕课网上提供的免费学习视频, 视频是

# 3. 学习视频

- [【尚硅谷雷神SpringBoot2零基础入门springboot全套完整版（spring boot2）-哔哩哔哩】](https://b23.tv/xcU4MzU) 
	- [配套学习大纲和教学笔记](https://www.yuque.com/atguigu/springboot)
- [【【尚硅谷】2022版MyBatisPlus教程（一套玩转mybatis-plus）-哔哩哔哩】](https://b23.tv/wQxw0Pe) 
- [Spring Boot 参考指南](https://www.apiref.com/spring-boot-zh/) 中文
