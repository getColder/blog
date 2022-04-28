---
title: PS4内网尝试remote play【折腾】
date: 2022-04-26 17:14:59
tags:
---
## 一、准备工作

**思考Remote Play工作流程：**

PlayStation4主机本身作为服务器向remote客户端发送tcp流，windows平台上客户端RemotePlay接收tcp流。当然也有其他客户端，据说比官方的Remote Play效果更好，后面再看。它们是通过sony的私有协议建立连接的，需要用到PS4生成的一个认证码。

**其建立连接的过程是：**

1. 服务器生成序列码xxx
2. 在客户端输入xxx序列码后配对成功
3. 客户端建立连接后直接投屏显示
4. 通过windows自带设备进行控制

> _**内网穿透面临的问题：**_
>
> * ps4本身不支持ssh，或许要借助PC主机作转发
> * 若用PC主机作转发，是转发客户端数据还是转发请求？
> * 若转发客户端流数据，转发的同时是否必须在转发的PC上播放，造成资源浪费？

**预设方案工具：**

1. ssh反向隧道 +  ps4内置Proxy
2. nginx_stream 模块tcp协议转发

# 二、SSH反向隧道

> SSH建立隧道又两种，一种是本地隧道，第二种隧道是远程隧道，也称反向隧道。
>
> 它的作用是用作端口转发，与端口映射不同，它需要借助”中转站“。而端口映射比较简单，例如NAT和DMZ，只要有公网IP，在路由器设置好就可以直接访问了。但可惜的是，学校内网在一层又一层的路由器之下，所以无法逐步映射上去。因此，建立SSH反向隧道来作内网穿透。

SSH内网穿透有3个实体：

* 目标主机
* 公网主机
* 远程主机

**目标主机**，即我们”困在内网里的主机“，它是最终我们要访问的目标。

**公网主机**，作为跳板，它需要有一个公网IP。

**远程主机，** 最终，我们要通过远程主机访问来访问目标主机。

---

*最后要明确三个实体需要做的工作：*

目标主机作为反向隧道的服务器，借助SSH -R 建立远程服务，这个服务针对某一个ssh端提供（这里就是我们的公网主机），将自己的某个端口A作为最终目标（疑问：能否指定端口范围？）。

```bash
SSH -vNCR 12345:192.168.1.179:8080 root@57.208.222.111 -p 22
```

* **-v**	代表显示打印信息，包括报错、执行结果、转发记录显示等
* -N	隧道只作转发，而不执行任何shell命令。不加的话会很危险吧？
* **-C**	压缩数据，CPU时间换空间=CPU时间换网络传输时间？这么理解对么
* **-R**	建立反向隧道的指令
* **-p** 指定对方（SSH服务器）的端口，默认22
* **反向隧道的参数** : listen-port:host:port
  * listen-port 是对方（公网主机）监听端口，要在公网主机上配置ssh的config监听才行，此处为12345
  * host:port 即目标主机的地址和端口，是我们的访问目标，此处为192.168.1.179:8080
  * 另外，listen-port之前可以指定一个host，从而只允许某个ip访问
  * 所有端口和地址纯属虚构

<div style="margin: auto;width:75%;text-align: center">{% asset_img ssh_target.png ssh_remote %}
<b>目标主机建立远程隧道</b></div>


接下来，在公网主机（云服务器，且称它为主机吧）上测试能否访问内网内的端口。在本地目标主机上我用express创建了个web服务，将web服务监听的端口转发给隧道的监听的端口。然后用curl自测试一下，获取到了html，成功！

<div style="margin: auto;width:75%;text-align: center">{% asset_img traverseSuccess.png traverseSuccess %}
<b>内网穿透成功</b></div>


**但这个时候出了点问题！** 用公网ip访问这个端口却被拒绝，看一下是不是防火墙的原因。

<div style="margin: auto;width:75%;text-align: center">{% asset_img refuse.png refuse %}
<b>用公网IP访问被拒绝</b></div>

一开始很疑惑：明明防火墙运行端口开放了啊，为什么连接不上呢？又是重启SSHD、又是改配置的，终于查明了原因：**原来是公网主机对外开发的端口没有被监听！**  并不是被refused了。

在sshd_config设置两个东西：

* `GateWayPorts = yes`       默认的端口只会监听127.0.0.1，指明为网关端口后，就监听0.0.0.0了
* `AllowTcpForwarding=yes`  允许转发TCP

添加参数后，重新连接ssh，通过公网ip地址可以访问了，内网穿透成功！




<div style="margin: auto;width:30%;text-align: center">{% asset_img finish.png finish %}
<b>4G网络成功访问内网</b></div>


# 三、为客户端转发端口

...回去尝试
