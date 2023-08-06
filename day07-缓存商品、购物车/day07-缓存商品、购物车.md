

# 今日内容

- 缓存菜品
- 缓存套餐
- 添加购物车
- 查看购物车
- 清空购物车

功能实现：**缓存商品**、**购物车**

**效果图**：

<img src="img/image1.png" alt="image1" style="zoom:50%;" /> <img src="img/image2.png" alt="image2" style="zoom:50%;" /><img src="img/image3.png" alt="image3" style="zoom:50%;" />

# 一、缓存菜品

## 1、问题说明

用户端小程序展示的菜品数据都是通过查询数据库获得，如果用户端访问量比较大，数据库访问压力随之增大。

<img src="img/image4.png" alt="image4" style="zoom:80%;" /> 

**结果**：系统响应慢、用户体验差

## 2、实现思路

通过 Redis 来缓存菜品数据，减少数据库查询操作。

<img src="img/image5.png" alt="image5" style="zoom:80%;" /> 

**缓存逻辑分析**：

- 每个分类下的菜品保存一份缓存数据
- 数据库中菜品数据有变更时清理缓存数据

<img src="img/image6.png" alt="image6" style="zoom:67%;" /> 

## 3、代码开发

**修改用户端接口 DishController 的 list 方法，加入缓存处理逻辑**：

```java
@Autowired
private RedisTemplate redisTemplate;

/**
 * 根据分类id查询菜品
 * @param categoryId
 * @return
 */
@GetMapping("/list")
@ApiOperation("根据分类id查询菜品")
public Result<List<DishVO>> list(Long categoryId) {
    log.info("根据分类id查询菜品：{}",categoryId);

    //构造redis中的key，规则：dish_分类id
    String key = "dish_" + categoryId;

    //查询redis中是否存在菜品数据
    List<DishVO> list = (List<DishVO>) redisTemplate.opsForValue().get(key);
    if(list != null && list.size() > 0){
        //如果存在，直接返回，无须查询数据库
        return Result.success(list);
    }

    //如果不存在，查询数据库，将查询到的数据放入redis中
    Dish dish = new Dish();
    dish.setCategoryId(categoryId);
    dish.setStatus(StatusConstant.ENABLE);//查询起售中的菜品

    list = dishService.listWithFlavor(dish);
    redisTemplate.opsForValue().set(key, list);

    return Result.success(list);
}
```

为了保证**数据库**和 **Redis** 中的数据保持一致，修改**管理端接口 DishController** 的相关方法，加入清理缓存逻辑。

需要改造的方法：

- 新增菜品
- 修改菜品
- 批量删除菜品
- 起售、停售菜品

**抽取清理缓存的方法**：

在管理端 DishController 中添加

```java
@Autowired
private RedisTemplate redisTemplate;

/**
 * 清理缓存数据
 * @param pattern
 */
private void cleanCache(String pattern){
    Set keys = redisTemplate.keys(pattern);
    redisTemplate.delete(keys);
}
```

**调用清理缓存的方法，保证数据一致性**：

**1). 新增菜品优化**

```java
/**
 * 新增菜品
 * @param dishDTO
 * @return
 */
@PostMapping
@ApiOperation("新增菜品")
public Result save(@RequestBody DishDTO dishDTO) {
    log.info("新增菜品：{}", dishDTO);
    dishService.saveWithFlavor(dishDTO);

    //清理缓存数据
    String key = "dish_" + dishDTO.getCategoryId();
    cleanCache(key);
    
    return Result.success();
}
```

**2). 菜品批量删除优化**

```java
/**
 * 菜品批量删除
 * @param ids
 * @return
 */
@DeleteMapping
@ApiOperation("菜品批量删除")
public Result delete(@RequestParam List<Long> ids) {
    log.info("菜品批量删除：{}", ids);
    dishService.deleteBatch(ids);

    //将所有的菜品缓存数据清理掉，所有以dish_开头的key
    cleanCache("dish_*");

    return Result.success();
}
```

**3). 修改菜品优化**

