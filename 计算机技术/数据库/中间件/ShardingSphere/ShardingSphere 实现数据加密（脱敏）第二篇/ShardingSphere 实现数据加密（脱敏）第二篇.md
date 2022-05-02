# ShardingSphere 实现数据加密（脱敏）第二篇

上一篇文章中说道数据加密分两种场景

分别是：

*   新上线业务

*   已上线业务

这篇我们对已上线业务进行模拟实验。

## 已上线业务改造

### 系统迁移前

**建表语句和配置文件**

```sql
CREATE TABLE `t_cipher_old` (
  `id` bigint(20) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `pwd` varchar(100) DEFAULT NULL,
  `mobile` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

为了模拟已经上线的业务，我们为表中造一些测试数据，并编写业务接口实现 CURD

![](https://tva1.sinaimg.cn/large/008i3skNly1guqjo1fm0cj60jt0a5tbd02.jpg)

然而需要在数据库表 t\_user 里新增一个字段叫做 pwd\_cipher，即 cipherColumn，用于存放密文数据，同时我们把 plainColumn 设置为 pwd，用于存放明文数据，而把 logicColumn 也设置为 pwd。&#x20;

```sql
ALTER TABLE test.t_cipher_old ADD pwd_cipher varchar(100) NULL;

```

由于之前的代码 SQL 就是使用 pwd 进行编写，即面向逻辑列进行 SQL 编写，所以业务代码无需改动。 通过 Apache ShardingSphere，针对新增的数据，会把明文写到 pwd 列，并同时把明文进行加密存储到 pwd\_cipher 列。 此时，由于 queryWithCipherColumn 设置为 false，对业务应用来说，依旧使用 pwd 这一明文列进行查询存储，却在底层数据库表 pwd\_cipher 上额外存储了新增数据的密文数据

配置文件如下（本文只需要关注 `encrypt` 节点部分）：

```yaml
spring:
  profiles:
    include: common-local
  shardingsphere:
    datasource:
      names: write-ds,read-ds-0
      write-ds:
        jdbcUrl: jdbc:mysql://mysql.local.test.glzhapp.com:23306/test?allowPublicKeyRetrieval=true&useSSL=false&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: Qq2e66hxnNd9MdNc
        connectionTimeoutMilliseconds: 3000
        idleTimeoutMilliseconds: 60000
        maxLifetimeMilliseconds: 1800000
        maxPoolSize: 50
        minPoolSize: 1
        maintenanceIntervalMilliseconds: 30000
      read-ds-0:
        jdbcUrl: jdbc:mysql://mysql.local.test.read1.glzhapp.com:23306/test?allowPublicKeyRetrieval=true&useSSL=false&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: Qq2e66hxnNd9MdNc
        connectionTimeoutMilliseconds: 3000
        idleTimeoutMilliseconds: 60000
        maxLifetimeMilliseconds: 1800000
        maxPoolSize: 50
        minPoolSize: 1
        maintenanceIntervalMilliseconds: 30000
    rules:
      readwrite-splitting:
        data-sources:
          glapp:
            write-data-source-name: write-ds
            read-data-source-names:
              - read-ds-0
            load-balancer-name: roundRobin # 负载均衡算法名称
        load-balancers:
          roundRobin:
            type: ROUND_ROBIN # 一共两种一种是 RANDOM（随机），一种是 ROUND_ROBIN（轮询）
      encrypt:
        encryptors:
          pwd-encryptor:
            props:
              aes-key-value: 123456abc
            type: AES
        tables:
          t_cipher_old:
            columns:
              pwd: # pwd 与 pwd_cipher 的转换映射
                plain-column: pwd # 原文列名称
                cipher-column: pwd_cipher # 加密列名称
                encryptor-name: pwd-encryptor # 加密算法名称（名称不能有下划线）
        queryWithCipherColumn: false # 是否使用加密列进行查询。在有原文列的情况下，可以使用原文列进行查询

