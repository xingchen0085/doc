​	在项目的技术选型中，持久层框架常用的有Hibernate/JPA,Mybatis，JDBC等等，虽然之前特别喜欢JPA的简洁强大，但是在一些项目上还是必须得使用Mybatis这个小巧，面向SQL的持久层框架，结合Springboot,将之前的技术融合到新技术中，所以写了这个测试demo,该测试只单独写了持久层，不包含web和业务层代码，即整合了springboot和mybatis后，再加一个单元测试。

------

测试一 「完全注解形式」

1.pom.xml 配置文件中引入 mybatis-spring-boot-starter ，支持mybatis相关

```xml
<dependency>
   <groupId>org.mybatis.spring.boot</groupId>
   <artifactId>mybatis-spring-boot-starter</artifactId>
   <version>1.3.1</version>
</dependency>
```

2.项目的父工程为 spring-boot-starter-parent , 这是springboot项目必须的。

```xml
 <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.8.RELEASE</version>
    <relativePath/>
</parent>
```

3.该测试数据库使用Mysql，因此也需要引入mysql驱动jar包。

```xml
 <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```

4.创建Mapper接口，使用注解配置SQL和方法之间的关系，接口如下：

```java
public interface UserMapper {
	
	@Select( "select * from user" )
	@Results( {
			  @Result( property = "uid", column = "id" ),
			  @Result( property = "username", column = "name" )
		  } )
  	/**查找全部用户信息*/
	public List<User> findAll();

	
	@Insert( "insert into user values (null,#{username},#{age})" )
  	/**添加用户信息*/
	public void inser( User user );


	@Delete( "delete from user where id = #{value}" )
  	/**根据 id 删除用户信息*/
	public void delete( Integer uid );
}
```

说明：

- @Result : 为了测试使用，我的实体类User与数据库user表列不一样，即当属性和列名不一致时，需要指明property和 column之间的映射关系，根据业务需要还可以配置所对应的javaType，jdbcType(数据库对应类型)，多对一和一对 一等其他复杂映射，具体可参见XML配置中的ResultMap,它们其实本质相同。


- 其他配置项可根据XML配置等推测该注解的所能表示的功能，不细述。

5.编写单元测试

```java
@RunWith( SpringRunner.class )
@SpringBootTest
/* 这里是在Spring初始化是提供Mapper所在包，该注解放在程序入口即可 */
@MapperScan( "cn.myxinge.mapper" )
public class BootDemoMybatisApplicationTests {
	/* 注入mapper */
	@Autowired
	private UserMapper userMapper;


	/**测试mybatis配置是否完成*/
	@Test
	public void contextLoads()
	{
		List<User> users = userMapper.findAll();
		System.out.println( users );
		/* ---------------------------- */
		User user = new User();
		user.setUsername( "这是测试数据 Name" );
		user.setAge( 100 );
		/* userMapper.inser(user); */

		/* ------------ */
		userMapper.delete( 10 );
	}
}
```

6.测试结束，此时可以进行简单增删改查。

---

测试二 「XML配置文件形式」

1.使用XML配置文件形式整合Mybatis和Springboot,思路和注解方式基本一致，结合之前Spring+Mybatis配置的方式，我们需要思考的点在于：

a. 告诉spring mybatis的config配置文件位置；

b. 告诉spring mybatis的mappper配置文件位置；

c. 写接口mapper

d. 注入，使用。

现在开始相关配置

2.在 application.propertis 中添加配置

```shell
mybatis.config-location=classpath:mybatis/sqlMapConfig.xml
mybatis.mapper-locations=classpath:mapper/*.xml
```

3.编写 sqlMapConfig.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
         PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
         "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
</configuration>
```

4.编写Mapper接口

```java
/**使用xml配置的Mapper*/
public interface UserMapperByXML {
	/**查找全部用户信息*/
	public List<User> findAll();


	/**添加用户信息*/
	public void inser( User user );


	/**根据 id 删除用户信息*/
	public void delete( Integer uid );
}
```

5.编写UserMapper.xml

```xml
 
<?xml version="1.0" encoding="utf8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.myxinge.mapper.UserMapperByXML">
        
     <resultMap id="userMap" type="cn.myxinge.pojo.User">
         <id property="uid" column="id" jdbcType="BIGINT"></id>
         <result property="username" column="name" jdbcType="VARCHAR"></result>
         <result property="age" column="age" jdbcType="BIGINT"></result>
     </resultMap>
        
     <select id="findAll" resultMap="userMap">
          SELECT * FROM user
     </select>
            
     <insert id="inser" parameterType="cn.myxinge.pojo.User">
          insert into user values (null,#{username},#{age})
     </insert>
        
     <delete id="delete" parameterType="java.lang.Integer">
          delete from user where id = #{value}
     </delete>
</mapper>
```

6.开始测试，测试代码和上述的注解形式开发无多少区别，这里就不贴出来了，如需源码，下面会有该项目的GitHub地址。

------

总结：

​	该demo结合了Springboot和Mybatis,使用注解形式和xml配置形式搭建了最简易的持久层代码实现，出于本人也在 学习中，该测试也是自己做的笔记，所以将很多东西都简单化，没有真正深入，比如Mybatis的一些配置细节，数据库连接池的配置，缓存等等，如果哪里有错误或需要修改地方，可以留言给我，感谢。

 ![timg](C:\Users\Public\Pictures\Sample Pictures\timg.jpg)

[Demo源代码地址](https://github.com/196049736796/boot-demo/tree/master/boot-demo-mybatis)

