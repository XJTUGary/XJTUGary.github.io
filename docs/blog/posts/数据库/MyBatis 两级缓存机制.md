---
date: 2025-09-23
title: MyBatis 两级缓存机制
categories:
  - 数据库
draft: false
comments: true
---
在数据库应用开发中，性能始终是我们关注的核心焦点之一。频繁地与数据库进行交互，执行重复的 SQL 查询，会消耗大量的系统资源，成为应用性能的主要瓶颈。想象一下，在一个高并发的场景下，每秒有成千上万的用户请求同一个热门商品的信息，如果每次请求都去查询数据库，数据库的压力将不堪重负。

为了解决这个问题，“缓存”技术应运而生。它像是一个高速的数据中转站，将频繁访问的数据暂存在内存中。当后续请求需要相同的数据时，无需再访问慢速的磁盘数据库，直接从内存中获取，从而极大地提升了响应速度。

作为一款优秀的持久层框架，MyBatis 自然也提供了一套强大且灵活的缓存机制。理解和正确使用 MyBatis 缓存，是优化 MyBatis 性能的关键一步。 这篇博客将带你深入浅出地了解 MyBatis 的两级缓存：一级缓存和二级缓存，帮助你更好地利用它们来提升应用性能。
<!-- more -->


## 一、MyBatis缓存概览：两级缓存结构

MyBatis提供了一套**两级缓存**体系，这是理解其缓存机制的核心：

```
应用层面
    ↑
┌─────────────────────────────────────────┐
│          二级缓存 (Mapper级别)            │ ← 多个SqlSession共享
└─────────────────────────────────────────┘
    ↑
┌─────────────────────────────────────────┐
│          一级缓存 (SqlSession级别)        │ ← 单个SqlSession内部
└─────────────────────────────────────────┘
    ↑
┌─────────────────────────────────────────┐
│              数据库 (持久层)               │
└─────────────────────────────────────────┘
```

---

## 二、一级缓存（本地缓存）深度解析

### 1. 什么是⼀级缓存？
- **作用域**：`SqlSession` 级别（同一个数据库会话）
- **生命周期**：与`SqlSession`共存亡
- **默认状态**：**默认开启**，无法关闭

### 2. ⼯作原理与验证

```java
// 验证一级缓存的示例
@Test
public void testFirstLevelCache() {
    SqlSession sqlSession = sqlSessionFactory.openSession();
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    
    try {
        System.out.println("=== 第一次查询 ===");
        User user1 = mapper.selectUserById(1L); // 发送SQL到数据库
        System.out.println("查询结果: " + user1);
        
        System.out.println("=== 第二次查询（相同参数）===");
        User user2 = mapper.selectUserById(1L); // 从一级缓存获取，不发送SQL
        System.out.println("查询结果: " + user2);
        
        System.out.println("是否是同一个对象: " + (user1 == user2)); // true（默认配置下）
        
        System.out.println("=== 执行更新操作后再次查询 ===");
        mapper.updateUserStatus(1L, 2); // 更新操作
        User user3 = mapper.selectUserById(1L); // 清空缓存，重新查询数据库
        System.out.println("查询结果: " + user3);
        
    } finally {
        sqlSession.close();
    }
}
```

**控制台输出：**
```
=== 第一次查询 ===
DEBUG: ==>  Preparing: SELECT * FROM users WHERE id = ? 
DEBUG: ==> Parameters: 1(Long)
查询结果: User{id=1, name='John'}

=== 第二次查询（相同参数） ===
查询结果: User{id=1, name='John'} 
// 注意：没有SQL日志，说明从缓存读取

是否是同一个对象: true

=== 执行更新操作后再次查询 ===
DEBUG: ==>  Preparing: UPDATE users SET status = ? WHERE id = ?
DEBUG: ==> Parameters: 2(Integer), 1(Long)
DEBUG: ==>  Preparing: SELECT * FROM users WHERE id = ? 
DEBUG: ==> Parameters: 1(Long)
查询结果: User{id=1, name='John', status=2}
```