```

此时调用业务接口，新插入的数据就会在明文列 pwd 和加密列 pwd\_cipher 同时存储数据。

![](https://tva1.sinaimg.cn/large/008i3skNly1guqkg6k31bj60sv02gmxt02.jpg)

上面整个的处理流程如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1guqj4veyc4j60wx0fadha02.jpg)

至此，改造以后时间点进入的数据都是加密的了。

### 系统迁移中

将旧的数据自行加密处理

具体到我们这个例子来讲，需要手动将 pwd 字段未加密的值全部手动加密后将密文存储到 pwd\_cipher.

形象地说，就是将空的位置手动补齐。

![](https://tva1.sinaimg.cn/large/008i3skNly1guqkkqfclvj60qr0b0wi502.jpg)

首先我们参考 ShardingSphere 的 AES 加解密码算法改造了一个工具类：

```java
/**
 * AES 加解密
 *
 * @author xiaohezi
 * @since 2021-09-23 15:49
 */
public class AesUtils {

    private static byte[] createSecretKey(String aesKey) {
        return Arrays.copyOf(DigestUtils.sha1(aesKey), 16);
    }

    /**
     * AES 加密方法
     *
     * @param plaintext 加密文本
     * @param aesKey    加密 key
     * @return
     * @throws NoSuchAlgorithmException
     * @throws InvalidKeyException
     * @throws BadPaddingException
     * @throws NoSuchPaddingException
     * @throws IllegalBlockSizeException
     */
    public static Object encrypt(String plaintext, String aesKey) throws NoSuchAlgorithmException, InvalidKeyException, BadPaddingException, NoSuchPaddingException, IllegalBlockSizeException {
        try {
            if (null == plaintext) {
                return null;
            } else {
                byte[] result = getCipher(1, aesKey).doFinal(StringUtils.getBytesUtf8(plaintext));
                return Base64.encodeBase64String(result);
            }
        } catch (GeneralSecurityException var3) {
            throw var3;
        }
    }

    /**
     * AES 解密方法
     *
     * @param ciphertext 密码
     * @param aesKey     加密 Key
     * @return
     * @throws NoSuchAlgorithmException
     * @throws InvalidKeyException
     * @throws BadPaddingException
     * @throws NoSuchPaddingException
     * @throws IllegalBlockSizeException
     */

    public static Object decrypt(String ciphertext, String aesKey) throws NoSuchAlgorithmException, InvalidKeyException, BadPaddingException, NoSuchPaddingException, IllegalBlockSizeException {
        try {
            if (null == ciphertext) {
                return null;
            } else {
                byte[] result = getCipher(2, aesKey).doFinal(Base64.decodeBase64(ciphertext));
                return new String(result, StandardCharsets.UTF_8);
            }
        } catch (GeneralSecurityException var3) {
            throw var3;
        }
    }

    private static Cipher getCipher(int decryptMode, String aesKey) throws NoSuchPaddingException, NoSuchAlgorithmException, InvalidKeyException {
        Cipher result = Cipher.getInstance(getType());
        result.init(decryptMode, new SecretKeySpec(createSecretKey(aesKey), getType()));
        return result;
    }

    public static String getType() {
        return "AES";
    }
}

```

然后为了简单演示，我的思路是用 java 程序将数据查出来以后直接更新，查询简单，更新的话用 mybatisplus 的 mapper 简单写了个自定义 sql 的方法

```java
/**
 * 根据 id 将密码的密文更新
 *
 * @param id
 * @param pwdCipher
 */
