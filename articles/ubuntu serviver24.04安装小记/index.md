---
title: "Ubuntu Serviver24.04安装小记"
publish_time: "2024-09-02"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">网上搜了一圈没有介绍静态IP安装的.我来记一笔吧.<p>

![1](./index/attachments/image.png)  

![2](./index/attachments/image-1.png)  

这里我选的minimized方式安装  

![3](./index/attachments/image-2.png)  

选Edit IPv4  

![4](./index/attachments/image-24.png)  

![5](./index/attachments/image-23.png)  

静态IP配置  
    1. Subnet, 如果你的网关是 `192.168.32.1/24` 这里填: `192.168.32.0/24`  
    如果你的网关是如果你的网关是 `192.168.32.1/16` 这里填: `192.168.0.0/16`  
    一般来说小型局域网都是`/24`的  
    2. Address, 就是你的IP  
    3. Gateway, 这里填`192.168.32.1`, 没有`/24`  
    4. Nameservers和Search domains, 和DNS有关, 可填可不填.  
    5.
![6](./index/attachments/image-21.png)  

![7](./index/attachments/image-22.png)  

![8](./index/attachments/image-8.png)  

选Custom storage layout  

![9](./index/attachments/image-9.png)  

![10](./index/attachments/image-10.png)  

这里说法很多, 我选择了最简单的那种.  

![11](./index/attachments/image-3.png)  

![12](./index/attachments/image-12.png)  

![13](./index/attachments/image-13.png)  

![14](./index/attachments/image-14.png)  

![15](./index/attachments/image-15.png)  

![16](./index/attachments/image-16.png)  

![17](./index/attachments/image-17.png)  