### 3. ⼀级缓存的清空时机

```java
// 以下操作会清空一级缓存：
sqlSession.clearCache();                    // 手动清空
mapper.updateUser(user);                    // 更新操作
mapper.insertUser(user);                    // 插入操作  
mapper.deleteUser(1L);                      // 删除操作
sqlSession.commit();                        // 提交事务
sqlSession.rollback();                      // 回滚事务
```

### 4. ⼀级缓存的配置（在mybatis-config.xml中）

```xml
<settings>
    <!-- 本地缓存作用域（默认是SESSION） -->
    <setting name="localCacheScope" value="SESSION"/>
    
    <!-- 可选值：
         SESSION: 整个SqlSession期间缓存（默认）
         STATEMENT: 仅对当前语句有效，相当于关闭一级缓存
    -->
</settings>
```

**`STATEMENT`级别的使用场景**：
```xml
<settings>
    <setting name="localCacheScope" value="STATEMENT"/>
</settings>
```
适用于需要避免脏读的场景，比如在同一个事务中先查询后更新再查询，希望第二次查询能获取最新数据。

---

## 三、二级缓存（全局缓存）深度解析

### 1. 什么是⼆级缓存？
- **作用域**：`Mapper` 级别（`namespace`级别）
- **生命周期**：与应用生命周期一致
- **默认状态**：**默认关闭**，需要手动开启

### 2. 如何开启⼆级缓存？

#### 步骤1：全局开启（mybatis-config.xml）
```xml
<settings>
    <!-- 开启全局缓存（默认就是true，但显式配置更清晰） -->
    <setting name="cacheEnabled" value="true"/>
</settings>
```

#### 步骤2：在Mapper.xml中配置缓存
```xml
<!-- 最简单的开启方式 -->
<mapper namespace="com.example.mapper.UserMapper">
    <cache/>
    
    <select id="selectUserById" resultType="User">
        SELECT * FROM users WHERE id = #{id}
    </select>
</mapper>
```

#### 步骤3：实体类实现Serializable接口
```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    // ... 属性和方法
}
```

### 3. ⼆级缓存的高级配置

```xml
<cache
  eviction="FIFO"                          <!-- 回收策略：FIFO/LRU/SOFT/WEAK -->
  flushInterval="60000"                    <!-- 刷新间隔：60秒 -->
  size="512"                               <!-- 缓存对象数量 -->
  readOnly="true"                          <!-- 是否只读 -->
  type="org.mybatis.caches.ehcache.EhcacheCache"/> <!-- 自定义缓存实现 -->
```

**配置参数详解：**

| 参数 | 可选值 | 说明 |
|------|--------|------|
| `eviction` | `LRU`（最近最少使用）<br>`FIFO`（先进先出）<br>`SOFT`（软引用）<br>`WEAK`（弱引用） | 缓存淘汰策略，默认`LRU` |
| `flushInterval` | 毫秒数 | 定时清空缓存间隔，默认不清空 |
| `size` | 正整数 | 缓存最多存储的对象数，默认1024 |
| `readOnly` | `true`/`false` | `true`：返回相同对象（性能好）<br>`false`：返回拷贝对象（安全） |

### 4. ⼆级缓存的工作机制验证

```java
@Test
public void testSecondLevelCache() {
    System.out.println("=== 第一个SqlSession ===");
    SqlSession sqlSession1 = sqlSessionFactory.openSession();
    UserMapper mapper1 = sqlSession1.getMapper(UserMapper.class);
    User user1 = mapper1.selectUserById(1L); // 查询数据库
    System.out.println("第一次查询结果: " + user1);
    sqlSession1.close(); // 必须关闭，一级缓存内容才会存入二级缓存
    
    System.out.println("=== 第二个SqlSession ===");
    SqlSession sqlSession2 = sqlSessionFactory.openSession();
    UserMapper mapper2 = sqlSession2.getMapper(UserMapper.class);
    User user2 = mapper2.selectUserById(1L); // 从二级缓存获取
    System.out.println("第二次查询结果: " + user2);
    sqlSession2.close();
    
    System.out.println("=== 验证缓存命中 ===");
    // 观察日志：第二次查询应该没有SQL输出
}
```

