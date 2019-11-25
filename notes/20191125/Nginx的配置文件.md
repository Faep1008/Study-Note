# Nginx的配置文件

## 一、nginx.conf配置文件的主要结构
```shell
......
events
{
  ......
}
http
{
  ......
  server
  {
    ......
  }
  server
  {
    ......
  }
}
```

## 二、Nginx的虚拟主机配置
利用虚拟主机，不用为每个要运行的网站提供一台单独的Nginx服务器或单独运行一组Nginx进程。虚拟主机提供了在同一台服务器、同一组Nginx进程上运行多个网站的功能。  

在Nginx配置文件中，一个最简化的虚拟主机配置如下所示：
```shell
http
{
  server
  {
    listen 80 default;
    server_name  _ *;
    access_log  logs/default.access.log combined;
    location / {
      index index.html;
      root /data0/htdocs/htdocs;
    }
  }
}
```
nginx可以配置多种类型的虚拟主机：一是基于IP的虚拟主机，二是基于域名的虚拟主机，三是基于端口的虚拟主机。

### 2.1、配置基于IP的虚拟主机
Linux等操作系统允许添加IP别名，IP别名背后的概念很简单：可以在一块物理网卡上绑定多个IP地址。这样就能够在使用单一网卡的同一个服务器上运行多个基于IP的虚拟主机。  
设置IP别名也很容易，只需配置系统上的网络接口，让它监听额外的IP地址。  

```html
http
{
  #虚拟主机配置
  server
  {
    #监听的IP和端口
    listen 192.168.8.43:80
    #主机名称
    server_name 192.168.8.43
    #访问日志文件存放路径
    access_log logs/server1.access.log combined;
    location /
    {
      #默认首页文件，顺序从左到右
      index index.html index.htm;
      #html网页文件存放的目录
      root /data0/htdocs/server1;
    }
  }
}
```

从配置文件可以看出，一段server{......}就是一个虚拟主机，如果要配置多个虚拟主机，建立多段server{......}配置即可，非常方便。  监听的IP和端口也可以不写IP地址，只写端口，把它配置成“listen 80”，则表示监听该服务器上所有IP的80端口，可通过`server_name`区分不同的虚拟主机。  

### 2.2、配置基于域名的虚拟主机（推荐）
基于域名的虚拟主机是最常见的一种虚拟主机，只需配置你的DNS服务器，将每个主机名映射到正确的IP地址，然后配置Nginx服务器，令其识别不同的主机名就可以了。这种方式，使很多虚拟主机可以共享同一个IP地址，有效的解决了IP地址不足的问题。所以，在没有特殊要求的情况下，推荐使用基于域名的虚拟主机。  

```html
http
{
  #第一个虚拟主机
  server
  {
    #监听的端口
    listen 80;
    #主机名称
    server_name aaa.domain
    #访问日志文件存放路径
    access_log logs/aaa.domain.com.access.log combined;
    location /
    {
      #默认首页，顺序从左到右
      index index.html index.htm
      #HTML网页文件存放目录
      root /data0/htdocs/aaa.domain.com
    }
  }
}
```

## 四、nginx.conf配置文件的简要说明

```shell
#使用的用户和组  
user www www;  

#指定工作衍生进程数（一般等于CPU的总核数或总核数的两倍）  
worker_processes 8;  

#指定错误日志存放路径，错误日志记录级别可选项为：[debug|info|notice|warn|error|crit]  
error_log /data/log/nginx_error.log crit;  

#指定pid存放的路径  
pid /usr/local/webserver/nginx/nginx.pid;  

#指定文件描述符数量  
worker_rlimit_nofile 51200;  

events  
{  
  #使用的网络I/O模型，Linux系统推荐epoll模型  
  use epoll;  
  #允许的连接数  
  worker_connections 51200;  
}  

http
{
  include  mime.types;
  default_type application/octet-stream;
  #设置使用的字符集，如果一个网站有多种字符集，请不要随便设置，应让程序员在HTML代码中通过Meta标签设置
  #charset gb2312;

  server_names_hash_bucket_size 128;
  client_header_buffer_size 32k;
  large_client_header_buffers 4 32k;

  #设置客户端能够上传文件大小
  client_max_body_size 8m;

  keepalive_timeout 60;

  #虚拟主机配置
  server
  {
    listen 80;
    server_name www.faep.com;
    index  index.html;
  }
}



```
