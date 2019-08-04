# Mybatis + Springboot系列(四) 
## 前文
#### 至此我们已经在项目中注入了这个mapperProxy类了，本文接着看看具体调用sql的时候发生了什么
## 正文
#### 用我们一开始的例子引出我们现在的内容
````
public interface UserMapper {

    @Select("SELECT * FROM entity limit #{count}")
    @Results({
            @Result(property = "id",  column = "user_sex"),
            @Result(property = "entityCode", column = "entity_code"),
            @Result(property = "entityTypeId",  column = "entity_type_id"),
    })
    List<Entity> getAll(@Param("count") Integer count);
}

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
#### 现在我们已经在Test类中得到了UserMapper类，现在准备调用userMapper.getAll(1)方法，因为我们知道我们现在真正操作的类是mapperProxy代理的mapper类，由代理模式的原理可得，我们现在会进入到代理类的invoke方法中。
````
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
````