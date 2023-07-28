

# 今日内容

- Redis 入门
- Redis 数据类型
- Redis 常用命令
- 在 Java 中操作 Redis
- 店铺营业状态设置

功能实现：**营业状态设置**

**效果图**：

<img src="img/image1.png" alt="image1" style="zoom:50%;" /> 

选择**营业中**，客户可在小程序端下单：

<img src="img/image2.png" alt="image2" style="zoom:50%;" />

选择**打烊中**，客户无法在小程序端下单：

<img src="img/image3.png" alt="image3" style="zoom:50%;" />

# 一、Redis入门

## 1、Redis简介

Redis 是一个基于**内存**的 key-value 结构数据库。Redis 是互联网技术领域使用最为广泛的**存储中间件**。

**官网：**https://redis.io
**中文网：**https://www.redis.net.cn/

**key-value 结构存储**：

![image4](img/image4.png) 

**主要特点**：

- 基于内存存储，读写性能高  
- 适合存储热点数据（热点商品、资讯、新闻）
- 企业应用广泛

Redis 是用 C 语言开发的一个开源的高性能键值对（key-value）数据库，官方提供的数据是可以达到 100000+ 的QPS（每秒内查询次数）。它存储的 value 类型比较丰富，也被称为结构化的 NoSql 数据库。

NoSql（Not Only SQL），不仅仅是 SQL，泛指**非关系型数据库**。NoSql 数据库并不是要取代关系型数据库，而是关系型数据库的补充。

**关系型数据库（RDBMS）**：

- Mysql
- Oracle
- DB2
- SQLServer

**非关系型数据库（NoSql）**：

- Redis
- Mongo db
- MemCached

## 2、Redis下载与安装

### 2.1、Redis下载

Redis 安装包分为 windows 版和 Linux 版：

- Windows 版下载地址：https://github.com/microsoftarchive/redis/releases
- Linux 版下载地址： https://download.redis.io/releases/ 

### 2.2、Redis安装

**1）在 Windows 中安装 Redis（项目中使用）**

Redis 的 Windows 版属于绿色软件，直接解压即可使用，解压后目录结构如下：

<img src="img/image5.png" alt="image5" style="zoom:50%;" /> 

**2）在 Linux 中安装 Redis（简单了解）**

在 Linux 系统安装 Redis 步骤：

1. 将 Redis 安装包上传到 Linux
2. 解压安装包，命令：`tar -zxvf redis-4.0.0.tar.gz -C /usr/local`
3. 安装 Redis 的依赖环境 gcc，命令：`yum install gcc-c++`
4. 进入 /usr/local/redis-4.0.0，进行编译，命令：`make`
5. 进入 redis 的 src 目录进行安装，命令：`make install`

安装后重点文件说明：

- /usr/local/redis-4.0.0/src/redis-server：Redis 服务启动脚本
- /usr/local/redis-4.0.0/src/redis-cli：Redis 客户端脚本
- /usr/local/redis-4.0.0/redis.conf：Redis 配置文件

## 3、Redis 服务启动与停止

以 window 版 Redis 进行演示：

### 3.1、服务启动命令

`redis-server.exe redis.windows.conf`

<img src="img/image6.png" alt="image6" style="zoom:50%;" /> 

Redis 服务默认端口号为 **6379** ，通过快捷键 **Ctrl + C** 即可停止 Redis 服务

当 Redis 服务启动成功后，可通过客户端进行连接。

### 3.2、客户端连接命令

`redis-cli.exe`

<img src="img/image7.png" alt="image7" style="zoom:50%;" /> 

通过 redis-cli.exe 命令默认连接的是本地的 redis 服务，并且使用默认 6379 端口。也可以通过指定如下参数连接：

- -h ip地址
- -p 端口号
- -a 密码（如果需要）

### 3.3、修改 Redis 配置文件

设置 Redis 服务密码，修改 redis.windows.conf

```
requirepass 123456
```

**注意**：

- 修改密码后需要重启 Redis 服务才能生效
- Redis 配置文件中 # 表示注释

重启 Redis 后，再次连接 Redis 时，需加上密码，否则连接失败。

```
redis-cli.exe -h localhost -p 6379 -a 123456
```

<img src="img/image8.png" alt="image8" style="zoom:50%;" /> 

此时，-h 和 -p 参数可省略不写。

### 3.4、Redis 客户端图形工具