```java
/**
 * 修改菜品
 * @param dishDTO
 * @return
 */
@PutMapping
@ApiOperation("修改菜品")
public Result update(@RequestBody DishDTO dishDTO) {
    log.info("修改菜品：{}", dishDTO);
    dishService.updateWithFlavor(dishDTO);

    //将所有的菜品缓存数据清理掉，所有以dish_开头的key
    cleanCache("dish_*");

    return Result.success();
}
```

**4). 菜品起售停售优化**

```java
/**
 * 菜品起售停售
 * @param status
 * @param id
 * @return
 */
@PostMapping("/status/{status}")
@ApiOperation("菜品起售停售")
public Result<String> startOrStop(@PathVariable Integer status, Long id) {
    log.info("菜品起售停售：{}",id);
    dishService.startOrStop(status, id);

    //将所有的菜品缓存数据清理掉，所有以dish_开头的key
    cleanCache("dish_*");

    return Result.success();
}
```

## 4、功能测试

可以通过如下方式进行测试：

- 查看控制台 sql
- 前后端联调
- 查看 Redis 中的缓存数据

以**加入缓存**、**菜品修改**两个功能测试为例，通过前后端联调方式，查看控制台 sql 的打印和 Redis 中的缓存数据变化。

**1). 加入缓存**

当第一次查询某个分类的菜品时，会从数据库中进行查询，同时将查询的结果存储到 Redis 中，在后续的访问中，若查询相同分类的菜品时，直接从 Redis 缓存中查询，不再查询数据库。

**登录小程序**：选择蜀味牛蛙（id=17）

<img src="img/image7.png" alt="image7" style="zoom:50%;" /> 

**查看控制台 sql**：有查询语句，说明是从数据库中进行查询

<img src="img/image8.png" alt="image8" style="zoom:50%;" /> 

**查看 Redis 中的缓存数据**：说明缓存成功

<img src="img/image9.png" alt="image9" style="zoom:50%;" /> 

**再次访问**：选择蜀味牛蛙（id=17）

<img src="img/image10.png" alt="image10" style="zoom:50%;" /> 

说明是从 Redis 中查询的数据。

**2). 菜品修改**

当在后台修改菜品数据时，为了保证 Redis 缓存中的数据和数据库中的数据时刻保持一致，当修改后，需要清空对应的缓存数据。用户再次访问时，还是先从数据库中查询，同时再把查询的结果存储到 Redis 中，这样就能保证缓存和数据库的数据保持一致。

**进入后台**：修改蜀味牛蛙分类下的任意一个菜品，当前分类的菜品数据已在 Redis 中缓存

<img src="img/image11.png" alt="image11" style="zoom:50%;" /> 

**修改**：

<img src="img/image12.png" alt="image12" style="zoom:50%;" /> 

**查看 Redis 中的缓存数据**：说明修改时，已清空缓存

<img src="img/image13.png" alt="image13" style="zoom:50%;" /> 

用户再次访问同一个菜品分类时，需要先查询数据库，再把结果同步到 Redis 中，保证了两者数据一致性。

其它功能测试步骤基本一致，自已测试即可。

## 5、代码提交

<img src="img/image14.png" alt="image14" style="zoom:50%;" /> 

后续步骤和其它功能代码提交一致，不再赘述。

# 二、缓存套餐

## 1、Spring Cache

### 1.1、介绍

Spring Cache 是一个框架，实现了基于注解的缓存功能，只需要简单地加一个注解，就能实现缓存功能。

Spring Cache 提供了一层抽象，底层可以切换不同的缓存实现，例如：

- EHCache
- Caffeine
- Redis（常用）

