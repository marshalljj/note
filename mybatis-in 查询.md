## precondition
存在user表及User类，需要根据id做in查询。
```
CREATE TABLE `user` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(256) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```
```
public class User {
    private long id;
    private String name;

    @Override
    public String toString() {
        final StringBuffer sb = new StringBuffer("User{");
        sb.append("id=").append(id);
        sb.append(", name='").append(name).append('\'');
        sb.append('}');
        return sb.toString();
    }
}

```

## 方式一

```
@Mapper
public interface UserMapper {

    @Select({"<script>",
        "select * from user where id in (<foreach collection='ids' item='item' separator=',' >#{item}</foreach>)",
        "</script>"
    })
    List<User> selectByIdIn(@Param("ids") List<Long> ids);

}
```
这是最寻常的方式，使用了#表明用的预编方式处理，但是类似`<script><foreach>`等嵌入的标签影响了sql的可读性。

## 方式二
```
@Mapper
public interface UserMapper {

    @Select("select * from user where id in (${ids})")
    List<User> selectByIdIn(@Param("ids") String ids);

}
```
这里使用了$，用toString的方式做了字符串替换。虽然大大提高了可读性，但是存在sql注入的风险。

## 方式三

```
@Mapper
public interface UserMapper {
    @Lang(SimpleSelectInExtendedLanguageDriver.class)
    @Select("select * from user where id in (#{ids})")
    List<User> selectByIdIn(@Param("ids") String ids);
}

public class SimpleSelectInExtendedLanguageDriver extends XMLLanguageDriver {
    private final Pattern inPattern = Pattern.compile("\\(#\\{(\\w+)\\}\\)");
    @Override
    public SqlSource createSqlSource(Configuration configuration, String script, Class<?> parameterType) {
        Matcher matcher = inPattern.matcher(script);
        if (matcher.find()) {
            script = matcher.replaceAll("(<foreach collection='$1' item='item' separator=',' >#{item}</foreach>)");
        }

        script = "<script>" + script + "</script>";
        return super.createSqlSource(configuration, script, parameterType);
    }
}

```

关键在于对`org.apache.ibatis.annotations.Lang`的使用。加上`@Lang(SimpleSelectInExtendedLanguageDriver.class)`这一行会让mybatis对从@Select中获取的sql进行后置处理，将`(#{ids})`替换成`(<foreach collection='ids' item='item' separator=',' >#{item}</foreach>)`，关键源码如下：

```
private SqlSource getSqlSourceFromAnnotations(Method method, Class<?> parameterType, LanguageDriver languageDriver) {
    try {
      Class<? extends Annotation> sqlAnnotationType = getSqlAnnotationType(method);
      Class<? extends Annotation> sqlProviderAnnotationType = getSqlProviderAnnotationType(method);
      if (sqlAnnotationType != null) {
        if (sqlProviderAnnotationType != null) {
          throw new BindingException("You cannot supply both a static SQL and SqlProvider to method named " + method.getName());
        }
        Annotation sqlAnnotation = method.getAnnotation(sqlAnnotationType);
        <!--此处从注解中获取sql-->
        final String[] strings = (String[]) sqlAnnotation.getClass().getMethod("value").invoke(sqlAnnotation);
        return buildSqlSourceFromStrings(strings, parameterType, languageDriver);
      } else if (sqlProviderAnnotationType != null) {
        Annotation sqlProviderAnnotation = method.getAnnotation(sqlProviderAnnotationType);
        return new ProviderSqlSource(assistant.getConfiguration(), sqlProviderAnnotation, type, method);
      }
      return null;
    } catch (Exception e) {
      throw new BuilderException("Could not find value method on SQL annotation.  Cause: " + e, e);
    }
  }

  private SqlSource buildSqlSourceFromStrings(String[] strings, Class<?> parameterTypeClass, LanguageDriver languageDriver) {
    final StringBuilder sql = new StringBuilder();
    for (String fragment : strings) {
      sql.append(fragment);
      sql.append(" ");
    }
    <!--调用languageDriver进行后置处理-->
    return languageDriver.createSqlSource(configuration, sql.toString().trim(), parameterTypeClass);
  }
```
> org.apache.ibatis.builder.annotation.MapperAnnotationBuilder#getSqlSourceFromAnnotations

## 总结：
对mybatis进行in查询有三中方式： 方式一安全性好，可读性差； 方式二安全性差，可读性好； 方式三安全性好，可读性好； 所以推荐使用方式三。