**关键点**：二级缓存只有在`SqlSession`关闭或提交后，才会生效！

### 5. 缓存的使用策略控制

#### 在语句级别细粒度控制缓存：

```xml
<select id="selectUserById" resultType="User" 
        useCache="true"                     <!-- 是否使用缓存（默认true） -->
        flushCache="false">                 <!-- 是否清空缓存（默认false） -->
    SELECT * FROM users WHERE id = #{id}
</select>

<insert id="insertUser" flushCache="true">  <!-- 插入后清空缓存 -->
    INSERT INTO users(name, email) VALUES(#{name}, #{email})
</insert>

<update id="updateUser" flushCache="true">  <!-- 更新后清空缓存 -->
    UPDATE users SET name = #{name} WHERE id = #{id}
</update>
```

---

## 四、缓存执行流程与源码级原理

### 1. 完整的查询缓存流程

```
查询请求
    ↓
Executor.query() 
    ↓
检查二级缓存（CachingExecutor） 
    ↓
    ┌─ 缓存命中 → 返回结果
    ↓
检查一级缓存（BaseExecutor）
    ↓
    ┌─ 缓存命中 → 返回结果  
    ↓
查询数据库（真正执行SQL）
    ↓
存入一级缓存
    ↓
SqlSession关闭时 → 存入二级缓存
    ↓
返回结果
```

### 2. 源码层面的关键类

```java
// 二级缓存执行器（装饰器模式）
CachingExecutor → 负责二级缓存逻辑

// 基础执行器  
BaseExecutor → 负责一级缓存逻辑

// 缓存接口
Cache → 所有缓存实现的统一接口
```

---

## 五、实战中的缓存问题与解决方案

### 1. 脏读问题（最常见）

**场景**：多个应用实例或不同Mapper操作同一张表。

**解决方案1**：在关联的Mapper中引用同一个缓存
```xml
<!-- OrderMapper.xml -->
<mapper namespace="com.example.mapper.OrderMapper">
    <cache-ref namespace="com.example.mapper.UserMapper"/>
</mapper>
```

**解决方案2**：在更新操作中手动清空相关缓存
```xml
<update id="updateUser" flushCache="true">
    UPDATE users SET ... WHERE ...
</update>
```

### 2. 缓存穿透/击穿/雪崩

这些问题在MyBatis原生缓存中同样存在，解决方案：

**自定义缓存实现**（如集成Redis）：
```xml
<cache type="org.mybatis.caches.redis.RedisCache"/>
```

### 3. 二级缓存的最佳实践

```xml
<!-- 推荐配置：适合读多写少的场景 -->
<cache
  eviction="LRU"
  flushInterval="300000"    <!-- 5分钟自动刷新 -->
  size="1024"
  readOnly="false"/>        <!-- 保证线程安全 -->

<!-- 对实时性要求高的查询禁用缓存 -->
<select id="getRealTimeData" useCache="false">
    SELECT * FROM real_time_table
</select>
```

---

## 六、总结：缓存使用策略

| 场景 | 一级缓存 | 二级缓存 | 建议 |
|------|----------|----------|------|
| **单体应用，数据量小** | ✅ 开启 | ✅ 开启 | 简单场景全开 |
| **高并发读，数据变更少** | ✅ 开启 | ✅ 开启+合理配置 | 缓存最大化 |
| **读写频繁，数据实时性要求高** | ✅ 开启 | ❌ 关闭 | 避免脏读 |
| **分布式环境** | ✅ 开启 | ✅ 自定义缓存（如Redis） | 解决共享问题 |

**最终建议**：

- 理解业务的数据访问模式后再决定缓存策略
- 一级缓存通常保持开启，二级缓存根据实际情况谨慎使用
- 生产环境务必进行压力测试，验证缓存效果
- 监控缓存命中率，及时调整策略