**起步依赖**：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-cache</artifactId>  		            		       	 <version>2.7.3</version> 
</dependency>
```

### 1.2、常用注解

在 SpringCache中 提供了很多缓存操作的注解，以下是常见的几个：

| **注解**       | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| @EnableCaching | 开启缓存注解功能，通常加在启动类上                           |
| @Cacheable     | 在方法执行前先查询缓存中是否有数据，如果有数据，则直接返回缓存数据；如果没有缓存数据，调用方法并将方法返回值放到缓存中 |
| @CachePut      | 将方法的返回值放到缓存中                                     |
| @CacheEvict    | 将一条或多条数据从缓存中删除                                 |

在 Spring Boot 项目中，使用缓存技术只需在项目中导入相关缓存技术的依赖包，并在启动类上使用@EnableCaching 开启缓存支持即可。

例如，使用 Redis 作为缓存技术，只需要导入 Spring data Redis 的 maven 坐标即可。

### 1.3、入门案例

**1). 环境准备**

**导入基础工程**：底层已使用 Redis 缓存实现

基础环境的代码，在今天的资料中已经准备好了， 只需要将这个工程导入进来就可以了。导入进来的工程结构如下： 

<img src="img/image15.png" alt="image15" style="zoom:50%;" /> 

**数据库准备**：

创建名为 spring_cache_demo 数据库，将 springcachedemo.sql 脚本直接导入数据库中。

<img src="img/image16.png" alt="image16" style="zoom:80%;" /> 

**引导类上加 @EnableCaching**：

```java
package com.itheima;

import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@Slf4j
@SpringBootApplication
@EnableCaching//开启缓存注解功能
public class CacheDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(CacheDemoApplication.class,args);
        log.info("项目启动成功...");
    }
}
```



**2). @CachePut 注解**

**@CachePut 说明** ：

* 作用：将方法返回值，放入缓存
* value：缓存的名称，每个缓存名称下面可以有很多 key
* key：缓存的 key  ----------> 支持 Spring 的表达式语言 SPEL 语法

**在 save 方法上加注解 @CachePut**

当前 UserController 的 save 方法是用来保存用户信息的，我们希望在该用户信息保存到数据库的同时，也往缓存中缓存一份数据，我们可以在 save 方法上加上注解 @CachePut，用法如下：

```java
/**
 * CachePut：将方法返回值放入缓存
 * value：缓存的名称，每个缓存名称下面可以有多个key
 * key：缓存的key
 */
@PostMapping
@CachePut(value = "userCache", key = "#user.id")//key的生成：userCache::1
public User save(@RequestBody User user){
    userMapper.insert(user);
    return user;
}
```

**说明**：key 的写法如下

1. `#user.id`：#user 指的是方法形参的名称，id 指的是 user 的 id 属性，也就是使用 user 的 id 属性作为 key

2. `#result.id`：#result 代表方法返回值，该表达式代表以返回对象的 id 属性作为 key ；

3. `#p0.id`：#p0 指的是方法中的第一个参数，id 指的是第一个参数的 id 属性，也就是使用第一个参数的 id 属性作为 key ；

4. `#a0.id`：#a0 指的是方法中的第一个参数，id 指的是第一个参数的 id 属性，也就是使用第一个参数的 id 属性作为 key ；

5. `#root.args[0].id`：#root.args[0] 指的是方法中的第一个参数，id 指的是第一个参数的 id 属性，也就是使用第一个参数的 id 属性作为 key ；

**启动服务，通过 swagger 接口文档测试，访问 UserController 的 save() 方法**

因为 id 是自增，所以不需要设置 id 属性

 <img src="img/image17.png" alt="image17" style="zoom:50%;" /> 

**查看 user 表中的数据**

<img src="img/image18.png" alt="image18" style="zoom: 67%;" /> 

**查看 Redis 中的数据**

<img src="img/image19.png" alt="image19" style="zoom:50%;" /> 



**3). @Cacheable 注解**

**@Cacheable 说明**：

* 作用：在方法执行前，spring 先查看缓存中是否有数据，如果有数据，则直接返回缓存数据；若没有数据，调用方法并将方法返回值放到缓存中

* value：缓存的名称，每个缓存名称下面可以有多个 key

* key：缓存的 key  ----------> 支持 Spring 的表达式语言 SPEL 语法

 **在 getById 上加注解 @Cacheable**

```java
/**
 * Cacheable：在方法执行前spring先查看缓存中是否有数据，如果有数据，则直接返回缓存数据；若没有数据，	  *调用方法并将方法返回值放到缓存中
 * value：缓存的名称，每个缓存名称下面可以有多个key
 * key：缓存的key
 */
@GetMapping
@Cacheable(cacheNames = "userCache",key="#id")
public User getById(Long id){
    User user = userMapper.getById(id);
    return user;
}
```

**重启服务，通过 swagger 接口文档测试，访问 UserController 的 getById() 方法**

