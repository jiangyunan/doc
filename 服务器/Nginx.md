## 配置

### 全局变量

```nginx
$args 此变量与请求行中的参数相等
$content_length 等于请求行的“Content_Length”的值。
$content_type 等同与请求头部的”Content_Type”的值
$document_root 等同于当前请求的root指令指定的值
$document_uri 与$uri一样
$host 与请求头部中“Host”行指定的值或是request到达的server的名字（没有Host行）一样
$limit_rate 允许限制的连接速率
$request_method 等同于request的method，通常是“GET”或“POST”
$remote_addr 客户端ip
$remote_port 客户端port
$remote_user 等同于用户名，由ngx_http_auth_basic_module认证
$request_filename 当前请求的文件的路径名，由root或alias和URI request组合而成
$request_body_file
$request_uri 含有参数的完整的初始URI
$query_string 与$args一样
$server_protocol 等同于request的协议，使用“HTTP/1.0”或“HTTP/1.1”
$server_addr request到达的server的ip，一般获得此变量的值的目的是进行系统调用。为了避免系统调用，有必要在listen指令中指明ip，并使用bind参数。
$server_name 请求到达的服务器名
$server_port 请求到达的服务器的端口号
$uri 等同于当前request中的URI，可不同于初始值，例如内部重定向时或使用index
```



## upstream 配置详解

### 轮询（默认）

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

### weight

指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。
 例如：



```undefined
upstream bakend {
    server 192.168.159.10 weight=10;
    server 192.168.159.11 weight=10;
}
```

### ip_hash

每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。
 例如：



```undefined
upstream resinserver{
    ip_hash;
    server 192.168.159.10:8080;
    server 192.168.159.11:8080;
}
```

### 4、fair（第三方）

按后端服务器的响应时间来分配请求，响应时间短的优先分配。



```undefined
upstream resinserver{
    server server1;
    server server2;
    fair;
}
```

### 5、url_hash（第三方）

按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。

例：在upstream中加入hash语句，server语句中不能写入weight等其他的参数，hash_method是使用的hash算法



```bash
upstream resinserver{
    server squid1:3128;
    server squid2:3128;
    hash $request_uri;
    hash_method crc32;
}
```

### 定义负载均衡设备的Ip及设备状态



```undefined
upstream resinserver{
    ip_hash;
    server 127.0.0.1:8000 down;
    server 127.0.0.1:8080 weight=2;
    server 127.0.0.1:6801;
    server 127.0.0.1:6802 backup;
}
```

### 每个设备的状态设置为:



```css
1. down 表示单前的server暂时不参与负载
2. weight 默认为1.weight越大，负载的权重就越大。
3. max_fails：允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream 模块定义的错误
4. fail_timeout:max_fails次失败后，暂停的时间。
5. backup： 其它所有的非backup机器down或者忙的时候，请求backup机器。所以这台机器压力会最轻。
```

------

------

## location 配置详解

> 语法规则： location [=||*|^~] /uri/ { … }



```cpp
= 开头表示精确匹配

^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。

~ 开头表示区分大小写的正则匹配

~* 开头表示不区分大小写的正则匹配

!~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则

/ 通用匹配，任何请求都会匹配到。

多个location配置的情况下匹配顺序为（参考资料而来，还未实际验证，试试就知道了，不必拘泥，仅供参考）：

首先匹配 =，其次匹配^~, 其次是按文件中顺序的正则匹配，最后是交给 / 通用匹配。当有匹配成功时候，停止匹配，按当前匹配规则处理请求。
```

### 例子，有如下匹配规则：



```csharp
location = / {
   #规则A
}
location = /login {
   #规则B
}
location ^~ /static/ {
   #规则C
}
location ~ \.(gif|jpg|png|js|css)$ {
   #规则D
}
location ~* \.png$ {
   #规则E
}
location !~ \.xhtml$ {
   #规则F
}
location !~* \.xhtml$ {
   #规则G
}
location / {
   #规则H
}
```

