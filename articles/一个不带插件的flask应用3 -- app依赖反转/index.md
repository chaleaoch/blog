---
title: "一个不带插件的flask应用3 -- app依赖反转"
publish_time: "2024-04-08"
hidden: false
---

虽然flask的设计初衷是一个项目一个app. 但在实践中, 一个项目可能会有多个app, 且各个app可能是多个团队开发. 这时候, 我们希望各个app和flask app之间的依赖关系反转, 这样各个app更加独立.

目录结构

```txt
├── README.md
├── pdm.lock
├── pyproject.toml
├── src
│   ├── app.py
│   ├── apps
│   │   ├── blueprint.py
│   │   ├── example
│   │   │   ├── __init__.py
│   │   │   ├── api.py
│   │   │   ├── models.py
│   │   │   ├── pydantic_model.py
│   │   │   └── urls.py
│   │   ├── exmaple2
│   │   │   ├── api.py
│   │   │   └── urls.py
│   │   ├── fields.py
│   │   ├── filters.py
│   │   ├── pagination.py
│   │   ├── response.py
│   │   └── views.py
│   ├── cli
│   │   ├── __init__.py
│   │   └── migrate
│   │       ├── __init__.py
│   │       ├── migrate.py
│   │       └── models_init_copy
│   │           ├── __init__.py
│   │           └── example.py
│   ├── config.py
│   ├── extensions.py
│   ├── middleware
│   │   ├── redis.py
│   │   └── utils.py
```

app.py

```python
from extensions import app_router_manager

def create_app(test_config: dict[str, Any] = None) -> Flask:
    app = Flask(__name__)
    # ...
    app_router_manager.init_app(app)
```

extensions.py

```python
class AppRouterManager:
    def __init__(self, app=None) -> None:
        self.app = None

    def init_app(self, app):
        self.app = app
        self.load_apps()

    def load_apps(self):
        for app_name in os.listdir(os.path.join(self.app.config["PROJECT_PATH"], "apps")):
            if (
                os.path.isdir(os.path.join(self.app.config["PROJECT_PATH"], "apps", app_name))
            ):
                import_module(f"apps.{app_name}.urls")

    def register(self, blueprint):
        self.app.register_blueprint(blueprint)


app_router_manager = AppRouterManager()
```

apps/example/urls.py

```python
from extensions import app_router_manager

bp = ApiBlueprint("api", __name__, url_prefix="/example")
# ...

app_router_manager.register(bp)
```
