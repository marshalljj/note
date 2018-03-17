# 使用mybatis typehandler轻松实现加解密

## step1:新建Cipher类
```java
public class Cipher {
    private String text;

    public Cipher(String text) {
        this.text = text;
    }
    public String getText() {
        return text;
    }
}
```

## step2:对Cipher实现mybatis 的 TypeHandler
`setNonNullParameter`中加密，`getXXXResult`中解密。

```
@MappedTypes(Cipher.class)
public class CipherTypeHandler extends BaseTypeHandler<Cipher> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Cipher parameter, JdbcType jdbcType)
        throws SQLException {
        String cipher = EncryptAdaptor.encrypt(parameter.getText());
        ps.setString(i, cipher);

    }

    @Override
    public Cipher getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String cipher = rs.getString(columnName);
        String plainText = EncryptAdaptor.decrypt(cipher);
        return new Cipher(plainText);
    }

    @Override
    public Cipher getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String cipher = rs.getString(columnIndex);
        String plainText = EncryptAdaptor.decrypt(cipher);
        return new Cipher(plainText);
    }

    @Override
    public Cipher getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String cipher = cs.getString(columnIndex);
        String plainText = EncryptAdaptor.decrypt(cipher);
        return new Cipher(plainText);
    }
}
```

## step3: 注册TypeHandler

```
mybatis.type-handlers-package=com.enniu.cloud.services.fc.ialert.gateway.encrypt
```

## demo
使用时用cipher代替原先的string即可
```
public class UserEntity {
    private int id;
    private Cipher name;
    private Cipher phone;
    private Cipher idcard;
}
```
