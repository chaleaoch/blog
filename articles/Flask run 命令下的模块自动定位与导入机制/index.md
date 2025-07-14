---
title: "Flask run 命令下的模块自动定位与导入机制"
publish_time: "2025-07-14"
updates:
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">在开发一个开源filter库的过程中, 需要用到写一个flask demo app去测试,因为和之前熟悉的项目结构不一样, 引起我得好奇, debug一下, 找到了其中的道道.</p>

版本信息:

```
Flask == 3.1.1
```

```python
def prepare_import(path: str) -> str:
    """Given a filename this will try to calculate the python path, add it
    to the search path and return the actual module name that is expected.
    """
    path = os.path.realpath(path)

    fname, ext = os.path.splitext(path)
    if ext == ".py":
        path = fname

    if os.path.basename(path) == "__init__":
        path = os.path.dirname(path)

    module_name = []

    # move up until outside package structure (no __init__.py)
    while True:
        path, name = os.path.split(path)
        module_name.append(name)

        if not os.path.exists(os.path.join(path, "__init__.py")): # 从app往上找, 找第一个不包含__init__.py的目录
            break

    if sys.path[0] != path:
        sys.path.insert(0, path) # 这里将路径添加到 sys.path 中

    return ".".join(module_name[::-1])
```

<完>