第一次访问，会请求我们 controller 的方法，查询数据库。后面再查询相同的 id，就直接从 Redis 中查询数据，不用再查询数据库了，就说明缓存生效了。

提前在 redis 中手动删除掉 id=1 的用户数据

<img src="img/image20.png" alt="image20" style="zoom:50%;" /> 

**查看控制台 sql 语句**：说明从数据库查询的用户数据

<img src="img/image21.png" alt="image21" style="zoom: 67%;" /> 

**查看 Redis 中的缓存数据**：说明已成功缓存

<img src="img/image22.png" alt="image22" style="zoom:50%;" /> 

再次查询相同 id 的数据时，直接从 Redis 中直接获取，不再查询数据库。



**4). @CacheEvict注解**

**@CacheEvict 说明** ：

* 作用：清理指定缓存

* value：缓存的名称，每个缓存名称下面可以有多个 key

* key：缓存的 key  ----------> 支持 Spring 的表达式语言 SPEL 语法

**在 delete 方法上加注解 @CacheEvict**

```java
@DeleteMapping
@CacheEvict(cacheNames = "userCache",key = "#id")//删除某个key对应的缓存数据
public void deleteById(Long id){
    userMapper.deleteById(id);
}

@DeleteMapping("/delAll")
@CacheEvict(cacheNames = "userCache",allEntries = true)//删除userCache下所有的缓存数据
public void deleteAll(){
    userMapper.deleteAll();
}
```

**重启服务，通过 swagger 接口文档测试，访问 UserController 的 deleteAll() 方法**

<img src="img/image23.png" alt="image23" style="zoom:50%;" /> 

**查看 user 表**：数据清空

<img src="img/image24.png" alt="image24" style="zoom:80%;" /> 

**查询 Redis 缓存数据**

<img src="img/image25.png" alt="image25" style="zoom:50%;" /> 



## 2、实现思路

**实现步骤**：

1). 导入 Spring Cache 和 Redis 相关 maven 坐标

2). 在启动类上加入 @EnableCaching 注解，开启缓存注解功能

3). 在用户端接口 SetmealController 的 list 方法上加入 @Cacheable注解

4). 在管理端接口 SetmealController 的 save、delete、update、startOrStop 等方法上加入 CacheEvict 注解

## 3、代码开发

按照上述实现步骤：

**1). 导入 Spring Cache 和 Redis 相关 maven 坐标（已实现）**

```xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

**2). 在启动类上加入 @EnableCaching 注解，开启缓存注解功能**

```java
package com.sky;

import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@SpringBootApplication
@EnableTransactionManagement //开启注解方式的事务管理
@Slf4j
@EnableCaching//开启缓存注解功能
public class SkyApplication {
    public static void main(String[] args) {
        SpringApplication.run(SkyApplication.class, args);
        log.info("server started");
    }
}
```

**3). 在用户端接口 SetmealController 的 list 方法上加入 @Cacheable 注解**

```java
/**
 * 条件查询
 * @param categoryId
 * @return
 */
@GetMapping("/list")
@ApiOperation("根据分类id查询套餐")
@Cacheable(cacheNames = "setmealCache",key = "#categoryId") //key: setmealCache::100
public Result<List<Setmeal>> list(Long categoryId) {
    Setmeal setmeal = new Setmeal();
    setmeal.setCategoryId(categoryId);
    setmeal.setStatus(StatusConstant.ENABLE);

    List<Setmeal> list = setmealService.list(setmeal);
    return Result.success(list);
}
```

**4). 在管理端接口 SetmealController 的 save、delete、update、startOrStop 等方法上加入 @CacheEvict 注解**

```java
/**
 * 新增套餐
 * @param setmealDTO
 * @return
 */
@PostMapping
@ApiOperation("新增套餐")
@CacheEvict(cacheNames = "setmealCache",key = "#setmealDTO.categoryId")//key: setmealCache::100
public Result save(@RequestBody SetmealDTO setmealDTO) {
    setmealService.saveWithDish(setmealDTO);
    return Result.success();
}

/**
 * 批量删除套餐
 * @param ids
 * @return
 */