默认提供的客户端连接工具界面不太友好，同时操作也较为麻烦，接下来，引入一个 Redis 客户端图形工具。

推荐使用 Another-Redis-Desktop-Manager，下载地址：https://github.com/qishibo/AnotherRedisDesktopManager/releases

安装完毕后，直接双击启动

**新建连接**

<img src="img/image9.png" alt="image9" style="zoom: 33%;" />  

**连接成功** 

<img src="img/image10.png" alt="image10" style="zoom: 33%;" />  

# 二、Redis 数据类型

## 1、五种常用数据类型介绍

Redis 存储的是 key-value 结构的数据，其中 key 是字符串类型，value 有 5 种常用的数据类型：

- 字符串 string
- 哈希 hash
- 列表 list
- 集合 set
- 有序集合 sorted set / zset

## 2、各种数据类型特点

<img src="img/image11.png" alt="image11" style="zoom:50%;" /> 

**解释说明**：

- 字符串（string）：普通字符串，Redis 中最简单的数据类型
- 哈希（hash）：也叫散列，类似于 Java 中的 HashMap 结构
- 列表（list）：按照插入顺序排序，可以有重复元素，类似于 Java 中的 LinkedList
- 集合（set）：无序集合，没有重复元素，类似于 Java 中的 HashSet
- 有序集合（sorted set/zset）：集合中每个元素关联一个分数（score），根据分数升序排序，没有重复元素

# 三、Redis常用命令

## 1、字符串操作命令

Redis 中字符串类型常用命令：

- `SET key value`：设置指定 key 的值
- `GET key`：获取指定 key 的值
- `SETEX key seconds value`：设置指定 key 的值，并将 key 的过期时间设为 seconds 秒
- `SETNX key value`：只有在 key 不存在时设置 key 的值

更多命令可以参考 Redis 中文网：https://www.redis.net.cn

```shell
> set name jack
OK
> get name
jack
> get abc
null
> setex code 60 1234
OK
> setnx key1 itcast
1
> setnx key1 itheima
0
> get key1
itcast
```

## 2、哈希操作命令

Redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象，常用命令：

- `HSET key field value`：将哈希表 key 中的字段 field 的值设为 value
- `HGET key field`：获取存储在哈希表中指定字段的值
- `HDEL key field`：删除存储在哈希表中的指定字段
- `HKEYS key`：获取哈希表中所有字段
- `HVALS key`：获取哈希表中所有值

<img src="img/image12.png" alt="image12" style="zoom: 67%;" /> 

```shell
> hset 100 name xiaoming
1
> hset 100 age 22
1
> hget 100 name
xiaoming
> hget 100 age
22
> hdel 100 name
1
> hset 100 name itcast
1
> hkeys 100
age
name
> hvals 100
22
itcast
```

## 3、列表操作命令

Redis 列表是简单的字符串列表，按照插入顺序排序，常用命令：

- `LPUSH key value1 [value2]`：将一个或多个值插入到列表头部
- `LRANGE key start stop`：获取列表指定范围内的元素
- `RPOP key`：移除并获取列表最后一个元素
- `LLEN key`：获取列表长度
- `BRPOP key1 [key2 ] timeout`：移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止

<img src="img/image13.png" alt="image13" style="zoom: 67%;" />

```shell
> lpush mylist a b c
3
> lpush mylist d
4
> lrange mylist 0 -1
d
c
b
a
> rpop mylist
a
> rpop mylist
b
> llen mylist
2
```

## 4、集合操作命令

Redis set 是 string 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据，常用命令：

- `SADD key member1 [member2]`：向集合添加一个或多个成员
- `SMEMBERS key`：返回集合中的所有成员
- `SCARD key`：获取集合的成员数
- `SINTER key1 [key2]`：返回给定所有集合的交集
- `SUNION key1 [key2]`：返回所有给定集合的并集
- `SREM key member1 [member2]`：移除集合中一个或多个成员

<img src="img/image14.png" alt="image14" style="zoom: 67%;" /> 

```shell
> sadd set1 a b c d
4
> sadd set1 a
0
> smembers set1
b
a
d
c
> scard set1
4
> sadd set2 a b x y
4
> sinter set1 set2
b
a
> sunion set1 set2
x
a
b
d
y
c
> srem set1 a
1
> smembers set1
b
d
c
```

## 5、有序集合操作命令

