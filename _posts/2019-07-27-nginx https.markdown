---
layout: post
title: linux下nginx+ssl实现https访问
tags: [nginx, ssl, https, web]
---

目前只尝试了在linux下进行实现。

#### 一 、 环境准备

要编译nginx源码和生成ssl证书，需要linux环境下安装依赖模块：
```markdown
yum -y install gcc gcc-c++ automake pcre pcre-devel zlip zlib-devel openssl openssl-devel 
```

#### 二 、 下载nginx源码，编译nginx并添加ssl模块

目前nginx默认版本中没有http_ssl_module模块，所以需要在[nginx官网](http://nginx.org/en/download.html)下载最新的nginx源码，然后编译nginx并添加SSL模块。

这里我选择最新的稳定版本：nginx-1.16.0，下载源码包nginx-1.16.0.tar.gz；然后将下载的nginx源码拷贝到linux服务器上一个自定义目录中，进入该目录，然后解压nginx源码`sudo tar -xzvf nginx-1.16.0.tar.gz`。
进入解压后的目录`cd nginx-1.16.0`
- 执行以下命令进行nginx源码编译配置：
```markdown
./configure --prefix=./nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre
```
注意：--prefix配置的是nginx的安装目录，这里配置安装到当前目录，--with配置是编译nginx时额外添加的模块，这里主要添加了http_ssl_module，其他模块可以选择安装。

- 执行以下命令编译并安装nginx
```markdown
make && make install
```
安装完成后，在--prefix指定的目录下会有刚编译好的nginx文件夹，进入文件夹，目录结构如下：
```
nginx
├── conf
├── html
├── logs
└── sbin
```
看到编译后的目录结构，正是我们日常使用的nginx，到此nginx就安装成功。

#### 三 、 配置并启动nginx，用http来访问web
将准备好的网页拷贝到linux服务器中，配置conf/nginx.conf：
```nginx
server {
    listen       89;
    server_name  localhost;

    root /home/web/https/site/index.html;
}

```

启动nginx，通过127.0.0.1:89访问刚部署好的网页：

![alt](/images/posts/https/01.png)

由于nginx上没有配置SSL证书相关的内容，网页的访问仍然是http1.0的协议，高版本浏览器会标记该网页不安全。
如果我们强行使用https访问，则得到如下结果：

![alt](/images/posts/https/02.png)

#### 四 、 自定义生成SSL证书

由于https需要SSL证书，一般来说，SSL证书需要购买才能真正使用，这里我们只是为了验证可以通过nginx配置来使用https访问网页，所以只需要生成一个自定义证书就行。

```shell
首先执行如下命令生成一个key
openssl genrsa -des3 -out ssl.key 1024
然后他会要求你输入这个key文件的密码。不推荐输入。因为以后要给nginx使用。每次reload nginx配置时候都要你验证这个PAM密码的。
由于生成时候必须输入密码。你可以输入后 再删掉。

mv ssl.key xxx.key
openssl rsa -in xxx.key -out ssl.key
rm xxx.key
然后根据这个key文件生成证书请求文件
openssl req -new -key ssl.key -out ssl.csr
以上命令生成时候要填很多东西 一个个看着写吧（可以随便，毕竟这是自己生成的证书）

最后根据这2个文件生成crt证书文件
openssl x509 -req -days 365 -in ssl.csr -signkey ssl.key -out ssl.crt
这里365是证书有效期 推荐3650, 这个大家随意, 最后使用到的文件是key和crt文件。

如果需要用pfx 可以用以下命令生成
openssl pkcs12 -export -inkey ssl.key -in ssl.crt -out ssl.pfx

```

执行完毕后，会在输出目录里面生成ssl.csr 、 ssl.crt 、 ssl.key 、 ssl.pfx四个证书文件。

#### 五 、 配置并启动nginx，用https来访问web

最后一步，我们已经生成了自己的sll证书，接下来就可以配置nginx.conf:

```nginx
server {
    listen       443 ssl;

    ssl_certificate /home/web_zhangyu/https/ssl.crt;
    ssl_certificate_key /home/web_zhangyu/https/ssl.key;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout 5m;

    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        root /home/web/https/site/;
        index  index.html index.htm;
    }
}

```

其中443是https的默认端口（http的默认端口是80），现在，你可以通过https://172.16.129.89来访问你的网页了。