@DeleteMapping
@ApiOperation("批量删除套餐")
@CacheEvict(cacheNames = "setmealCache",allEntries = true)
public Result delete(@RequestParam List<Long> ids) {
    setmealService.deleteBatch(ids);
    return Result.success();
}

/**
 * 修改套餐
 * @param setmealDTO
 * @return
 */
@PutMapping
@ApiOperation("修改套餐")
@CacheEvict(cacheNames = "setmealCache",allEntries = true)
public Result update(@RequestBody SetmealDTO setmealDTO) {
    setmealService.update(setmealDTO);
    return Result.success();
}

/**
 * 套餐起售停售
 * @param status
 * @param id
 * @return
 */
@PostMapping("/status/{status}")
@ApiOperation("套餐起售停售")
@CacheEvict(cacheNames = "setmealCache",allEntries = true)
public Result startOrStop(@PathVariable Integer status, Long id) {
    setmealService.startOrStop(status, id);
    return Result.success();
}
```

## 4、功能测试

通过前后端联调方式来进行测试，同时观察 redis 中缓存的套餐数据。和**缓存菜品**功能测试基本一致，不再赘述。

## 5、代码提交

<img src="img/image26.png" alt="image26" style="zoom:50%;" /> 

后续步骤和其它功能代码提交一致，不再赘述。

# 三、添加购物车

## 1、需求分析和设计

### 1.1、产品原型

用户可以将菜品或者套餐添加到购物车。对于菜品来说，如果设置了口味信息，则需要选择规格后才能加入购物车；对于套餐来说，可以直接点击 <img src="img/image27.png" alt="image27" style="zoom: 67%;" /> 将当前套餐加入购物车。在购物车中可以修改菜品和套餐的数量，也可以清空购物车。

**效果图**：

 <img src="img/image28.png" alt="image28" style="zoom:50%;" /> 

### 1.2、接口设计

通过上述原型图，设计出对应的添加购物车接口。

<img src="img/image29.png" alt="image29" style="zoom:50%;" /> <img src="img/image30.png" alt="image30" style="zoom:50%;" />

**说明**：添加购物车时，有可能添加菜品，也有可能添加套餐。故传入参数要么是菜品 id，要么是套餐 id。

### 1.3、表设计

用户的购物车数据，也是需要保存在数据库中的，购物车对应的数据表为 shopping_cart 表，具体表结构如下：

| **字段名**  | **数据类型**  | **说明**     | **备注** |
| ----------- | ------------- | ------------ | -------- |
| id          | bigint        | 主键         | 自增     |
| name        | varchar(32)   | 商品名称     | 冗余字段 |
| image       | varchar(255)  | 商品图片路径 | 冗余字段 |
| user_id     | bigint        | 用户 id      | 逻辑外键 |
| dish_id     | bigint        | 菜品 id      | 逻辑外键 |
| setmeal_id  | bigint        | 套餐 id      | 逻辑外键 |
| dish_flavor | varchar(50)   | 菜品口味     |          |
| number      | int           | 商品数量     |          |
| amount      | decimal(10,2) | 商品单价     | 冗余字段 |
| create_time | datetime      | 创建时间     |          |

**说明**： 

- 购物车数据是关联用户的，在表结构中，我们需要记录，每一个用户的购物车数据是哪些
- 菜品列表展示出来的既有套餐，又有菜品，如果用户选择的是套餐，就保存套餐 ID（setmeal_id），如果用户选择的是菜品，就保存菜品 ID（dish_id）
- 对同一个菜品 / 套餐，如果选择多份不需要添加多条记录，增加数量 number 即可

## 2、代码开发

### 2.1、DTO 设计

**根据添加购物车接口的参数设计 DTO**：

<img src="img/image31.png" alt="image31" style="zoom:50%;" /> 

在 sky-pojo 模块，ShoppingCartDTO.java 已定义

```java
package com.sky.dto;

import lombok.Data;
import java.io.Serializable;

@Data
public class ShoppingCartDTO implements Serializable {
    private Long dishId;
    private Long setmealId;
    private String dishFlavor;
}
```

### 2.2、Controller 层

**根据添加购物车接口创建 ShoppingCartController**：

```java
package com.sky.controller.user;

