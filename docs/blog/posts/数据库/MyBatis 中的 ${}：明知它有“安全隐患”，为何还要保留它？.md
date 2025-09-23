---
date: 2025-09-23
title: MyBatis 中的 ${}：明知它有“安全隐患”，为何还要保留它？
categories:
  - 数据库
draft: false
comments: true
---
在MyBatis的世界里，#{}和${}这对看似相似的符号，却扮演着截然不同的角色。我们都被教导要警惕${}的SQL注入风险，把它视为代码中的“安全隐患”。但一个有趣的问题随之而来：如果它真的如此危险，为什么MyBatis的设计者没有直接将其抛弃？
<!-- more -->

作为一名Java开发者，只要你用过MyBatis，就一定被前辈或文档告诫过：**“能用 `#{}` 就别用 `${}`，小心SQL注入！”** 这个建议完全正确，`${}` 确实是我们代码中一个潜在的安全风险点。

那么，一个自然而然的问题就产生了：既然 `${}` 这么危险，像个“坏孩子”，为什么MyBatis的设计者不一劳永逸地移除它，强制大家使用安全的 `#{}` 呢？

今天，我们就来深入探讨一下这个“坏孩子”存在的意义，你会发现，在某些场景下，它甚至是**不可替代**的帮手。

### 一、重温“好学生”与“坏孩子”的根本区别

在理解其存在意义前，我们必须先彻底弄懂 `#{}` 和 `${}` 的本质差异。

*   **`#{}`（好学生）：参数占位符**
    *   **工作方式**：MyBatis会将其转换为JDBC中的 `?`，并使用 `PreparedStatement` 进行**预编译**。
    *   **安全性**：参数值在SQL执行时才被传入，数据库驱动会对其进行严格的转义，**从根本上杜绝了SQL注入**。
    *   **类比**：就像填空题，语句的骨架是固定的，只是填入具体的值。

    ```xml
    <select id="findUserById" resultType="User">
        SELECT * FROM users WHERE id = #{userId}
    </select>
    ```
    **最终执行的SQL：** `SELECT * FROM users WHERE id = ?` （参数`userId`被安全地设置进去）

*   **`${}`（坏孩子）：字符串替换**
    *   **工作方式**：MyBatis会直接将其中的内容**拼接**到SQL语句中。
    *   **危险性**：因为内容是直接拼接，如果内容来自用户输入且未经验证，就可能篡改原始SQL结构，导致**SQL注入攻击**。
    *   **类比**：就像拼积木，直接把一块积木（字符串）嵌入到整个SQL结构中。

    ```xml
    <!-- 假设 orderBy 参数来自用户输入 -->
    <select id="findUsers" resultType="User">
        SELECT * FROM users ORDER BY ${orderBy}
    </select>
    ```
    如果用户传入 `orderBy = "name; DROP TABLE users; --"`，拼接后的SQL将是：
    `SELECT * FROM users ORDER BY name; DROP TABLE users; --`
    **你的用户表可能就瞬间消失了！**

看到这里，你可能会更坚定地认为 `${}` 是个祸害。别急，让我们进入关键部分。

### 二、`${}` 的“高光时刻”：不可替代的应用场景

MyBatis保留 `${}`，绝不是因为设计失误，而是因为在某些领域，`#{}` 这位“好学生”**无能为力**。`${}` 是用来处理 **SQL语句本身的结构**，而不是**值**。

#### 场景一：动态排序（ORDER BY）

这是最常见的必须使用 `${}` 的场景。

```xml
<select id="findUsers" resultType="User">
    SELECT * FROM users
    ORDER BY ${sortField} ${sortOrder}
</select>
```

*   **为什么不能用 `#{}`？**
    如果写成 `ORDER BY #{sortField}`，MyBatis会将其预处理为 `ORDER BY ?`，然后传入 `"create_time"` 这个字符串。最终SQL变成：
    `ORDER BY 'create_time'`
    这会导致数据库按一个**常量**排序，而不是按 `create_time` 列排序，结果完全错误。

#### 场景二：动态表名与列名

在分库分表或某些动态 schema 设计中非常有用。

```xml
<!-- 按年份分表查询 -->
<select id="findLogByYear" resultType="Log">
    SELECT * FROM log_${year}
</select>

<!-- 动态选择查询的列 -->
<select id="getDynamicData" resultType="map">
    SELECT ${columns} FROM some_table
</select>
```

*   **为什么不能用 `#{}`？**
    同理，`FROM #{tableName}` 会被处理为 `FROM 'log_2023'`，数据库会认为你在查询一个名为 `'log_2023'` 的表（通常带引号），而不是 `log_2023` 表。

#### 场景三：SQL关键字与函数

当你需要动态使用SQL函数或关键字时。

```xml
<select id="findWithDynamicFunction" resultType="Data">
    SELECT ${functionName}(column) FROM table
</select>
```

### 三、如何与“坏孩子”安全共处？—— 给 `${}` 戴上枷锁

既然不得不使用 `${}`，我们绝不能放任自流。核心原则是：**绝对不让用户直接控制 `${}` 的内容！**

解决方案是：**白名单校验**。

我们不应该信任任何前端传递过来的、用于 `${}` 的参数。必须在后端进行严格的校验。

**反面教材（绝对禁止！）：**
```java
// 前端传什么，SQL就拼什么，这是自杀行为
userMapper.findUsers(request.getSortField()); 
```

**正确做法（安全卫士）：**
```java
public List<User> findUsersSafe(String sortField, String sortOrder) {
    // 1. 定义允许排序的字段白名单
    Set<String> allowedFields = new HashSet<>(Arrays.asList("id", "name", "create_time"));
    Set<String> allowedOrders = new HashSet<>(Arrays.asList("ASC", "DESC"));
    
    // 2. 校验参数是否在白名单内
    if (!allowedFields.contains(sortField)) {
        sortField = "id"; // 不在白名单，使用默认值
    }
    if (!allowedOrders.contains(sortOrder.toUpperCase())) {
        sortOrder = "ASC";
    }
    
    // 3. 将校验后的、安全的参数传递给Mapper
    return userMapper.findUsers(sortField, sortOrder);
}
```

在这个安全的版本中，即使用户传入了恶意的 `sortField` 值，也会被后端的白名单机制过滤掉，确保最终拼接进SQL的是我们认可的安全内容。

### 四、总结

回到最初的问题：**为什么明知道 `${}` 不安全，MyBatis还要保留它？**

答案现在已经很清晰了：

1.  **职责不同**：`#{}` 用于传递 **值（Data）**，而 `${}` 用于动态指定 **SQL语句的组成部分（Structure）**，如表名、列名、排序关键字等。
2.  **不可替代性**：在需要动态改变SQL结构的场景下，`#{}` 由于预编译机制的限制，无法完成任务。`${}` 提供了这种必要的灵活性。
3.  **权衡之道**：MyBatis的设计哲学是“权力与责任对等”。它给了开发者强大的灵活性（`${}`），同时也要求开发者肩负起安全使用的责任。

因此，`${}` 不是MyBatis的“设计缺陷”，而是一个强大的“高级功能”。它就像一把锋利的手术刀，在经验丰富的外科医生手中能救死扶伤，但若交给孩童则危险万分。

**最终的建议依然是：**

-   **默认、无条件地使用 `#{}`。**
-   当且仅当需要动态改变SQL结构时，才考虑 `${}`。
-   **一旦使用 `${}`，必须配合严格的白名单校验机制，绝不信任任何用户输入。**
