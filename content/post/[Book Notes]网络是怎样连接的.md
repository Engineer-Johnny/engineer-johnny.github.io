---
title: "[Book Notes]网络是怎样连接的"
date: 2022-11-29T23:12:04+08:00
draft: false
---

# 网络是怎样连接的 - 户根勤
## 第一章 浏览器生成消息 --- 探索浏览器内部
## 第二章 用电信号传输TCP-IP --- 探索协议栈和网卡
### 2.1 创建套接字

- 协议栈内部有一块用于存放控制信息的内存空间
- 创建套接字，会分配其所需的内存空间，然后写入初始状态
- 该套接字会获得一个唯一文件描述符（fd）
- 连接：就是将服务端和套接字和客户端的套接字进行套在一起，形成管道。
- 通过套接字中的控制信息完成连接过程中，做好数据收发的准备，并且会分配一块临时内存存放收发数据的临时空间，称之为缓冲区
- ![image.png](https://cdn.nlark.com/yuque/0/2022/png/27466443/1669729511531-a6711d9a-9fce-4673-aa61-1429a91171cc.png#averageHue=%23b1dae9&clientId=u4fa29f8e-1a7c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=832&id=ue2b70044&margin=%5Bobject%20Object%5D&name=image.png&originHeight=832&originWidth=690&originalType=binary&ratio=1&rotation=0&showTitle=false&size=503085&status=done&style=none&taskId=u89350335-5cab-49c7-9e54-44cd1bf05c0&title=&width=690)
- socket库中connect (<fd>, <IP:Port>), 客户端（发送方）的套接字根据这个找到服务器（接收方）的套接字。将TCP头部SYN设置为1,表示连接。服务器返回相应时候，也同样类似设定TCP头部，SYN=1, ACK=1,表示接收到了相应的数据包。同样发送方也设置ACK=1,发回接收方。这也就是三次握手。
- 在调用close断开之前，连接将会一直存在。connection操作结束之后，协议栈的操作就退出了，控制权交给上游的应用程序。
- 应用程序调用write将发送的数据交给协议栈，对于协议栈来看，要发送的数据就是有一定长度的二进制字节流罢了。
- 协议栈会将数据存放在内部发送缓冲区中。这样作是为了避免发送小数据包，导致网络效率下降。要积累多少数据才发送，不同操作系统会有不同体现。但是主要由二方面来判断：
   - 1. 每个网络数据包可以容纳的TCP数据最大长度，协议栈用MSS表示。网络数据包 = 头部长度 + 数据长度，网络数据包最大长度，用MTU表示，在以太网中MTU一般是1500字节。
   - 2. 时间。协议栈会在一定时间之后，将网络数据包发送出去。
   - 3. 上述两者从某种意义上是矛盾的，他们的实际表现，由协议栈的开发者决定的，因此不同协议栈和操作系统上有不同体现。
   - 4. 一般来说，协议栈会提供给应用程序接口，让应用程序可以控制发送时机，比如缓冲区没有填满就可以发送数据包等。
- ![image.png](https://cdn.nlark.com/yuque/0/2022/png/27466443/1669730666478-2cdbc9f9-9150-4407-afd4-3270e867b688.png#averageHue=%23fafaf7&clientId=u4fa29f8e-1a7c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=406&id=ua29e3d07&margin=%5Bobject%20Object%5D&name=image.png&originHeight=406&originWidth=879&originalType=binary&ratio=1&rotation=0&showTitle=false&size=171754&status=done&style=none&taskId=u17160f82-1109-4ca8-9a03-33204214f1f&title=&width=879)
- 如果HTTP请求带表单数据，发送缓冲区数据会超过MSS长度，那么缓冲区数据会以MSS长度为单位进行拆分。每块数据会包装成单独的网络数据包。也就是每块数据前面加上TCP头部。根据套接字的控制信息标记发送方和接收方的端口，然后交给IP模块。
- ![image.png](https://cdn.nlark.com/yuque/0/2022/png/27466443/1669730886302-6095a738-e944-42be-84b6-6556a8a47612.png#averageHue=%23f9f9f9&clientId=u4fa29f8e-1a7c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=683&id=u5bc23d15&margin=%5Bobject%20Object%5D&name=image.png&originHeight=683&originWidth=876&originalType=binary&ratio=1&rotation=0&showTitle=false&size=193243&status=done&style=none&taskId=u85cdb98e-4d92-4bb9-876c-6cc9df3963d&title=&width=876)
- TCP头部的序号，为了安全性，并非从1开始计数，而是随机数一个初始值。发送方开始建立连接的时候，SYN控制位为1，发给接收方，于此同时将序号的初始值告知接收方。
- 数据包长度没有包括在TCP头部中，而是用整个网络包长度 - TCP头部长度，计算获得。
- ![image.png](https://cdn.nlark.com/yuque/0/2022/png/27466443/1669731302460-32337305-e2d5-4484-8ccb-445f4a727168.png#averageHue=%23f5f5f4&clientId=u4fa29f8e-1a7c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=844&id=u34a8c6ad&margin=%5Bobject%20Object%5D&name=image.png&originHeight=844&originWidth=612&originalType=binary&ratio=1&rotation=0&showTitle=false&size=196820&status=done&style=none&taskId=u25ab99b9-d788-4767-b0ea-8114905fcac&title=&width=612)
- 这里的ACK号，并不是控制位的ACKbit，而是接着序号的ACK号。ACK号为接收到的数据字节数目 + 1，换句话说ACK号数目之前的数据，都已经接收到了。
- TCP在获得对方ACK之前，发送过的包都会保存在发送缓冲区中。如果对方没有返回某些包的对应的ACK号，就会重新发送这些包。如果重试几次依然无效，会强制结束通信，并向应用程序报错。
- 正式TCP通过序号和ACK号，保证了可靠通信。
- 网卡，集线器，路由器都没有补偿机制，遇到错误就丢弃相应包。
- TCP数据收发是双向的。只不过双方要分别计算各自的序号和ACK号。
### 2.2 连接服务器
### 2.3 收发数据