import com.sky.dto.ShoppingCartDTO;
import com.sky.result.Result;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * 购物车
 */
@RestController
@RequestMapping("/user/shoppingCart")
@Slf4j
@Api(tags = "C端-购物车相关接口")
public class ShoppingCartController {
    @Autowired
    private ShoppingCartService shoppingCartService;

    /**
     * 添加购物车
     * @param shoppingCartDTO
     * @return
     */
    @PostMapping("/add")
    @ApiOperation("添加购物车")
    public Result add(@RequestBody ShoppingCartDTO shoppingCartDTO){
        log.info("添加购物车，商品信息为：{}", shoppingCartDTO);
        shoppingCartService.addShoppingCart(shoppingCartDTO);//后续步骤实现
        return Result.success();
    }
}
```

### 2.3、Service 层接口

**创建 ShoppingCartService 接口**：

```java
package com.sky.service;

import com.sky.dto.ShoppingCartDTO;

public interface ShoppingCartService {
    /**
     * 添加购物车
     * @param shoppingCartDTO
     */
    void addShoppingCart(ShoppingCartDTO shoppingCartDTO);
}
```

### 2.4、Service 层实现类

**创建 ShoppingCartServiceImpl 实现类，并实现 add 方法**：

```java
package com.sky.service.impl;

import com.sky.context.BaseContext;
import com.sky.dto.ShoppingCartDTO;
import com.sky.entity.Dish;
import com.sky.entity.Setmeal;
import com.sky.entity.ShoppingCart;
import com.sky.mapper.DishMapper;
import com.sky.mapper.SetmealMapper;
import com.sky.service.ShoppingCartService;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.List;

@Service
@Slf4j
public class ShoppingCartServiceImpl implements ShoppingCartService {
    @Autowired
    private ShoppingCartMapper shoppingCartMapper;
    @Autowired
    private DishMapper dishMapper;
    @Autowired
    private SetmealMapper setmealMapper;
    
    /**
     * 添加购物车
     * @param shoppingCartDTO
     */
    public void addShoppingCart(ShoppingCartDTO shoppingCartDTO) {
        ShoppingCart shoppingCart = new ShoppingCart();
        BeanUtils.copyProperties(shoppingCartDTO, shoppingCart);
        //只能查询自己的购物车数据
        Long userId = BaseContext.getCurrentId();
        shoppingCart.setUserId(userId);

        //判断当前商品是否在购物车中
        List<ShoppingCart> shoppingCartList = shoppingCartMapper.list(shoppingCart);

        if (shoppingCartList != null && shoppingCartList.size() > 0) {
            //如果已经存在，就更新数量，数量加1
            shoppingCart = shoppingCartList.get(0);
            shoppingCart.setNumber(shoppingCart.getNumber() + 1);
            shoppingCartMapper.updateNumberById(shoppingCart);
        } else {
            //如果不存在，插入一条购物车数据，数量就是1

            //判断当前添加到购物车的是菜品还是套餐
            Long dishId = shoppingCartDTO.getDishId();
            if (dishId != null) {
                //本次添加到购物车的是菜品
                Dish dish = dishMapper.getById(dishId);
                shoppingCart.setName(dish.getName());
                shoppingCart.setImage(dish.getImage());
                shoppingCart.setAmount(dish.getPrice());
            } else {
                //本次添加到购物车的是套餐
                Setmeal setmeal = setmealMapper.getById(shoppingCartDTO.getSetmealId());
                shoppingCart.setName(setmeal.getName());
                shoppingCart.setImage(setmeal.getImage());
                shoppingCart.setAmount(setmeal.getPrice());
            }
            shoppingCart.setNumber(1);
            shoppingCart.setCreateTime(LocalDateTime.now());
            shoppingCartMapper.insert(shoppingCart);
        }
    }
}
```

### 2.5、Mapper 层

**创建 ShoppingCartMapper 接口**：

```java
package com.sky.mapper;

import com.sky.entity.ShoppingCart;
import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Update;
import java.util.List;

@Mapper
public interface ShoppingCartMapper {
    /**
     * 动态条件查询
     * @param shoppingCart
     * @return
     */
    List<ShoppingCart> list(ShoppingCart shoppingCart);

