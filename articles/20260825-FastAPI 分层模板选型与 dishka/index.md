---
title: "FastAPI 分层模板选型与 dishka"
publish_time: "2026-08-25"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">事情是这样的, 在 matt skill 的加持下, 纯 AI 做了一个基于 fastapi + deep agents 的 agent 项目, 能跑, 但是代码闻着味道不对, 但是还说不出来是哪里不对, 于是就有了这篇文章.</p>

基本思路是 `api → service → repo → db`, 然后各个层级之间通过 FastAPI 提供的 `Depends` 注入依赖. 理想很丰满, 现实就很现实了.

## 1. FastAPI 官方模板

首先想到的肯定是 FastAPI 官方推荐模板:  
https://github.com/fastapi/full-stack-fastapi-template

```python
from sqlmodel import Session

def get_db() -> Generator[Session]:
    with Session(engine) as session:
        yield session


SessionDep = Annotated[Session, Depends(get_db)]

@router.get("/", response_model=ItemsPublic)
def read_items(
    session: SessionDep, current_user: CurrentUser, skip: int = 0, limit: int = 100
)
```

基本是现实满足的, 也用到了 `Depends` 注入依赖, 但是这个项目没有 Repo, 没有 Service, 在 API 里面直接查询数据库了, 不是很理想. 没关系, 继续找别的项目参考.

## 2. fastapi-best-architecture

https://github.com/fastapi-practices/fastapi-best-architecture

看项目名就起的很霸气, 最好的架构, 没有之一. 这个项目是有 Service 的, 官方文档也说了, 是三层架构. 但问题是, 传统 Python 项目的单例全局变量又出现了:

```python
from backend.database.redis import redis_client

router = APIRouter()

@router.get('', summary='获取在线用户', dependencies=[DependsSuperUser])
async def get_sessions(
    username: Annotated[str | None, Query(description='用户名')] = None,
) -> ResponseSchemaModel[list[GetTokenDetail]]:
    token_keys = await redis_client.get_by_prefix(settings.TOKEN_REDIS_PREFIX)
    online_clients = await redis_client.smembers(settings.TOKEN_ONLINE_REDIS_PREFIX)
```

然后和官方项目一样, db 是注入的. 这就有一点点两边都不靠的感觉. Service 调用的时候, 都是直接将 db 手动注入进去; 更离谱的是, Service 下面还有一层 dao (本项目叫 crud), db 又手动传了两层:

```python
class DeptService:
    @staticmethod
    async def get(*, db: AsyncSession, pk: int) -> Dept:
        dept = await dept_dao.get(db, pk)

dept_service: DeptService = DeptService()

@router.get('/{pk}', summary='获取部门详情', dependencies=[DependsJwtAuth])
async def get_dept(
    db: CurrentSession, pk: Annotated[int, Path(description='部门 ID')]
) -> ResponseSchemaModel[GetDeptDetail]:
    data = await dept_service.get(db=db, pk=pk)
    return response_base.success(data=data)
```

## 3. fastapi-clean-example

https://github.com/0xTheProDev/fastapi-clean-example

这个项目写得好. 虽然只有 500 多 star, 但是非常符合我的口味. 如果一定要鸡蛋里挑骨头, 可以看到下面的代码, 注入全都是「半自动」——在类的 `__init__` 里直接写 `Depends`, 而官方更推荐的方式是用 `dependencies.py` 统一管理组装:

```python
Engine = create_engine(
    DATABASE_URL, echo=env.DEBUG_MODE, future=True
)

SessionLocal = sessionmaker(
    autocommit=False, autoflush=False, bind=Engine
)

def get_db_connection():
    db = scoped_session(SessionLocal)
    try:
        yield db
    finally:
        db.close()

class BookRepository:
    db: Session

    def __init__(
        self, db: Session = Depends(get_db_connection)
    ) -> None:
        self.db = db

class BookService:
    authorRepository: AuthorRepository
    bookRepository: BookRepository

    def __init__(
        self,
        authorRepository: AuthorRepository = Depends(),
        bookRepository: BookRepository = Depends(),
    ) -> None:
        self.authorRepository = authorRepository
        self.bookRepository = bookRepository
```