Redis 有序集合是 string 类型元素的集合，且不允许有重复成员。每个元素都会关联一个 double 类型的分数。

常用命令：

- `ZADD key score1 member1 [score2 member2]`：向有序集合添加一个或多个成员
- `ZRANGE key start stop [WITHSCORES]`：通过索引区间返回有序集合中指定区间内的成员
- `ZINCRBY key increment member`：有序集合中对指定成员的分数加上增量 increment
- `ZREM key member [member ...]`：移除有序集合中的一个或多个成员

<img src="img/image15.png" alt="image15" style="zoom: 67%;" />

 ```shell
 > zadd zset1 10.0 a 10.5 b
 2
 > zadd zset1 10.2 c
 1
 > zrange zset1 0 -1
 a
 c
 b
 > zrange zset1 0 -1 withscores
 a
 10
 c
 10.199999999999999
 b
 10.5
 > zincrby zet1 5.0 a
 15
 > zrange zset1 0 -1 withscores
 c
 10.199999999999999
 b
 10.5
 a
 15
 > zrem zet1 b
 1
 ```

## 6、通用命令

Redis 的通用命令是不分数据类型的，都可以使用的命令：

- `KEYS pattern`：查找所有符合给定模式（pattern）的 key
- `EXISTS key`：检查给定 key 是否存在
- `TYPE key`：返回 key 所储存的值的类型
- `DEL key`：该命令用于在 key 存在时删除 key

```shell
> keys *
key1
set1
zet1
100
name
set2
mylist
> keys set*
set1
set2
> exists name
1
> exists abc
0
> type name
string
> type set1
set
> type mylist
list
> del name
1
> del 100
1
> del set1 set2 zset1
3
```

# 四、在 Java 中操作 Redis

## 1、Redis的Java客户端

前面我们讲解了 Redis 的常用命令，这些命令是我们操作 Redis 的基础，那么我们在 Java 程序中应该如何操作Redis 呢？这就需要使用 Redis 的 Java 客户端，就如同我们使用 JDBC 操作 MySQL 数据库一样。

Redis 的 Java 客户端很多，常用的几种：

- Jedis
- Lettuce
- Spring Data Redis

Spring 对 Redis 客户端进行了整合，提供了 Spring Data Redis，在 SpringBoot 项目中还提供了对应的 Starter，即 spring-boot-starter-data-redis。

接下来我们重点学习 **Spring Data Redis**。

## 2、Spring Data Redis 使用方式

### 2.1、介绍

Spring Data Redis 是 Spring 的一部分，提供了在 Spring 应用中通过简单的配置就可以访问 Redis 服务，对 Redis 底层开发包进行了高度封装。在 Spring 项目中，可以使用 Spring Data Redis 来简化 Redis 操作。

网址：https://spring.io/projects/spring-data-redis

<img src="img/image16.png" alt="image16" style="zoom: 33%;" /> 

Spring Boot 提供了对应的 Starter，maven坐标：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Spring Data Redis 中提供了一个高度封装的类：**RedisTemplate**，对相关 api 进行了归类封装，将同一类型操作封装为 operation 接口，具体分类如下：

- ValueOperations：string 数据操作
- SetOperations：set 类型数据操作
- ZSetOperations：zset 类型数据操作
- HashOperations：hash 类型的数据操作
- ListOperations：list 类型的数据操作

### 2.2、环境搭建

进入到 sky-server 模块

**1). 导入 Spring Data Redis 的 maven 坐标(已完成)**

```xml
<dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**2). 配置 Redis 数据源**

在 application-dev.yml 中添加

```yaml
sky:
  redis:
    host: localhost
    port: 6379
    password: 123456
    database: 10
```

**解释说明**：

database：指定使用 Redis 的哪个数据库，Redis 服务启动后默认有 16 个数据库，编号分别是从 0 到 15。

可以通过修改 Redis 配置文件来指定数据库的数量。

在 application.yml 中添加读取 application-dev.yml 中的相关 Redis 配置

```yaml
spring:
  profiles:
    active: dev
  redis:
    host: ${sky.redis.host}
    port: ${sky.redis.port}
    password: ${sky.redis.password}
    database: ${sky.redis.database}
