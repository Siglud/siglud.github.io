---
layout: post
title:  "Kotlin+Spring 开发坑点一览（三）"
date:   2018-08-05 19:05:00 +0800
author: Siglud
categories:
  - kotlin
tags:
  - kotlin
  - spring
  - Mockito
comment: true
share: true
---

## Mybatis 连表查询

MyBatis 连表查询时铁定是要用 mapper 了，考虑这么一个简单的一对一连表

对应的 data class 为：

```kotlin
data class User(val uid: Int = 0, val name: String = "")

data class Score(val uid: Int = 0, val score: Int = 0)

data class UserScore(val uid: Int = 0, val user: User = User(), val score: Score = Score())
```

对应的 XML ：

```xml
    <resultMap id="userScore" type="UserScore">
        <id column="u_uid" property="uid" javaType="Int" />
        <association property="user" javaType="User" columnPrefix="u_" autoMapping="true"/>
        <association property="score" javaType="Score" columnPrefix="s_" autoMapping="true" />
    </resultMap>
    
    <select id="getUserScore" parameterType="Int" resultMap="userScore">
        SELECT u.uid AS u_uid, u.name AS u_name, s.uid AS s_uid, s.score AS s_score FROM user u INNER JOIN score s ON u.uid = s.uid WHERE u.uid = #{uid}
    </select>
```

测试代码：
```kotlin 
    @Autowired
    private lateinit var dao: Dao

    @Test
    @DisplayName("测试")
    @Sql("classpath:/test_sql/test.sql")
    fun getUserScoreTest() {
        val userScore = dao.getUserScore(1)
        Assertions.assertEquals(userScore.user.uid, 1)
        Assertions.assertEquals(userScore.user.name, "aaa")
        Assertions.assertEquals(userScore.score.score, 80)
        Assertions.assertEquals(userScore.score.uid, 1)
    }
```
