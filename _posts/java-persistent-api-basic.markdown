#Java Persistent API基础

现在使用Java保存数据到数据库，一般不再直接使用jdbc编程接口，而使用ORM(Object Relational Mapping)组件。Java Persistent API(JPA)就是用于提供ORM功能。

##基本元素

###Entity

Entity是指一个需要持久存储的一个领域对象，经过ORM后一般对应到关系数据库的一张表，每个Entity实例存储成表的一行记录。

下面的代码实例用于定义一个Entity类，需要注意注释说明的一些要求。

```java
@Entity
public class Book {
    private String isbn;
    private String name;
}
```

#调试

##输出JPA生成的SQL的日志

在application.properties配置文件中，增加以下配置项

    spring.jpa.show_sql=true

#GeneratedValue,SequenceGenerator,TableGenerator

#数据库初始化

http://docs.spring.io/spring-boot/docs/current/reference/html/howto-database-initialization.html

##DDL

##初始数据spring.datasource.platform=

##模型升级工具

Spring Boot JPA - configuring auto reconnect
http://stackoverflow.com/questions/22684807/spring-boot-jpa-configuring-auto-reconnect
