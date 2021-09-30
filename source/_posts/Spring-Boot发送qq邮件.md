---
title: Spring Boot发送qq邮件
date: 2018-08-11 15:10:13
categories: 
  - 操作实战
tags: 
  - little_eight
---

### Spring Boot中可以直接使用JavaMailSender发送邮件。
* 新建工程，在pom.xml中引入相关依赖，注意加上版本号，不然可能有意想不到的bug。
```
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
            <version>1.2.0.RELEASE</version>
        </dependency>
```

* 配置application.yml，注意密码填的是授权码，填密码的话会报503错误。如何获取可自行百度，非常简单。
``` 
spring:
  mail:
    #设置邮箱类型为qq
    host: smtp.qq.com
    #qq邮箱账户
    username: xxx@qq.com
    #qq邮箱授权码
    password: iuxynhopeeqecb
```
配置完这些我们便可直接在测试用例中发送简单的邮件了

* 在测试类中引入JavaMailSender，在测试方法里使用它的send方法，咦，发送成功。
```
   @Autowired
    private JavaMailSender javaMailSender;
    
    @Test
    public void sendSimpleMail() {
        SimpleMailMessage message = new SimpleMailMessage();
        // 发送方邮箱账户
        message.setFrom("111@qq.com");
        // 接收方邮箱账户
        message.setTo("222@qq.com");
        message.setSubject("测试邮件标题");
        message.setText("测试邮件内容");
		// 发送邮件
        javaMailSender.send(message);
    }
```

