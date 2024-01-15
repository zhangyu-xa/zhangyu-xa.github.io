---
layout: post
title: 利用 Verdaccio 搭建 Npm 包管理私服
tags: [verdiaccio, npm, bin]
image: '/images/pages/cover5.jpg'
---

之前开发项目的过程中，多多少少都会沉淀积累一些组件，后来队伍越来越大，项目越来越多，通过拷贝粘贴的方式干重复的事情总感觉太low，想着能不能通过高雅的方式来解决这些重复工作，于是就想到了自己来搭建一个`npm`私服试试，这里我选的是`Verdaccio`。

> ##### 什么是 Verdaccio
>
> “一个基于 Node.js 的轻量级私有仓库”。
>  平时使用 npm publish 进行发布时，上传的仓库默认地址是 npm，通过 Verdaccio 工具在本地新建一个仓库地址，再把本地的默认上传仓库地址切换到本地仓库地址即可。当 npm install 时没有找到本地的仓库，则 Verdaccio 默认配置中会从 npm 中央仓库下载。

运行环境的`nodejs`和`npm`版本如下：

```shell
C:> node -v
v16.20.2
C:> npm -v
8.19.4
```

### 一、安装和配置

话不多说，接下来我们就直接安装 `verdaccio`：

```shell
npm install -g verdaccio
```

默认安装在 `/usr/local/bin/verdaccio`，相关的配置文件在 `/etc/verdaccio/config.yaml`，这里我们看看配置文件的内容，这里只挑几个重要的说说（其他的后续用到再完善）。

######  storage: ./storage

```yaml
storage: ./storage
```

设置托管或缓存包的存放目录 ，即所有的请求过的包均被缓存在这个目录下；

######  auth 

```yaml
auth:
  htpasswd:
     file: ./htpasswd
     max_users: 10
```

权限控制，`file`指定的是用户存储的文件及路径，通过 `npm adduser`添加的用户均被添加到此文件中：

![alt](/images/posts/verdaccio/1.png)

如果要删除用户，直接从这里删除也是可以的。

######  uplinks 

```yaml
uplinks:
  mirror:
    url: https://registry.npmmirror.com/
  taobao:
    url: https://registry.npm.taobao.org/
  npmjs:
    url: https://registry.npmjs.org/
```

 设置外部仓储，如果 verdaccio 找不到请求的包（非 verdaccio 托管），就会查找外部仓储。常见格式的有 ：

> - {name}: 外部仓储名称
> 	- url: 访问路径
> 	- timeout: 超时
> 	- maxage: 默认值 2m，2m 钟内不会就同样的请求访问外部仓储
> 	- fail_timeout: 如果外部访问失败，在多长时间内不回重试
> 	- headers: 添加自定义 http 头当外部仓储访问请求中，例如`authorization: "Basic YourBase64EncodedCredentials=="`
> 	- cache: 是否启用缓存，默认启用。
>
> 更多[uplinks]( [Verdaccio Uplinks 文档](https://link.juejin.cn/?target=https%3A%2F%2Fverdaccio.org%2Fdocs%2Fen%2Fuplinks) )配置

###### publish

```yaml
publish:
  allow_offline: true
```

设置允许离线发包，因为verdaccio是在线检查包的合法性，在很多局域网内容使用时，允许离线发包很有用，能解决发布时检测失败的问题（503错误）。

二、启动和管理

启动 `verdaccio`有三种方案，

第一种是直接启动，关闭窗口即停止：

```shell
/usr/local/bin/verdaccio --config /etc/verdaccio/config.yaml
```

第二种是静默启动：

```shell
nohup /usr/local/bin/verdaccio --config /etc/verdaccio/config.yaml >/dev/null 2>log &
```

将这种方式包装到一个`shell`脚本中：

```shell
#!/bin/bash

ip_addr=$(ip addr | grep eth0 | grep inet | awk '{print $2}' | cut -d / -f 1)

echo 'vedaccio 启动成功，server as http://'$ip_addr

nohup /usr/local/bin/verdaccio --config /etc/verdaccio/config.yaml >/dev/null 2>log &

exit 0
```

第三种是通过`pm2`来管理`verdaccio`进程：

```shell
npm install -g pm2
#查看状态
pm2 status verdaccio
#启动
pm2 start verdaccio
#停止
pm2 stop verdaccio
#重启
pm2 restart verdaccio
```

启动后，可以通过 `http://127.0.0.1:4873`访问页面：

![alt](/images/posts/verdaccio/2.png)

三、发包管理

发包管理主要包含注册、登录、发布、管理包几个部分，和 公共`npm`包管理器一样

```shell
#注册
npm adduser -registry=http://127.0.0.1:4873
#登录
npm login -registry=http://127.0.0.1:4873
#发布
npm publish -registry=http://127.0.0.1:4873
#撤销发布
npm unpublish [packageName]@[version] -registry=http://127.0.0.1:4873
```

撤销发布时，可以指定版本，也可以全部撤销发布，如 `npm unpublish [packageName]@* `。

常见的包管理及信息查询命令如下：

| `npm view [packageName]`          | 查看包的详细信息                                             |
| --------------------------------- | ------------------------------------------------------------ |
| `npm view [packageName] versions` | 查看所有版本信息                                             |
| `npm view [packageName] version`  | 查看当前版本信息                                             |
| npm ls [packageName] [-g]         | 查看包当前安装的版本<br />`-g`标示全局查看<br />此命令需要在某个具体的项目目录下查看 |

发布后，可以通过 `http://127.0.0.1:4873`访问页面：

![alt](/images/posts/verdaccio/3.png)