那么产生的效果如下：



```php
访问根目录/， 比如http://localhost/ 将匹配规则A   

访问 http://localhost/login 将匹配规则B，http://localhost/register 则匹配规则H   

访问 http://localhost/static/a.html 将匹配规则C   

访问 http://localhost/a.gif, http://localhost/b.jpg 将匹配规则D和规则E，但是规则D顺序优先，规则E不起作用， 而 http://localhost/static/c.png 则优先匹配到 规则C   

访问 http://localhost/a.PNG 则匹配规则E， 而不会匹配规则D，因为规则E不区分大小写。   

访问 http://localhost/a.xhtml 不会匹配规则F和规则G，http://localhost/a.XHTML不会匹配规则G，因为不区分大小写。规则F，规则G属于排除法，符合匹配规则但是不会匹配到，所以想想看实际应用中哪里会用到。    

访问 http://localhost/category/id/1111 则最终匹配到规则H，因为以上规则都不匹配，这个时候应该是nginx转发请求给后端应用服务器，比如FastCGI（php），tomcat（jsp），nginx作为方向代理服务器存在。    
```

### 所以实际使用中，个人觉得至少有三个匹配规则定义，如下：



```csharp
#直接匹配网站根，通过域名访问网站首页比较频繁，使用这个会加速处理，官网如是说。
#这里是直接转发给后端应用服务器了，也可以是一个静态首页
# 第一个必选规则
location = / {
    proxy_pass http://tomcat:8080/index
}

# 第二个必选规则是处理静态文件请求，这是nginx作为http服务器的强项
# 有两种配置模式，目录匹配或后缀匹配,任选其一或搭配使用
location ^~ /static/ {
    root /webroot/static/;
}
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    root /webroot/res/;
}

#第三个规则就是通用规则，用来转发动态请求到后端应用服务器
#非静态文件请求就默认是动态请求，自己根据实际把握
#毕竟目前的一些框架的流行，带.php,.jsp后缀的情况很少了

location / {
    proxy_pass http://tomcat:8080/
}
```

------

### 三、ReWrite语法

last – 基本上都用这个Flag。
 break – 中止Rewirte，不在继续匹配
 redirect – 返回临时重定向的HTTP状态302
 permanent – 返回永久重定向的HTTP状态301

#### 1、下面是可以用来判断的表达式：



```undefined
-f和!-f用来判断是否存在文件
-d和!-d用来判断是否存在目录
-e和!-e用来判断是否存在文件或目录
-x和!-x用来判断文件是否可执行
```

#### 2、下面是可以用作判断的全局变量

> 例：[http://localhost:88/test1/test2/test.php](https://link.jianshu.com?t=http://localhost:88/test1/test2/test.php)



```php
$host：localhost
$server_port：88
$request_uri：http://localhost:88/test1/test2/test.php
$document_uri：/test1/test2/test.php
$document_root：D:\nginx/html
$request_filename：D:\nginx/html/test1/test2/test.php
```

------

## 四、Redirect语法



```bash
server {
    listen 80;
    server_name start.igrow.cn;
    index index.html index.php;
    root html;
    if ($http_host !~ "^star\.igrow\.cn$" {
        rewrite ^(.*) http://star.igrow.cn$1 redirect;
    }
}
```

------

## 五、防盗链



```ruby
location ~* \.(gif|jpg|swf)$ {
    valid_referers none blocked start.igrow.cn sta.igrow.cn;
    if ($invalid_referer) {
        rewrite ^/ http://$host/logo.png;
    }
}
```

------

## 六、根据文件类型设置过期时间



```ruby
location ~* \.(js|css|jpg|jpeg|gif|png|swf)$ {
    if (-f $request_filename) {
        expires 1h;
        break;
    }
}
```

------

## 七、禁止访问某个目录



```bash
location ~* \.(txt|doc)${
    root /data/www/wwwroot/linuxtone/test;
    deny all;
}
```