```

**3). 编写配置类，创建 RedisTemplate 对象**

```java
package com.sky.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
@Slf4j
public class RedisConfiguration {
    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory redisConnectionFactory){
        log.info("开始创建redis模板对象...");
        RedisTemplate redisTemplate = new RedisTemplate();
        //设置redis的连接工厂对象
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        //设置redis key的序列化器
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        return redisTemplate;
    }
}
```

**解释说明**：

当前配置类不是必须的，因为 SpringBoot 框架会自动装配 RedisTemplate 对象，但是默认的 key 序列化器为JdkSerializationRedisSerializer，导致我们存到 Redis 中后的数据和原始数据有差别，故设置为 StringRedisSerializer序列化器。

**4). 通过 RedisTemplate 对象操作 Redis**

在 test 下新建测试类

```java
package com.sky.test;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.redis.core.*;

@SpringBootTest
public class SpringDataRedisTest {
    @Autowired
    private RedisTemplate redisTemplate;

    @Test
    public void testRedisTemplate(){
        System.out.println(redisTemplate);
        //string数据操作
        ValueOperations valueOperations = redisTemplate.opsForValue();
        //hash类型的数据操作
        HashOperations hashOperations = redisTemplate.opsForHash();
        //list类型的数据操作
        ListOperations listOperations = redisTemplate.opsForList();
        //set类型数据操作
        SetOperations setOperations = redisTemplate.opsForSet();
        //zset类型数据操作
        ZSetOperations zSetOperations = redisTemplate.opsForZSet();
    }
}
```

测试：

<img src="img/image17.png" alt="image17" style="zoom:50%;" /> 

说明 RedisTemplate 对象注入成功，并且通过该 RedisTemplate 对象获取操作 5 种数据类型相关对象。

上述环境搭建完毕后，接下来，我们就来具体对常见 5 种数据类型进行操作。

### 2.3、操作常见类型数据

**1). 操作字符串类型数据**

```java
/**
 * 操作字符串类型的数据
 */
@Test
public void testString(){
    // set get setex setnx
    ValueOperations valueOperations = redisTemplate.opsForValue();
    valueOperations.set("city","北京");
    String city = (String) valueOperations.get("city");
    System.out.println(city);
    valueOperations.set("code","1234",3, TimeUnit.MINUTES);
    valueOperations.setIfAbsent("lock","1");
    valueOperations.setIfAbsent("lock","2");
}
```

**2). 操作哈希类型数据**

```java
/**
 * 操作哈希类型的数据
 */
@Test
public void testHash(){
    //hset hget hdel hkeys hvals
    HashOperations hashOperations = redisTemplate.opsForHash();

    hashOperations.put("100","name","tom");
    hashOperations.put("100","age","20");

    String name = (String) hashOperations.get("100", "name");
    System.out.println(name);

    Set keys = hashOperations.keys("100");
    System.out.println(keys);

    List values = hashOperations.values("100");
    System.out.println(values);

    hashOperations.delete("100","age");
}
```

**3). 操作列表类型数据**

```java
/**
 * 操作列表类型的数据
 */
@Test
public void testList(){
    //lpush lrange rpop llen
    ListOperations listOperations = redisTemplate.opsForList();

    listOperations.leftPushAll("mylist","a","b","c");
    listOperations.leftPush("mylist","d");

    List mylist = listOperations.range("mylist", 0, -1);
    System.out.println(mylist);

    listOperations.rightPop("mylist");

    Long size = listOperations.size("mylist");
    System.out.println(size);
}
```

**4). 操作集合类型数据**

```java
/**
 * 操作集合类型的数据
 */
@Test
public void testSet(){
    //sadd smembers scard sinter sunion srem
    SetOperations setOperations = redisTemplate.opsForSet();

    setOperations.add("set1","a","b","c","d");
    setOperations.add("set2","a","b","x","y");

    Set members = setOperations.members("set1");
    System.out.println(members);

    Long size = setOperations.size("set1");
    System.out.println(size);

    Set intersect = setOperations.intersect("set1", "set2");
    System.out.println(intersect);

    Set union = setOperations.union("set1", "set2");
    System.out.println(union);

    setOperations.remove("set1","a","b");
}
```

**5). 操作有序集合类型数据**

```java
/**
 * 操作有序集合类型的数据
 */