类似的项目还有:  
https://github.com/nsidnev/fastapi-realworld-example-app

## 4. 自己的第一版: FasterAPI 1.0

参考项目 1 和 3, 我自己的实现方案第一个版本是:  
https://github.com/chaleaoch/faster_api/releases/tag/1.0

分层很直白: `API → Service → Repository → Dao / Cache`. 路由只拿 `UserService`, 组装全部塞进 `dependencies.py`, 看起来已经「官方推荐」了:

```python
async def get_user_dao(
    session: Annotated[AsyncSession, Depends(get_db)],
    client: Annotated[RedisClient, Depends(get_redis_client)],
) -> UserDao:
    return UserDao(session, client)


async def get_user_repository(
    dao: Annotated[UserDao, Depends(get_user_dao)],
    cache: Annotated[UserCache, Depends(get_user_cache)],
) -> UserRepository:
    return UserRepository(dao, cache)


async def get_user_service(
    repository: Annotated[UserRepository, Depends(get_user_repository)],
    session: Annotated[AsyncSession, Depends(get_db)],
) -> UserService:
    return UserService(repository, session)
```

路由侧则是:

```python
@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: Annotated[UserService, Depends(get_user_service)],
) -> UserResponse:
    user = await service.get_user(user_id)
    return UserResponse.model_validate(user)
```

这一版解决了两件事:

1. **分层清晰**: API 不碰 SQL, Service 管业务和事务, Repository 管 cache-aside, Dao 管 ORM.
2. **依赖可测**: 测试里可以 override 某一层的 `Depends`, 不必硬碰真实 MySQL / Redis.

但闻着还是有味道:

- **长命对象挂在 `app.state` 上**, 再通过 `Request` + `cast` 取出来. Engine / Redis client 本该是应用级单例, 却和请求级组装揉在同一条 `Depends` 链里.
- **每次请求都会 `return Xxx(...)` 新建一堆对象**. `AsyncSession` 按请求创建是对的; 但 `RedisUserCache` 这种只依赖 Redis client + TTL 配置的东西, 完全可以活在 APP 作用域, 却也被每次请求重建一次.
- **组合根散落在 `dependencies.py` 的函数树里**. 层数一多, 谁依赖谁要顺着 `Depends` 链往上翻, FastAPI 之外的入口 (CLI / 脚本 / 后台任务) 也很难复用同一套组装.

一句话: 1.0 已经是「能用的分层 + Depends」, 但还缺一个真正的 IoC 容器——尤其是作用域.

## 5. 引入 dishka: FasterAPI 2.0