@Update("update t_cipher_old set pwd_cipher =#{pwdCipher}  where id = #{id}")
void updateCipher(@Param("id") Long id, @Param("pwdCipher") String pwdCipher);
```

下面是更新方法，注意我这里的 aesKey 和上面的配置文件是保持一致的。

```java
@Override
public void updateOldPwd() {
    QueryWrapper<CipherOldDO> wrapper = new QueryWrapper<>();
    wrapper.isNull("pwd_cipher");

    List<CipherOldDO> list = list(wrapper);

    String aesKey = "123456abc";

    try {
        for (CipherOldDO cipherOldDO : list) {

            Object encrypt = AesUtils.encrypt(cipherOldDO.getPwd(), aesKey);
            //更新密码的密文
            getBaseMapper().updateCipher(cipherOldDO.getId(), encrypt.toString());

        }
    } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
    } catch (InvalidKeyException e) {
        e.printStackTrace();
    } catch (BadPaddingException e) {
        e.printStackTrace();
    } catch (NoSuchPaddingException e) {
        e.printStackTrace();
    } catch (IllegalBlockSizeException e) {
        e.printStackTrace();
    }

}
```

程序执行完，加密列 pwd\_cipher 就有数据了。

![](https://tva1.sinaimg.cn/large/008i3skNly1guqnniyki2j60tn0fen3f02.jpg)

由于配置项中的 queryWithCipherColumn = false，所以密文一直没有被使用过。如果我们为了让系统能切到密文数据进行查询，需要将加密配置中的 queryWithCipherColumn 设置为 true。&#x20;

虽然现在业务系统通过将密文列的数据取出，解密后返回；但是，在存储的时候仍旧会存一份原文数据到明文列，这是为什么呢？ 答案是：为了能够进行系统回滚。 因为只要密文和明文永远同时存在，我们就可以通过开关项配置自由将业务查询切换到 cipherColumn 或 plainColumn。 也就是说，如果将系统切到密文列进行查询时，发现系统报错，需要回滚。那么只需将 queryWithCipherColumn = false，Apache ShardingSphere 将会还原，即又重新开始使用 plainColumn 进行查询。 处理流程如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1guqpxgr7hcj60x80fr3zz02.jpg)

### 系统迁移后

业务系统一般不可能让数据库的明文列和密文列永久同步保留，我们需要在系统稳定后将明文列数据删除。

但是删除列对于业务代码来说是不需要发动的，因为有 logicColumn 存在，用户的编写 SQL 都面向这个虚拟列，Apache ShardingSphere 就可以把这个逻辑列和底层数据表中的密文列进行映射转换。于是迁移后的加密配置即为：

```yaml
encrypt:
  encryptors:
      pwd-encryptor:
      props:
          aes-key-value: 123456abc
      type: AES
  tables:
      t_cipher_old:
      columns:
          pwd: # pwd 与 pwd_cipher 的转换映射
          cipher-column: pwd_cipher # 加密列名称
          encryptor-name: pwd-encryptor # 加密算法名称（名称不能有下划线）
  queryWithCipherColumn: true # 是否使用加密列进行查询。在有原文列的情况下，可以使用原文列进行查询
```

**在数据库中直接将 pwd 列删除**

![](https://tva1.sinaimg.cn/large/008i3skNly1guqqc2h1cbj60o809677m02.jpg)

可以看到已经没有 `pwd` 列的，只剩下加过密的 `pwd_cipher` , 从数据库这里我们已经看不出密码是什么了。然后我们调用查询接口，看到数据：

```json
{
    "code": 100000,
    "msg": "",
    "data": [
        {
            "id": 1,
            "name": "Tara",
            "pwd": "dogT",
            "mobile": "+425(864)267-129",
            "createTime": "1994-12-02 18:39:01",
            "updateTime": "2021-09-23 16:45:08"
        },
        {
            "id": 2,
            "name": "Earl",
            "pwd": "ju",
            "mobile": "+17(252)465-481",
            "createTime": "2016-10-05 15:15:43",
            "updateTime": "2021-09-23 16:45:08"
        },
        {
            "id": 3,
            "name": "Roberta",
            "pwd": "fo",
            "mobile": "+44(296)354-787",
            "createTime": "2008-10-09 17:21:36",
            "updateTime": "2021-09-23 16:45:08"
        },
        {
            "id": 4,
            "name": "Travis",
            "pwd": "brow",
            "mobile": "+77(975)452-214",
            "createTime": "2005-02-17 07:14:24",
            "updateTime": "2021-09-23 16:45:08"
        }
    ]
}
```

可以看到，数据是解密以后的样子。

其处理流程如下：

![](https://tva1.sinaimg.cn/large/008i3skNly1guqq25cgemj60wx0e8gms02.jpg)

## 参考

*   [https://shardingsphere.apache.org/document/5.0.0-beta/cn/features/encrypt/principle/](https://shardingsphere.apache.org/document/5.0.0-beta/cn/features/encrypt/principle/ "https://shardingsphere.apache.org/document/5.0.0-beta/cn/features/encrypt/principle/")
