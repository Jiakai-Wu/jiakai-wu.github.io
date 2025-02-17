---
title: Mybatis的mapper中$与#区别
date: 2023-02-04 21:14:15 +0800 
categories: [java, spring, Mybatis]
tags: [history, spring, mybatis] 
---
### 浅谈mybatis mapper.xml文件中$和＃的区别

`#{}`表示一个占位符即`?`，可以有效防止sql注入。在使用时不需要关心参数值的类型，mybatis会自动进行java类型和jdbc类型的转换。

`#{}`可以接收简单类型值或pojo属性值，如果传入简单类型值，`#{}`括号中可以是任意名称。

```xml
<!-- 根据名称模糊查询用户信息 -->
 <select id="findUserById" parameterType="String" resultType="user">
  select * from user where username like CONCAT(CONCAT('%', #{name}), '%')
 </select>
```

`${}`可以将parameterType 传入的内容拼接在sql中且不进行jdbc类型转换。

`${}`可以接收简单类型值或pojo属性值，如果传入简单类型值，${}括号中名称只能是value。

```xml
<!-- 根据名称模糊查询用户信息 -->
 <select id="selectUserByName" parameterType="string" resultType="user">
  select * from user where username like '%${value}%'
 </select>
```

对于`order by`排序，使用`#{}`将无法实现功能，应该写成如下形式：

`ORDER BY ${columnName}`

另外，对于mybatis逆向工程生成的代码中，进行模糊查询调用`andXxxLike()`方法时，需要手动加％，如下：

```java
CustomerExample customerExample = new CustomerExample();
customerExample.or().andNameLike("%"+name+"%");
```
