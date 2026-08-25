---
title: "example"
publish_time: "2025-07-11"
updates:
  - time: "2022-08-10"
    description: "macOS 文件共享服务"
hidden: true
---

<p style="color: rgba(127, 127, 127, 0.9);">事情是这样的, 在matt skill的加持下, 纯AI做了一个基于fastapi + deep agents的agent项目, 能跑, 但是代码闻着味道不对, 但是还说不出来是哪里不对, 于是就有了这篇文章</p>

基本思路是api --> service --> repo --> db, 然后各个层级之间通过fastapi提供的Depends注入依赖, 理想很丰满, 现实就很现实了,

首先想到的肯定是fastapi官方推荐模板:
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

基本是现实满足的, 也用到了Depends注入依赖, 但是这个项目没有Repo, 没有service, 在api里面直接查询数据库了, 不是很理想, 没关系继续找别的项目参考.

2. https://github.com/fastapi-practices/fastapi-best-architecture
   看项目名就起的很霸气, 最好的架构, 没有之一.
   这个项目是有service的. 官方文档也说了, 是三层架构. 但问题是, 传统python项目的单例全局变量的注入方式又出现了,

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

然后和官方的项目一样, db是注入的方式进行的. 这就有一点点两边都不靠的感觉, 然后service调用的时候,都是直接将db手动注入进入, 更离谱的是, service下面还有一层dao, 本项目叫crud, db手动传了两层.

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

3.  https://github.com/0xTheProDev/fastapi-clean-example
    这个项目写得好, 虽然只有500多star, 但是非常符合我的口味, 如果一定要鸡蛋里挑骨头, 可以看到, 下面的代码, 注入全都是"半自动", 而官方推荐的方式是使用dependencies.py的方式统一管理,

```python
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

<完>

```

```
