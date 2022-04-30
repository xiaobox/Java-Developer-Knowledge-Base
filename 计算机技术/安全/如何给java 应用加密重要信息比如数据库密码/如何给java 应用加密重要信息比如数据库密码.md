# 如何给 java 应用加密重要信息比如数据库密码

### 利用 jasypt 提供的工具类对明文密码进行加密

首先引入库：

```xml

 <!-- 配置文件加密 -->
  <dependency>
      <groupId>com.github.ulisesbocchio</groupId>
      <artifactId>jasypt-spring-boot-starter</artifactId>
  </dependency>

```

jasypt-spring-boot 组件会自动将 ENC() 语法包裹的配置项加密字段自动解密，数据得以还原。

注意这里面的：`-Djasypt.encryptor.password=QL2sIQYN2n5do6RKuZXZSkG1S5a80n`，当然也可以配置在工程配置文件里，比如这样：

```yaml
jasypt:
    encryptor:
        password: lybgeek
        algorithm: PBEWithMD5AndDES
        iv-generator-classname: org.jasypt.iv.NoIvGenerator
```

但那样就不“泄露”了吗，所以让有权限的运维配置在了 app.yaml 的 k8s 文件中。

java\_opts

```纯文本
 -Xms2048m -Xmx2048m -Djasypt.encryptor.password=QL2sIQYN2n5do6RKuZXZSkG1S5a80n
                -XX:+HeapDumpOnOutOfMemoryError
                -XX:+CrashOnOutOfMemoryError
                -XX:HeapDumpPath=/app/logs/heap-dump.hprof
                -javaagent:/opt/skywalking/skywalking-agent.jar
                -Dskywalking.trace.ignore_path=/hotel/actuator/**,/,Mysql/JDBI/Connection/close,Mysql/JDBI/Statement/execute,Mysql/JDBI/PreparedStatement/execute,/rest
```

这是一段本地加解密程序：

```java
  public static void main(String[] args) {
        // 固定配置，修改请考虑兼容性影响
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword("QL2sIQYN2n5do6RKuZXZSkG1S5a80n");
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setIvGeneratorClassName("org.jasypt.iv.RandomIvGenerator");
        config.setStringOutputType("base64");
        encryptor.setConfig(config);

        // 加密
        System.out.println("ENC(" + encryptor.encrypt("abcd") + ")");

        // 解密
//        System.out.println(encryptor.decrypt("pt//WPqBpMs99K33UBkv3zTYSTc/PTZbEUiWaR2ar4P07J3ypWnKHb91nhzl/H1MnneWvAaBOKYzuDR5y2Q9+w=="));
//        System.out.println(encryptor.decrypt("pyhnzSIZhJzfszLAWdp9EGQm+GhOiHwpN6Tld7f1UjbkgpJyR9W9a+wmlyO2UGlZ"));
//        System.out.println(encryptor.decrypt("q2V1hQjkR70P6a7PjMb7s737BWLWv+Gtk/CFZQCV38rmO0KRVvwbRNedhxgec2EdktbJz4SE/jDC/L4liKYEDAwQxmg4h3MNyjVuDh5JiQc="));

    }
```

也可以修改它的前后缀，不用 ENC() 这种：

spring-configuration-metadata.json

```json

 {
      "name": "jasypt.encryptor.property.prefix",
      "type": "java.lang.String",
      "description": "Specify a custom {@link String} to identify as prefix of encrypted properties. Default value is {@code \"ENC(\"}",
      "sourceType": "com.ulisesbocchio.jasyptspringboot.properties.JasyptEncryptorConfigurationProperties$PropertyConfigurationProperties",
      "defaultValue": "ENC("
    },
    {
      "name": "jasypt.encryptor.property.resolver-bean",
      "type": "java.lang.String",
      "description": "Specify the name of the bean to be provided for a custom {@link com.ulisesbocchio.jasyptspringboot.EncryptablePropertyResolver}. Default value is {@code \"encryptablePropertyResolver\"}",
      "sourceType": "com.ulisesbocchio.jasyptspringboot.properties.JasyptEncryptorConfigurationProperties$PropertyConfigurationProperties",
      "defaultValue": "encryptablePropertyResolver"
    },
    {
      "name": "jasypt.encryptor.property.suffix",
      "type": "java.lang.String",
      "description": "Specify a custom {@link String} to identify as suffix of encrypted properties. Default value is {@code \")\"}",
      "sourceType": "com.ulisesbocchio.jasyptspringboot.properties.JasyptEncryptorConfigurationProperties$PropertyConfigurationProperties",
      "defaultValue": ")"
    },
```

原理类似，直接修改配置文件，或加环境变量。

可以参考这个：[https://juejin.cn/post/6844904137423847438](https://juejin.cn/post/6844904137423847438 "https://juejin.cn/post/6844904137423847438")