调查过程中发现了 [dishka](https://github.com/reagento/dishka). 它是一个轻量的 Python DI 容器, 核心就三块:

| 概念          | 做什么                                           |
| ------------- | ------------------------------------------------ |
| **Provider**  | 声明「怎么造出某个类型」                         |
| **Scope**     | 声明「这个对象活多久」——常见是 `APP` / `REQUEST` |
| **Container** | 按图解析依赖, 在作用域内缓存实例                 |

和 FastAPI `Depends` 的差别可以粗暴理解成:

- `Depends`: 框架自带的请求级工厂链, 作用域几乎只有「这个请求」.
- `dishka`: 独立的组合根. 你可以明确说 Database 是 APP 级、Session 是 REQUEST 级; 同一作用域内多次 `get` 同一个类型, 拿到的是同一个实例; 请求结束自动清理 `yield` 出来的资源.

FastAPI 集成也很薄: `setup_dishka` 挂上中间件管 REQUEST 作用域, 路由用 `FromDishka[T]` 取值, 可选 `DishkaRoute` 省掉手写 `@inject`.

于是把 4 和 dishka 合到一起, 就有了 2.0:  
https://github.com/chaleaoch/faster_api/releases/tag/2.0

组合根集中在 `app/ioc.py`:

```python
class AppProvider(Provider):
    @provide(scope=Scope.APP)
    async def database(self, settings: Settings) -> AsyncIterator[Database]:
        database = Database(settings)
        yield database
        await database.dispose()

    database_lifecycle = alias(source=Database, provides=DatabaseLifecycle)

    @provide(scope=Scope.APP)
    async def redis_client(self, settings: Settings) -> AsyncIterator[RedisClient]:
        client = create_redis_client(settings)
        yield client
        await client.aclose()

    @provide(scope=Scope.APP)
    def user_cache(self, client: RedisClient, settings: Settings) -> UserCache:
        return RedisUserCache(
            client,
            ttl_seconds=settings.redis_user_cache_ttl_seconds,
        )

    @provide(scope=Scope.REQUEST)
    async def session(self, database: Database) -> AsyncIterator[AsyncSession]:
        async with database.session_factory() as session:
            yield session


class UserProvider(Provider):
    scope = Scope.REQUEST

    user_dao = provide(UserDao)
    user_repository = provide(UserRepository)
    user_service = provide(UserService)
```

对照 1.0, 变化很直观:

- `Database` / `RedisClient` / `UserCache` → **APP**: 进程内只建一次, `yield` 的收尾在 `container.close()` 时跑.
- `AsyncSession` / `UserDao` / `UserRepository` / `UserService` → **REQUEST**: 跟着一次 HTTP 请求走; Service / Dao 持有 Session, 本来就该是请求级, 这里不是浪费, 是正确的生命周期.
- 构造函数只写类型注解, dishka 按类型自动接线. 不再手写一整棵 `get_xxx → get_yyy` 工厂树.

启动时把容器绑到应用上:

```python
container = make_container(
    resolved_settings,
    extra_providers=(FastapiProvider(), *extra_providers),
)
setup_dishka(container, app)
```

路由侧几乎只剩业务:

```python
router = APIRouter(
    prefix="/api/v1/users",
    tags=["users"],
    route_class=DishkaRoute,
)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: FromDishka[UserService],
) -> UserResponse:
    user = await service.get_user(user_id)
    return UserResponse.model_validate(user)
```

额外的好处: 非 HTTP 入口可以复用同一个 `make_container()`, 不必再抄一份 Depends 链:

```python
container = make_container()
try:
    async with container() as request_container:
        service = await request_container.get(UserService)
        await service.list_users(offset=0, limit=20)
finally:
    await container.close()
```

测试也不再靠 `app.dependency_overrides`, 而是丢一个 `override=True` 的 Provider 进 `extra_providers`, 替换某层实现即可.

## 小结

| 方案                      | 分层              | 注入方式                    | 痛点                                 |
| ------------------------- | ----------------- | --------------------------- | ------------------------------------ |
| 官方 full-stack           | 基本没有          | Depends 注入 Session        | API 直连 DB                          |
| fastapi-best-architecture | 有 Service / crud | 全局单例 + 手动传 db        | 风格分裂, db 传两层                  |
| fastapi-clean-example     | 清晰              | `__init__` 里半自动 Depends | 组装分散, 作用域弱                   |
| FasterAPI 1.0             | 清晰              | dependencies.py 工厂树      | 每请求重建对象, app.state 取长命资源 |
| FasterAPI 2.0 + dishka    | 清晰              | Provider + APP/REQUEST      | 组合根集中, 生命周期说清楚           |

对我来说, dishka 不是「又一个黑魔法框架」, 而是把原本散在 `Depends` / `app.state` / 全局变量里的组装规则, 收拢成一份可读的组合根, 并补上 FastAPI 自己不太想管的作用域问题.

<完>