@Test
public void testZset(){
    //zadd zrange zincrby zrem
    ZSetOperations zSetOperations = redisTemplate.opsForZSet();

    zSetOperations.add("zset1","a",10);
    zSetOperations.add("zset1","b",12);
    zSetOperations.add("zset1","c",9);

    Set zset1 = zSetOperations.range("zset1", 0, -1);
    System.out.println(zset1);

    zSetOperations.incrementScore("zset1","c",10);

    zSetOperations.remove("zset1","a","b");
}
```

**6). 通用命令操作**

```java
/**
 * 通用命令操作
 */
@Test
public void testCommon(){
    //keys exists type del
    Set keys = redisTemplate.keys("*");
    System.out.println(keys);

    Boolean name = redisTemplate.hasKey("name");
    System.out.println(name);
    Boolean set1 = redisTemplate.hasKey("set1");
    System.out.println(set1);

    for (Object key : keys) {
        DataType type = redisTemplate.type(key);
        System.out.println(type.name());
    }

    redisTemplate.delete("mylist");
}
```

# 五、店铺营业状态设置

## 1、需求分析和设计

### 1.1、产品原型

进到苍穹外卖后台，显示餐厅的营业状态，营业状态分为**营业中**和**打烊中**，若当前餐厅处于营业状态，自动接收任何订单，客户可在小程序进行下单操作；若当前餐厅处于打烊状态，不接受任何订单，客户便无法在小程序进行下单操作。

<img src="img/image18.png" alt="image18" style="zoom:50%;" /> 

点击**营业状态**按钮时，弹出更改营业状态

<img src="img/image19.png" alt="image19" style="zoom:50%;" /> 

选择营业，设置餐厅为**营业中**状态

选择打烊，设置餐厅为**打烊中**状态

**状态说明**：

<img src="img/image20.png" alt="image20" style="zoom: 67%;" /> 

### 1.2、接口设计

根据上述原型图设计接口，共包含 3 个接口。

**接口设计**：

- 设置营业状态
- 管理端查询营业状态
- 用户端查询营业状态

**注**：从技术层面分析，其实管理端和用户端查询营业状态时，可通过一个接口去实现即可。因为营业状态是一致的。但是，本项目约定：

- **管理端**发出的请求，统一使用 /admin 作为前缀。
- **用户端**发出的请求，统一使用 /user 作为前缀。

因为访问路径不一致，故分为两个接口实现。

**1). 设置营业状态**

<img src="img/image21.png" alt="image21" style="zoom:50%;" /> 

**2). 管理端营业状态**

<img src="img/image22.png" alt="image22" style="zoom:50%;" /> 

**3). 用户端营业状态**

<img src="img/image23.png" alt="image23" style="zoom:50%;" /> 

### 1.3、营业状态存储方式

虽然，可以通过一张表来存储营业状态数据，但整个表中只有一个字段，所以意义不大。

营业状态数据存储方式：基于 Redis 的字符串来进行存储

<img src="img/image24.png" alt="image24" style="zoom:50%;" /> 

**约定**：1 表示营业，0 表示打烊

## 2、代码开发

### 2.1、设置营业状态

在 sky-server 模块中，创建 ShopController.java

**根据接口定义创建 ShopController 的 setStatus 设置营业状态方法**：

```java
package com.sky.controller.admin;

import com.sky.result.Result;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController("adminShopController")
@RequestMapping("/admin/shop")
@Api(tags = "店铺相关接口")
@Slf4j
public class ShopController {
    public static final String KEY = "SHOP_STATUS";

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 设置店铺的营业状态
     * @param status
     * @return
     */
    @PutMapping("/{status}")
    @ApiOperation("设置店铺的营业状态")
    public Result setStatus(@PathVariable Integer status){
        log.info("设置店铺的营业状态为：{}",status == 1 ? "营业中" : "打烊中");
        redisTemplate.opsForValue().set(KEY,status);
        return Result.success();
    }
}
```

### 2.2、管理端查询营业状态

**根据接口定义创建ShopController的getStatus查询营业状态方法：**

```java
/**
 * 获取店铺的营业状态
 * @return
 */
@GetMapping("/status")
@ApiOperation("获取店铺的营业状态")
public Result<Integer> getStatus(){
    Integer status = (Integer) redisTemplate.opsForValue().get(KEY);
    log.info("获取到店铺的营业状态为：{}",status == 1 ? "营业中" : "打烊中");
    return Result.success(status);
}
```

### 2.3、用户端查询营业状态

创建 com.sky.controller.user 包，在该包下创建 ShopController.java

**根据接口定义创建 ShopController 的 getStatus 查询营业状态方法**：

```java
package com.sky.controller.user;