    /**
     * 根据id修改商品数量
     * @param shoppingCart
     */
    @Update("update shopping_cart set number = #{number} where id = #{id}")
    void updateNumberById(ShoppingCart shoppingCart);

    /**
     * 插入购物车数据
     * @param shoppingCart
     */
    @Insert("insert into shopping_cart (name, image, user_id, dish_id, setmeal_id, dish_flavor, number, amount, create_time) " +
            "values (#{name}, #{image}, #{userId}, #{dishId}, #{setmealId}, #{dishFlavor}, #{number}, #{amount}, #{createTime})")
    void insert(ShoppingCart shoppingCart);
}
```

**创建 ShoppingCartMapper.xml**：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
		"http://mybatis.org/dtd/mybatis-3-mapper.dtd" >

<mapper namespace="com.sky.mapper.ShoppingCartMapper">
    <select id="list" parameterType="ShoppingCart" resultType="ShoppingCart">
        select * from shopping_cart
        <where>
            <if test="userId != null">
                and user_id = #{userId}
            </if>
            <if test="dishId != null">
                and dish_id = #{dishId}
            </if>
            <if test="setmealId != null">
                and setmeal_id = #{setmealId}
            </if>
            <if test="dishFlavor != null">
                and dish_flavor = #{dishFlavor}
            </if>
        </where>
        order by create_time desc
    </select>
</mapper>
```

## 3、功能测试

进入小程序，添加菜品

<img src="img/image32.png" alt="image32" style="zoom:50%;" /> <img src="img/image33.png" alt="image33" style="zoom:50%;" />

加入购物车，查询数据库

<img src="img/image34.png" alt="image34" style="zoom:80%;" /> 

因为现在没有实现查看购物车功能，所以只能在表中进行查看。

在前后联调时，后台可通断点方式启动，查看运行的每一步。

## 4、代码提交

<img src="img/image35.png" alt="image35" style="zoom:50%;" /> 

后续步骤和其它功能代码提交一致，不再赘述。

# 四、查看购物车

## 1、需求分析和设计

### 1.1、产品原型

当用户添加完菜品和套餐后，可进入到购物车中，查看购物中的菜品和套餐。

<img src="img/image36.png" alt="image36" style="zoom:50%;" /> 

### 1.2、接口设计

<img src="img/image37.png" alt="image37" style="zoom:50%;" /> <img src="img/image38.png" alt="image38" style="zoom:50%;" />

## 2、代码开发

### 2.1、Controller 层

**在 ShoppingCartController 中创建查看购物车的方法**：

```java
/**
 * 查看购物车
 * @return
 */
@GetMapping("/list")
@ApiOperation("查看购物车")
public Result<List<ShoppingCart>> list(){
    log.info("查看购物车");
    List<ShoppingCart> list = shoppingCartService.showShoppingCart();
    return Result.success(list);
}
```

### 2.2、Service 层接口

**在 ShoppingCartService 接口中声明查看购物车的方法**：

```java
/**
 * 查看购物车
 * @return
 */
List<ShoppingCart> showShoppingCart();
```

### 2.3、Service 层实现类

**在 ShoppingCartServiceImpl 中实现查看购物车的方法**：

```java
/**
 * 查看购物车
 * @return
 */
public List<ShoppingCart> showShoppingCart() {
    //获取当前微信用户的id
    Long userId = BaseContext.getCurrentId();
    ShoppingCart shoppingCart = ShoppingCart.builder()
        .userId(userId)
        .build();
    List<ShoppingCart> list = shoppingCartMapper.list(shoppingCart);
    return list;
}
```

## 3、功能测试

当进入小程序时，就会发起查看购物车的请求

<img src="img/image39.png" alt="image39" style="zoom:50%;" /> 

点击购物车图标

<img src="img/image40.png" alt="image40" style="zoom:50%;" /> 

测试成功。

## 4、代码提交

<img src="img/image41.png" alt="image41" style="zoom:50%;" /> 

后续步骤和其它功能代码提交一致，不再赘述。

# 五、清空购物车

## 1、需求分析和设计

### 1.1、产品原型

当点击清空按钮时，会把购物车中的数据全部清空。

<img src="img/image42.png" alt="image42" style="zoom:50%;" /> 

