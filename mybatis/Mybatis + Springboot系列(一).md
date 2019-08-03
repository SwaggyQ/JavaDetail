# Mybatis + Springboot系列(一) 
## 前文
#### 本系列文章会开始分析一下Mybatis的相关源码。本系列涉及的所有代码都是以springboot为框架进行编写的。
## 正文
#### 用springboot来开启mybatis是非常方便的，有两种方法。
#### 第一种只需要在启动的主类上加一个注解。
````
@SpringBootApplication
@RestController
@MapperScan("thinkinginspirngbot.firstappbygui.mapper")

public class FirstAppByGuiApplication {

	public static void main(String[] args) {
		SpringApplication.run(FirstAppByGuiApplication.class, args);
	}
}
````
#### 然后在对应的文件夹中编写相关的mapper类就可以了。
#### 第二种是在每个对应的mapper实体类上加上@Mapper注解。
````
@Mapper
public interface UserMapper {
````
#### 一般情况下，直接在写启动类上肯定是比较方便的。
#### 接下来就是对具体的Mapper类的逻辑编写。同样有两种方法
- 注解方式
- xml方式

#### 在sql比较简单的情况下，例如我们平时自己练习的时候，注解方式是比较直观的。但是在真正业务处理时，xml方式更能描述复杂的业务场景。在本系列的文章中，一般情况下会更多采用注解的方式。这边贴一个比较简单的Mapper的逻辑例子。

````
 	@Select("SELECT * FROM entity limit #{count}")
	@Results({
	        @Result(property = "id",  column = "user_sex"),
	        @Result(property = "entityCode", column = "entity_code"),
	        @Result(property = "entityTypeId",  column = "entity_type_id"),
	})
	List<Entity> getAll(@Param("count") Integer count);
````
#### 在编写了上述的Mapper文件之后，springboot就会为我们自动注入这个bean，我们在需要的时候直接获取bean来操作就行了。
````
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void testQuery() throws Exception {
        List<Entity> entities = new LinkedList<>();
        entities = userMapper.getAll(1);
        System.out.println(entities);
    }

}
````
#### 当然别忘了在配置文件中写上你的数据库的配置信息，不然就算框架再好，也是拉不到数据的。
````
spring.datasource.url=
spring.datasource.username=
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
````

#### 然后运行启动类就会从数据库中按我们写的sql来拉取我们需要的数据，以上我们就用springboot和mybatis完成了一个简单的数据库查询的demo。接下来的文章我们会主要研究一下这个执行过程在底层发生了什么事。
