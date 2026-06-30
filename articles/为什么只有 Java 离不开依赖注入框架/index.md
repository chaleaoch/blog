---
title: "为什么只有 Java 离不开依赖注入框架"
publish_time: "2026-06-30"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">
  因为昨天晚上看到 gin 关于 DI 的描述
  <a href="https://gin-gonic.com/zh-cn/docs/middleware/dependency-injection/">
    随着 Gin 应用的增长，你需要一种简洁的方式在处理函数之间共享数据库连接、配置和服务等依赖。Go 的简洁性鼓励使用直接的模式，而不是重量级的 DI 框架
  </a>
  勾起了我的写作欲望...
</p>

首先要说清楚一件事， 所有的语言都需要依赖注入， 无论是 python 还是 go， 只不过这两门语言的依赖注入的形式很多， 完全不需要依赖注入框架。

## gin

譬如上面链接中提到的三种注入方式

1. 闭包

```Go
package main

import (
  "database/sql"
  "net/http"

  "github.com/gin-gonic/gin"
  _ "github.com/lib/pq"
)

func GetUserHandler(db *sql.DB) gin.HandlerFunc {
  return func(c *gin.Context) {
    id := c.Param("id")
    var name string
    // 这里将db注入进来了
    err := db.QueryRowContext(c.Request.Context(), "SELECT name FROM users WHERE id = $1", id).Scan(&name)
    if err != nil {
      c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
      return
    }
    c.JSON(http.StatusOK, gin.H{"name": name})
  }
}

func main() {
  db, err := sql.Open("postgres", "postgres://user:pass@localhost/dbname?sslmode=disable")
  if err != nil {
    panic(err)
  }
  defer db.Close()

  r := gin.Default()
  r.GET("/ping", PingHandler(db))
  r.GET("/users/:id", GetUserHandler(db))
  r.Run(":8080")
}
```

2. 结构体注入

另外， 我们实际开发中， 使用最多的就是结构体注入， 只不过， 这里为了简化代码结构注入的是 db， 实际工程上注入的是 service。

```Go
package main

import (
  "database/sql"
  "log/slog"
  "net/http"

  "github.com/gin-gonic/gin"
  _ "github.com/lib/pq"
)

type App struct {
  DB     *sql.DB // 这里注入的
  Logger *slog.Logger
}

func (a *App) GetUser(c *gin.Context) {
  id := c.Param("id")
  var name string
  err := a.DB.QueryRowContext(c.Request.Context(), "SELECT name FROM users WHERE id = $1", id).Scan(&name)
  if err != nil {
    a.Logger.Error("user not found", slog.String("id", id), slog.String("error", err.Error()))
    c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
    return
  }
  c.JSON(http.StatusOK, gin.H{"id": id, "name": name})
}

func (a *App) CreateUser(c *gin.Context) {
  var input struct {
    Name string `json:"name" binding:"required"`
  }
  if err := c.ShouldBindJSON(&input); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
  }

  _, err := a.DB.ExecContext(c.Request.Context(), "INSERT INTO users (name) VALUES ($1)", input.Name)
  if err != nil {
    c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to create user"})
    return
  }
  c.JSON(http.StatusCreated, gin.H{"name": input.Name})
}

func main() {
  db, err := sql.Open("postgres", "postgres://user:pass@localhost/dbname?sslmode=disable")
  if err != nil {
    panic(err)
  }
  defer db.Close()

  app := &App{
    DB:     db,
    Logger: slog.Default(),
  }

  r := gin.Default()
  r.GET("/users/:id", app.GetUser)
  r.POST("/users", app.CreateUser)
  r.Run(":8080")
}
```

3. 中间件注入

中间件注入感觉就是为了凑数的\.\.\.

```Go
func DatabaseMiddleware(db *sql.DB) gin.HandlerFunc {
  return func(c *gin.Context) {
    c.Set("db", db)
    c.Next()
  }
}

func GetUser(c *gin.Context) {
  db := c.MustGet("db").(*sql.DB)
  // Use db...
}

func main() {
  db, _ := sql.Open("postgres", "postgres://user:pass@localhost/dbname?sslmode=disable")

  r := gin.Default()
  r.Use(DatabaseMiddleware(db))
  r.GET("/users/:id", GetUser)
  r.Run(":8080")
}
```

## golang

在实际的工程实践中， 一种比较常见的注入方式是通过接口参数， 你可以替换某个组件的实现来达到解耦的目的。

```Go
type Nower func() time.Time

func Foo(t Time.Time, now Nower) {
  n := now()
  *// ...*
}

*// 调用传参*
Foo(t, time.Now)
// 实现time.Now和Foo的解耦, 譬如,
Foo(t, func() time.Time {
  s := "2021-05-20T15:34:20Z"
  t, _ := time.Parse(time.RFC3339, s)
  return t
})
```

## python

上面的例子引出了一个新的话题， python 实际上也不是很强调依赖注入框架， 因为本质上不需要， 譬如，

```Python
class RedisList:
    def __init__(self, redis_client)
        self._client = redis_client
    def push(self, key, val):
        self._client.lpush(key, val)
redis_client = get_redis_client(...)
l = RedisList(redis_client)
```

因为 python 的动态类型特点， 这是一种特别自然的写法， 根本不需要框架为我们注入。 在譬如 Django 的另一种实现注入的方式， 这种能力还是依托于 python 的 importlib 的动态特性。

```JSON
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/0",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        },
    }
}
```

```Plain Text
from django.core.cache import cache

cache.set("user:1", "Tom", timeout=60)
value = cache.get("user:1")
```

## 全局变量算不算实现了 IOC？

不严谨的说， import 全局变量实际上确实实现了控制反转， 稍加变化， 譬如， 我维护一个超大的 dict， 然后再写一个函数， 参数是 key， 返回的变量是 value， 这不就是简化版的依赖查找嘛。

```Python
# 统一的依赖容器（注册表）
container = {
    "db": Database(),
    "cache": RedisCache()
}

# 统一的查找入口
def get_service(key: str):
    return container[key]

# 业务代码：主动发起查找，获取依赖
def get_user(user_id: int):
    db = get_service("db")  # 明确的查找动作
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    return user

print(get_user(1))
```

这种全局变量的写法优点是， 简单， 不需要浪费脑细胞， 凭直觉就把代码写完了。 但是缺点就很严重了，

1. 并发场景下的线程安全隐患
2. 依赖结构不清晰
3. 单元测试很难写（这里特指 golang）

但是为了实现方便写单元测试， 在开发业务代码的时候， 需要为了依赖编写大量的接口参数， 而整个项目中每个接口只有一个实现， 有些得不偿失。

目前还没有找到更好的办法完美的解决这一对儿矛盾。

## **三个总被搞混的词：DIP、IoC、DI**

为了满足 DIP（原则），我们采用 IoC（思想），而落地 IoC 最常用的招式就是 DI（手段）。挺无聊的， 业务面试的时候会用到。

---

参考链接：

1. https://www\.zhihu\.com/question/32108444/answer/581948457
2. https://tao\.zz\.ac/go/mock\.html\#%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5
3. https://www\.zhihu\.com/question/521822847/answer/2390238343
