---
title: "python -m 和 python 直接执行的区别"
publish_time: "2025-04-15"
updates:
hidden: false
---
<p style="color: rgba(127, 127, 127, 0.9);">水一篇笔记, 在不更新都长毛了<p>

假设目录结构如下

```txt
project/
    ├── mymodule/
    │   ├── __init__.py
    │   └── utils.py
    └── script.py
```

## 支持包的相对导入

`python -m project.script` 相当于将script看做一个模块, project 看做一个包.
`python ./project/script.py` 相当于将script看做一个脚本, project 什么也不是, 看不到更上层路径/包

## 模块搜索路径 (sys.path) 不同

`python ./project/script.py`
`sys.path[0]` 是脚本所在目录的绝对路径.

`python -m project.script`
`sys.path[0]` 是执行命令的目录 == `pwd`