import com.sky.result.Result;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.bind.annotation.*;

@RestController("userShopController")
@RequestMapping("/user/shop")
@Api(tags = "店铺相关接口")
@Slf4j
public class ShopController {
    public static final String KEY = "SHOP_STATUS";

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 获取店铺的营业状态
     * @return
     */
    @GetMapping("/status")
    @ApiOperation("获取店铺的营业状态")
    public Result<Integer> getStatus(){
        Integer status = (Integer) redisTemplate.opsForValue().get(KEY);
        log.info("获取到店铺的营业状态为：{}",status == 1 ? "营业中" : "打烊中");
        return Result.success(status);
    }
}
```

## 3、功能测试

### 3.1、接口文档测试

**启动服务**：访问 http://localhost:8080/doc.html，打开店铺相关接口

**注意**：使用 admin 用户登录重新获取 token，防止 token 失效。

**设置营业状态**：

<img src="img/image25.png" alt="image25" style="zoom:50%;" /> 

点击发送

<img src="img/image26.png" alt="image26" style="zoom:50%;" /> 

查看 Idea 控制台日志

<img src="img/image27.png" alt="image27" style="zoom:50%;" /> 

查看 Redis 中数据

<img src="img/image28.png" alt="image28" style="zoom:50%;" /> 



**管理端查询营业状态**：

<img src="img/image29.png" alt="image29" style="zoom:50%;" /> 

**用户端查询营业状态**：

<img src="img/image30.png" alt="image30" style="zoom:50%;" /> 

### 3.2、接口分组展示

在上述接口文档测试中，管理端和用户端的接口放在一起，不方便区分。

<img src="img/image31.png" alt="image31" style="zoom:50%;" /> 

接下来，我们要实现管理端和用户端接口进行区分。

在 WebMvcConfiguration.java 中，分别扫描 "com.sky.controller.admin" 和 "com.sky.controller.user" 这两个包。

```java
@Bean
public Docket docket1(){
    log.info("准备生成接口文档...");
    ApiInfo apiInfo = new ApiInfoBuilder()
        .title("苍穹外卖项目接口文档")
        .version("2.0")
        .description("苍穹外卖项目接口文档")
        .build();

    Docket docket = new Docket(DocumentationType.SWAGGER_2)
        .groupName("管理端接口")
        .apiInfo(apiInfo)
        .select()
        //指定生成接口需要扫描的包
        .apis(RequestHandlerSelectors.basePackage("com.sky.controller.admin"))
        .paths(PathSelectors.any())
        .build();

    return docket;
}

@Bean
public Docket docket2(){
    log.info("准备生成接口文档...");
    ApiInfo apiInfo = new ApiInfoBuilder()
        .title("苍穹外卖项目接口文档")
        .version("2.0")
        .description("苍穹外卖项目接口文档")
        .build();

    Docket docket = new Docket(DocumentationType.SWAGGER_2)
        .groupName("用户端接口")
        .apiInfo(apiInfo)
        .select()
        //指定生成接口需要扫描的包
        .apis(RequestHandlerSelectors.basePackage("com.sky.controller.user"))
        .paths(PathSelectors.any())
        .build();

    return docket;
}
```

重启服务器，再次访问接口文档，可进行选择**用户端接口**或者**管理端接口**

<img src="img/image32.png" alt="image32" style="zoom:50%;" /> 

### 3.3、前后端联调测试

启动 nginx，访问 http://localhost

进入后台，状态为**营业中**

<img src="img/image33.png" alt="image33" style="zoom:50%;" /> 

点击**营业状态设置**，修改状态为**打烊中**

<img src="img/image34.png" alt="image34" style="zoom:50%;" /> 

再次查看状态，状态已为**打烊中**

<img src="img/image35.png" alt="image35" style="zoom:50%;" /> 

## 4、代码提交

**点击提交**：

<img src="img/image36.png" alt="image36" style="zoom:50%;" /> 

**提交过程中**，出现提示：

<img src="img/image37.png" alt="image37" style="zoom:50%;" />  

**继续 push**：

<img src="img/image38.png" alt="image38" style="zoom:50%;" /> 

**推送成功**：

<img src="img/image39.png" alt="image" style="zoom: 67%;" /> 