### 1.2、接口设计

<img src="img/image43.png" alt="image43" style="zoom:50%;" /> 

## 2、代码开发

### 2.1、Controller 层

**在 ShoppingCartController 中创建清空购物车的方法**：

```java
/**
 * 清空购物车商品
 * @return
 */
@DeleteMapping("/clean")
@ApiOperation("清空购物车商品")
public Result<String> clean(){
    log.info("清空购物车");
    shoppingCartService.cleanShoppingCart();
    return Result.success();
}
```

### 2.2、Service 层接口

**在 ShoppingCartService 接口中声明清空购物车的方法**：

```java
/**
 * 清空购物车商品
 */
void cleanShoppingCart();
```

### 2.3、Service 层实现类

**在 ShoppingCartServiceImpl 中实现清空购物车的方法**：

```java
/**
 * 清空购物车商品
 */
public void cleanShoppingCart() {
    //获取当前微信用户的id
    Long userId = BaseContext.getCurrentId();
    shoppingCartMapper.deleteByUserId(userId);
}
```

### 2.4、Mapper 层

**在 ShoppingCartMapper 接口中创建删除购物车数据的方法**：

```java
/**
 * 根据用户id删除购物车数据
 * @param userId
 */
@Delete("delete from shopping_cart where user_id = #{userId}")
void deleteByUserId(Long userId);
```

## 3、功能测试

进入到购物车页面

<img src="img/image44.png" alt="image44" style="zoom:50%;" /> 

点击清空

<img src="img/image45.png" alt="image45" style="zoom:50%;" /> 

查看数据库中的数据

<img src="img/image46.png" alt="image46" style="zoom:80%;" /> 

说明当前用户的购物车数据已全部删除。

## 4、代码提交

<img src="img/image47.png" alt="image47" style="zoom:50%;" /> 

后续步骤和其它功能代码提交一致，不再赘述。

# 六、删除购物车中一个商品

## 1、产品原型

![image48](img/image48.png)

## 2、接口设计

![image49](img/image49.png)

## 3、数据模型

shopping_cart 表：

![image50](img/image50.png)

## 4、代码开发

### 4.1、ShoppingCartController

~~~java
/**
 * 删除购物车中一个商品
 * @param shoppingCartDTO
 * @return
*/
@PostMapping("/sub")
@ApiOperation("删除购物车中一个商品")
public Result sub(@RequestBody ShoppingCartDTO shoppingCartDTO){
    log.info("删除购物车中一个商品，商品：{}", shoppingCartDTO);
    shoppingCartService.subShoppingCart(shoppingCartDTO);
    return Result.success();
}
~~~

### 4.2、ShoppingCartService

~~~java
/**
 * 删除购物车中一个商品
 * @param shoppingCartDTO
*/
void subShoppingCart(ShoppingCartDTO shoppingCartDTO);
~~~

### 4.3、ShoppingCartServiceImpl

~~~java
/**
 * 删除购物车中一个商品
 * @param shoppingCartDTO
*/
public void subShoppingCart(ShoppingCartDTO shoppingCartDTO) {
    ShoppingCart shoppingCart = new ShoppingCart();
    BeanUtils.copyProperties(shoppingCartDTO,shoppingCart);
    //设置查询条件，查询当前登录用户的购物车数据
    shoppingCart.setUserId(BaseContext.getCurrentId());

    List<ShoppingCart> list = shoppingCartMapper.list(shoppingCart);

    if(list != null && list.size() > 0){
        shoppingCart = list.get(0);

        Integer number = shoppingCart.getNumber();
        if(number == 1){
            //当前商品在购物车中的份数为1，直接删除当前记录
            shoppingCartMapper.deleteById(shoppingCart.getId());
        }else {
            //当前商品在购物车中的份数不为1，修改份数即可
            shoppingCart.setNumber(shoppingCart.getNumber() - 1);
            shoppingCartMapper.updateNumberById(shoppingCart);
        }
    }
}
~~~

### 4.4、ShoppingCartMapper

~~~java
/**
 * 根据id删除购物车数据
 * @param id
*/
@Delete("delete from shopping_cart where id = #{id}")
void deleteById(Long id);
~~~