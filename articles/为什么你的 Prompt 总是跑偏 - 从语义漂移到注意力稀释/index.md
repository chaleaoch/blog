---
title: "为什么你的 Prompt 总是跑偏 - 从语义漂移到注意力稀释"
publish_time: "2026-08-05"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">
  Transformer 每一步都基于当前上下文计算下一个 token 的概率；生成出的 token 会加入上下文，从而改变下一步的概率分布。
</p>

## 原理

1. 语义漂移, 拐弯了

![image.png](./index/attachments/image9.png)

2. 注意力稀释, 一开始的要求忘记了,

![image.png](./index/attachments/image3.png)

3. 语义惯性, 上文决定下文,

![image.png](./index/attachments/image8.png)

4. 语义壁垒, 太难的题AI不会, 让他先思考/推理而不是直接输出答案, AI一着急就会说错\. COT,Chain of Thought,思维链

![image.png](./index/attachments/image.png)

5. 相变, 即使是胡言乱语, 也可能带来意外的惊喜, 但是说错了容易带来前面的\<语义漂移和注意力稀释\>

![image.png](./index/attachments/image6.png)

6. 特征纠缠, 你想让模型增加一个特点，却顺带带出了其他相关特点。譬如,

例如你说, "回答要严谨",

你想要的, “逻辑准确”，但模型可能同时变得, 学术化/术语很多

所以提示词应该变成, "逻辑要严谨，但表达简短、口语化，不展开背景。"

## 实战技巧

1. 复杂问题, 如果约束太多反而限制AI, 所以可以先发散后收敛

![image.png](./index/attachments/image11.png)

2. **先推理后结论**, 很多人\(包括我\)喜欢第一句话就给出结论, 后面再扩展, 但是这样的话, 会导致AI后面的扩展全都在圆这个结论而展开, 如果结论给错了就麻烦了\.

![image.png](./index/attachments/image5.png)

3. 入戏与共振采样, 靠连续重复同类关键词，把 AI 锁死在指定人设里，不让它跑偏。

![image.png](./index/attachments/image10.png)

4. 隐式语义优于显式语义
   1. 隐式语义 vs 显示语义, **当然这里的显示和隐式是相对的, 否则, 任何指令都可以认为是显示的\.**

   ```Plain Text
   隐式语义: 请按照 Kubernetes controller-runtime 项目的代码质量标准实现这个 HTTP client。
   显示语义: 不要使用全局变量。 HTTP 请求必须设置超时。 失败后最多重试两次。 使用结构化日志。
   ```

   2. 但是隐式语义会带来一个新的问题, 就是特征纠缠, 所以有需要通过**显示配平**收敛\.

![image.png](./index/attachments/image1.png)

    3. 但是显示配平会带来新的问题,

        1. 知识冗余,

![image.png](./index/attachments/image2.png)

    参考上图, 实际上AI已经知道md语法, 不需要显示指定, 你指定AI以为你在强调, 会导致**注意力劫持**,

![image.png](./index/attachments/image13.png)

    和 **认知降维**, 也就是AI的能力其实比你强大

![image.png](./index/attachments/image15.png)

    但是因为你的约束太多, 限制了AI的能力, 譬如, 前面提到的注意力漂移, 所以,

    4. 隐式提纯

![image.png](./index/attachments/image14.png)

以上说了这么多, 变成`咸了-->加水 --> 淡了 --> 加盐 `的死循环,

## 结论

其实结论也挺长的,

1. 引导采样与回滚, 先用一堆话帮模型“想清楚”，拿到准确关键词；然后开新对话，只用这些关键词重新提问。

![image.png](./index/attachments/image12.png)

2. 案例好于说明,

比如 ls \-al \| grep 的单行代码就已经告诉了模型这是 linux 生态，而不用花更多文字去说明

3. 引入数学符号和数学语义

![image.png](./index/attachments/image7.png)

![image.png](./index/attachments/image4.png)

> 参考并感谢: https://linux.do/t/topic/2538870
