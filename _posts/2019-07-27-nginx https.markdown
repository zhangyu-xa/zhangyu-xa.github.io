---
layout: post
title: linux下nginx+ssl实现https访问
tags: [nginx, ssl, https, web]
image: '/images/pages/cover2.jpg'
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

#### 四 、 生成自签名SSL证书

由于https需要SSL证书，一般来说，SSL证书需要购买才能真正使用，这里我们只是为了验证可以通过nginx配置来使用https访问网页，因为是本地环境，直接用OpenSSL给自己颁发一个CA根证书用于后面给服务器做CA签署。

```shell
1. 生成CA密钥
openssl genrsa -des3 -out ca.key 2048

2. 生成CA根证书
openssl req -sha256 -new -x509 -days 365 -key ca.key -out ca.crt
以上命令生成时候要填很多东西 一个个看着写吧【最好同第4步的信息保持一致，特别是Common Name，对应-subj的CN字段】
如果怕输入错误，可以在上述命令后面直接添加以下命令，显示指定相关信息
 -subj "/C=CN/ST=GD/L=SZ/O=lee/OU=study/CN=172.16.129.89"
 CN（Common Name）代表要授权签发证书的服务器IP

3. 生成服务器秘钥
openssl genrsa -des3 -out server.key 2048
然后他会要求你输入这个key文件的密码。不推荐输入。因为以后要给nginx使用。每次reload nginx配置时候都要你验证这个PAM密码的。
由于生成时候必须输入密码。你可以通过以下命令删除密码。
openssl rsa -in server.key -out server.key

4.生成服务器证书请求文件
openssl req -new -sha256 -key server.key  -out server.csr
以上命令生成时候要填很多东西 一个个看着写吧【最好同第2步的信息保持一致，特别是Common Name对应-subj的CN字段】
如果怕输入错误，可以在上述命令后面直接添加以下命令，显示指定相关信息
 -subj "/C=CN/ST=GD/L=SZ/O=lee/OU=study/CN=172.16.129.89"
 CN（Common Name）代表要授权签发证书的服务器IP
 
5. CA签署服务器证书
openssl ca -in server.csr -md sha256 -keyfile ca.key -cert ca.crt -out server.crt

```

执行完毕后，会在输出目录里面生成ca.key 、 ca.crt 、 server.key 、 server.crt四个证书文件。

使用时将ca.crt根证书添加到浏览器的“管理证书”中：
![alt](/images/posts/https/06.png)

#### 五 、 配置并启动nginx，用https来访问web

最后一步，我们已经生成了自己的sll证书，接下来就可以配置nginx.conf:

```nginx
server {
    listen       443 ssl;

    ssl_certificate server.crt; #根据具体的路径配置即可
    ssl_certificate_key server.key; #根据具体的路径配置即可

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

但是结果并不是我们想的那样，在chrome30版本下，我的网页顺利打开，也被标记为安全了：
![alt](/images/posts/https/03.png)

但是在最新的chrome74版本下，我们得到的是另外一种结果：
![alt](/images/posts/https/04.png)

我们的网页仍然被标记为不安全，并在打开前给出了安全提示，如果我们忽略，可以点击高级->继续访问，也能打开网页，但这样并不是我们想要的。

> 原因在于：Chrome58以后对https的证书认证较为严格，证书里必须带有正确的Common Name，也就是必须有DNS Name=ajax.googleapis.com， IP Address=127.0.0.1这样的信息，浏览器才认为真正安全。

打开cosole中Security选项，我们看到一些错误提示：缺少了Subject Alternative Name扩展
![alt](/images/posts/https/05.png)

接下来我们需要改造生成证书的脚本，主要是针对第4、5两项，添加SAN扩展：
```shell

4.生成服务器证书请求文件
openssl req -new \
    -sha256 \
    -key server.key \
    -subj "/C=CN/ST=GD/L=SZ/O=lee/OU=study/CN=172.16.129.89" \
    -reqexts SAN \
    -config <(cat /etc/pki/tls/openssl.cnf \
        <(printf "[SAN]\nsubjectAltName=IP:172.16.129.89")) \
    -out server.csr

5. CA签署服务器证书
openssl ca -in server.csr \
        -md sha256 \
        -keyfile ca.key \
    -cert ca.crt \
    -extensions SAN \
    -config <(cat /etc/pki/tls/openssl.cnf \
        <(printf "[SAN]\nsubjectAltName=IP:172.16.129.89")) \
    -out server.crt

其中/etc/pki/tls/openssl.cnf是openssl的配置文件地址，路径根据具体位置而定；
SAN扩展这里我们用的是IP地址，如果用的DNS，上述对应部分修改为"[SAN]\nsubjectAltName=DNS:172.16.129.89"，用“，”号隔开，可以DNS和IP同时使用。
注意：IP指定的地址一定要和CN（Common Name）指定的相同。
```

接下来重启nginx服务器，清楚浏览器缓存，打开chrome74版本，访问我们的网页https://172.16.129.89：
![alt](/images/posts/https/07.png)

这样，我们整个针对https的调试工作就完成了。