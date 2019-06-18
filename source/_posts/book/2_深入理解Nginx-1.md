---
title: 深入理解Nginx阅读笔记 --- 壹 · 初识Nginx
date: 2018-11-21 13:24:06
tags:
- nginx
categories:
- 读书笔记
---

## 第一章 Nginx安装

### 1.1 Nginx简介

#### 1. 应用场景

- 静态资源服务器 —— 通过本地文件系统提供服务
- 反向代理服务器 —— 负载均衡与缓存
- API服务 —— OpenResty

#### 2.  优点

- 高并发 高性能
- 可扩展性
- 高可靠性
- 热部署
- BSD许可证

#### 3 版本

Nginx可以通过开源的 http://nginx.org 获取，单数为当前主线版本如1.15.0，单数版本有新功能但存在一些不稳定的bug，双数为当前稳定版本如1.14.0。Nginx提供商业版 http://nginx.com 。

OpenResty是章亦春开发的一款基于 NGINX 和 LuaJIT 的 Web 平台，用来做API服务很合适，同样提供开源 http://openresty.org 和商业版本 http://openresty.com 。

### 1.2 Nginx准备环境

- Linux操作系统

  Nginx采用了epoll事件模型，所以需要Linux 2.6以上内核才会支持epoll。

- Nginx必备库

  1. PCRE库 —— 用于支持正则表达式

     ```
     yum install pcre pcre-devel
     ```

  2. zlib库 —— 用于支持gzip压缩

     ```
     yum install zlib zlib-devel
     ```

  3. OpenSSL开发库 —— 用于支持https

     ```
     yum install openssl openssl-devel
     ```

- Linux内核参数优化

  可以根据不同应用场景修改`/etc/sysctl.conf`优化内核参数来提高无服务器性能，如下所示：

  ```bash
  fs.file-max = 999999
  net.ipv4.tcp_tw_reuse = 1
  net.ipv4.tcp_keepalive_time = 600
  net.ipv4.tcp_fin_timeout = 30
  net.ipv4.tcp_max_tw_buckets = 5000
  net.ipv4.ip_local_port_range = 1024 61000
  net.ipv4.tcp_rmem = 4096 32768 262142
  net.ipv4.tcp_wmem = 4096 32768 262142
  net.core.netdev_max_backlog = 8096
  net.core.rmem_default = 262144
  net.core.wmem_default = 262144
  net.core.rmem_max = 2097152
  net.core.wmem_max = 2097152
  net.ipv4.tcp_syncookies = 1
  net.ipv4.tcp_max_syn.backlog = 1024
  
  # 执行sysctl -p 可使上述修改生效。
  ```


| 字段                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| file-max            | 系统级别所有进程可以同时打开的文件数，ulimit中的则是进程能够打开的文件数 |
| tcp_tw_reuse        | 设置为1，表示允许将TIME_WAIT状态的socket重新用于新的TCP连接  |
| tcp_keepalive_time  | 当keepalive启用时，TCP发送keepalive消息的频度，默认2小时     |
| tcp_fin_timeout     | 当服务器主动关闭连接时，socket保持在FIN_WAIT_2状态的最大时间 |
| tcp_max_tw_buckets  | 操作系统允许TIME_WAIT状态socket数量的最大值，若超过该数值，TIME_WAIT的socket将立刻被清除并打印警告信息，默认180000 |
| tcp_max_syn_backlog | TCP建立三次握手阶段收到SYN请求队列的最大长度，默认1024       |
| ip_local_port_range | 定义UPD和TCP连接中本地端口的取值范围                         |
| net.ipv4.tcp_rmem   | TCP接收缓存(用于TCP接收滑动窗口)的最小值、默认值和最大值     |
| net.ipv4.tcp_wmem   | TCP发送缓存(用于TCP发送滑动窗口)的最小值、默认值和最大值     |
| netdev_max_backlog  | 当网卡接收数据包的速度大于内核处理速度时，会有一个队列保存这些数据包，该参数表示这个队列的最大值 |
| rmem_default        | 内核套接字接收缓存区默认大小                                 |
| wmem_default        | 内核套接字发送缓存区默认大小                                 |
| rmem_max            | 内核套接字接收缓存区最大值                                   |
| wmem_max            | 内核套接字发送缓存区最大值                                   |
| tcp_syncookies      | 用于解决TCP的SYN攻击                                         |

  > 注意：滑动窗口的大小与套接字缓存区会在一定程度上影响并发连接的数量，每个TCP连接都会为维护TCP滑动窗口而消耗内存，这个窗口会根据服务器处理速度收缩或扩展。

### 1.3 Nginx源码安装

1. 获取Nginx源码

   ```
   wget http://nginx.org/download/nginx-1.14.1.tar.gz
   ```

2. 解压

   ```
   tar -xzf nginx-1.14.1.tar.gz
   ```

   > contrib/vim目录下的文件拷贝到本地vim可以获得nginx配置文件的语法配色

3. 编译安装

   ```bash
   ./configure  	# 通过./configure 可以定制化nginx模块
   make
   make install
   ```

### 1.4 configure详解

通过`./configure --help`可以查看可配置的选项，`--with_xxx_module` 表示开启该模块（默认为关闭），`--without_xxx_module`打头的表示关闭该模块（默认为开启），具体参数的信息可通过 http://nginx.org/en/docs/configure.html 进行查看。

当configure执行成功会生成objs目录，而其中`ngx_modules.c`则定义了配置的模块，其优先级也是根据`ngx_modules`数组中顺序决定。

```c
ngx_module_t *ngx_modules[] = {
    &ngx_core_module,
    &ngx_errlog_module,
    &ngx_conf_module,
    &ngx_regex_module,
    &ngx_events_module,
    &ngx_event_core_module,
    &ngx_epoll_module,
    &ngx_http_module,
    &ngx_http_core_module,
    &ngx_http_log_module,
    &ngx_http_upstream_module,
    &ngx_http_static_module,
    &ngx_http_autoindex_module,
    &ngx_http_index_module,
    &ngx_http_mirror_module,
    &ngx_http_try_files_module,
    // .....
    NULL
};
```

### 1.5 Nginx命令

1. 默认启动方式

   ```bash
   # 这种方式启动，默认读取相同安装目录下的配置文件：/usr/local/nginx/conf/nginx.conf
   /usr/local/nginx/sbin/nginx
   ```

2. 指定配置文件启动

   ```bash
   /usr/local/nginx/sbin/nginx -c /tmp/nginx.conf
   ```

3. 指定安装目录启动

   ```bash
   /usr/local/nginx/sbin/nginx -p /usr/local/nginx
   ```

4. 指定全局配置启动

   通过-g参数临时指定一些全局配置项，但约束条件是不能与默认配置文件nginx.conf的配置项冲突

   ```bash
   /usr/local/nginx/sbin/nginx -g "pid /var/nginx/test.pid;"
   ```

5. 测试配置信息是否有错

   ```bash
   /usr/local/nginx/sbin/nginx -t
   ```

6. 显示版本信息

   ```bash
   # -V 可以显示编译阶段的信息
   /usr/local/nginx/sbin/nginx -v
   ```

7. 快速停止服务

   ```bash
   # -s参数是向nginx服务发送信号，nginx通过nginx.pid文件得到pid，通过kill发送给进程
   /usr/local/nginx/sbin/nginx -s stop
   # kill -s SIGTERM/SIGINT <nginx master pid> 效果相同
   ```

8. 优雅的停止服务

   ```bash
   /usr/local/nginx/sbin/nginx -s quit
   # kill -s SIGQUIT <nginx master pid> 效果相同
   ```

   优雅的停止服务首先会关闭监听端口，停止接收新的连接，然后把已经建立的连接处理完毕后再退出进程。

9. 重新加载配置文件

   ```bash
   /usr/local/nginx/sbin/nginx -s reload
   # kill -s SIGHUP <nginx master pid> 效果相同
   ```

   使用该参数可以使运行中的Nginx服务重新加载nginx.conf文件，而且不中断服务。接收到该信号Nginx会先检查配置文件是否有错，如果正确，那么就以优雅的方式关闭，然后再重新启动。

10. 日志文件回滚

    ```bash
    /usr/local/nginx/sbin/nginx -s reopen
    # kill -s SIGUSR1 <nginx master pid> 效果相同
    ```

    该参数主要用于切分日志文件使用，让日志文件不至于过大。

11. 平滑升级Nginx

    当Nginx升级到新版本时，可使用平滑升级方式进行。

    1）通知正在运行的旧版Nginx准备升级，可发送`kill -s SIGUSR2 <nginx master pid>`，然后Nginx会将pid文件重命名为nginx.pid.oldbin。

    2）启动新版本的Nginx，通过`ps -ef | grep nginx`可看到新旧版本Nginx进程共存。

    3）通过kill命令向旧版master进程发送SIGQUIT信号，以优雅方式关闭旧版本，平滑升级成功。



## 第二章 Nginx配置

在生产环境下，部署Nginx都是使用一个master进程管理多个worker进程，一般来说，worker进程数等于CPU核心数，每个worker进程提供真正的网络服务，而master进程则用于监控worker进程，为管理员提供命令行服务，包括启动服务、停止服务、重新加载配置文件、平滑升级等。master进程需要较大权限所以通常使用root用户启动，当任意一个worker进程挂掉时，master会立即重启一个新的worker进程。

### 2.1 Nginx配置语法

```nginx
# 使用'#'打头表示注释
# user nobody;
worker_processes 1;

# 配置块由 配置名加一对大括号组成，如下events配置块
events {
    # ...
}

http {
    upstream backend {
        server 127.0.0.1:8080;
    }
    
    # 配置块可以进行嵌套，内层块会继承外层块配置
    gzip on;
    # K表示千字节，还有M表示兆字节
    gzip_buffers 4 8k;
    client_max_body_size 64M;
    access_log logs/access.log;
    
    # 表示时间的单位，ms(毫秒), s(秒), m(分钟), h(小时), d(天)
    # w(周), M(月), y(年)
    expires 10y;
    proxy_read_timeout 600;
    client_body_timeout 2m;
    
    # 使用'$'可以引用Nginx的变量，但只有少数模块支持
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    server {
        # ...
        location /webstatic {
            # 继承外层块配置，但出现处突时，还是使用内层块的配置
            access_log logs/static/access.log;
            gzip off;
        }
    }
}
```

### 2.2 Nginx服务基本配置

Nginx运行时，至少必须加载几个核心模块和一个事件模块，这些模块运行所支持的配置项称为基本配置。基本配置项可分为4大类，分别是用于调试、定位问题的配置项，正常运行必备的配置项，优化性能的配置项和事件类配置项。

#### 1. 用于调试和定位问题的配置项

- 是否以守护进程方式运行nginx

  ```nginx
  daemon on|off;
  # 默认 daemon on;
  ```

- 是否用master/worker工作方式

  ```nginx
  master_process on|off;
  # 默认 master_process on;
  ```

- error日志的设置

  ```nginx
  error_log /path/file level;
  # 默认 error_log logs/error.log error;
  ```

  level是日志输出级别，取值范围是debug, info, notice, warn, error, crit, alert, emerg，日志级别依次增大，设置级别后小于设置的不会输出，而大于设置级别的会输出。

- 是否处理几个特殊的调试点

  ```nginx
  debug_points[stop|abort]
  ```

  nginx在一些关键错误逻辑处设置了调试点，如设置为stop，那么当nginx的代码执行到这些调试点就会发出SIGSTOP信号，如果设置为abort，则会产生一个coredump文件，通过gdb可查看具体信息。

- 仅对指定的客户端输出debug级别的日志

  ```nginx
  debug_connection[IP|CIDR]
  ```

  属于事件类配置，必须放在events {...}中，它的值可以是IP或CIDR这样仅来自指定IP或CIDR请求才会输出debug级别日志。

- 限制coredump核心转储文件的大小

  ```nginx
  worker_rlimit_core size;
  ```

  Linux系统中，进程发生错误或收到信号而终止时，系统会将进程执行时的内存内容写入一个文件，以便调试时使用，而一个coredump文件可能有好几GB，几次dump就可能吧磁盘占满，所以使用该参数加以限定。

- 指定coredump文件生成路径

  ```nginx
  working_directory path;
  ```

#### 2. 正常运行的配置项

- 定义环境变量

  ```nginx
  env TESTPATH=/tmp/;
  ```

  可以让用户直接设置操作系统上的环境变量。

- 嵌入其他配置文件

  ```nginx
  include mime.types;
  include vhost/*.conf;
  ```

  include配置项可以将其他配置文件嵌入到当前配置中，参数可以是绝对路径也可以是相对路径。

- pid文件路径

  ```nginx
  pid logs/nginx.pid;
  ```

  保存master进程ID的pid文件存放路径。

- worker进程运行的用户及用户组

  ```nginx
  user username [groupname];
  # 默认 user nobody;
  ```

  user用于设置master进程启动后，fork出worker进程运行在哪个用户和用户组下。

- worker进程可打开最大描述符个数

  ```nginx
  worker_rlimit_nofile limit;
  ```

- 限制信号队列

  ```nginx
  worker_rlimit_sigpending limit;
  ```

  设置每个用户发送给Nginx信号的队列大小，当该用户的信号队列满了，该用户发送的信号会被忽略。

#### 3. 优化新能的配置项

- worker进程个数

  ```nginx
  worker_processes number;
  # 默认 worker_processes 1;
  ```

  每个worker进程都是单线程进程，它们调用各个模块以实现各种功能，如果这些模块没有阻塞式调用，那么配置的worker数量应该和CPU核数相等，但若可能出现阻塞式调用，那么则应该多配置一些worker进程。

  多worker进程可以充分利用多核系统架构，若worker进程多于CPU核数，则会增大进程间切换带来的消耗，一般情况下worker数量与CPU核数相等。

- 绑定worker进程到指定的CPU内核

  ```nginx
  worker_cpu_affinity cpumask [cpumask ...]
  ```

  使用该配置让worker进程绑定指定CPU，那么每个worker独享CPU，在内核调度上就实现了完全的并发，也不会存在进程切换而带来无畏的消耗。

  > 该配置仅对Linux操作系统有效，Linux使用`sched_setaffinity()`系统调用实现该功能。

- SSL硬件加速

  ```nginx
  ssl_engine device;
  ```

  如果服务器上有SSL硬件加速设备，可通过该配置提高处理速度，通过`openssl engine -t`可查看是否支持

- 系统调用`gettimeofday`的执行频率

  ```nginx
  timer_resolution t;
  ```

  默认情况下，每次内核的事件调用(epoll, select, poll, kqueue等)返回时，都会执行一次`gettimeofday`用内核时钟更新Nginx缓存时钟，早期Linux该操作代价不小，但目前大多架构中，代价可以忽略。如果希望日志文件中时间更加准确可以使用该配置。

- worker进程优先级设置

  ```nginx
  worker_priority nice;
  # 默认 worker_priority 0;
  ```

  设置worker进程的nice优先级，可以提高nginx占用的系统资源。但不建议不内核进程nice值小。

#### 4. 事件类配置项

- 是否打开accept锁

  ```nginx
  accept_mutex on|off;
  # 默认 accept_mutex on;
  ```

  `accept_mutex`是Nginx的负载均衡锁，可以让多个worker进程轮流的、序列化地与客户端建立TCP连接，当一个worker进程建立连接数达到`worker_connections`的7/8时，会大大减少该worker建立连接的机会。

- lock文件的路径

  ```nginx
  lock_file /pata/file;
  # 默认 lock_file logs/nginx.lock;
  ```

  由于编译程序、操作系统等原因导致Nginx不支持原子锁，这时会用文件锁实现accept锁。

- 使用accept锁后到真正建立连接之间的延迟时间

  ```nginx
  accept_mutex_delay Nms;
  # 默认 accept_mutex_delay 500ms;
  ```

  使用accept锁后，同一时间只有一个worker进程能够获取accept锁，如果另一个worker进程尝试获取未成功，那么`accept_mutex_delay`时间后才能再次尝试。

- 批量建立新连接

  ```nginx
  multi_accept on|off;
  # 默认 multi_accept off;
  ```

  当事件模型通知有新连接时，尽可能对本次所以TCP请求都建立连接。

- 选择事件模型

  ```nginx
  use kqueue|epoll|select|eventpoll;
  # Nginx自动选择最合适的事件模型
  ```

- 每个worker同时处理的最大连接数

  ```nginx
  worker_connections number;
  ```

### 2.3 搭建Web服务器

#### 1. 虚拟主机与请求分发

- 监听端口

  ```nginx
  listen address;
  # 默认 listen 80；
  # 配置块 server
  ```



| 可选参数       | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| default        | 将所在的server块作为整个web服务器的默认块，若无该参数，则nginx选择第一个server块作为默认块 |
| default_server | 作用同上                                                     |
| backlog=num    | 表示TCP中backlog队列的大小，默认-1，表示不设置。TCP建立三次握手过程中，进程未处理时，会将连接放入backlog队列，如果队列已满，则客户端建立连接失败 |
| rcvbuf=size    | 设置socket接收缓冲区大小，即设置SO_RCVBUF参数                |
| sndbuf=size    | 设置socket发送缓冲区大小，即设置SO_SNDBUF参数                |
| accept_filter  | 设置accept过滤器，仅对FreeBSD有用                            |
| deferred       | 建立三次握手后不会立即唤起worker进程，只有用户发送请求数据时才唤起worker |
| bind           | 绑定当前端口/地址对，只有同时对一个端口监听多个地址时才生效  |
| ssl            | 在当前监听的端口上建立的连接必须基于SSL协议                  |

- 主机名称

  ```nginx
  server_name name [...]
  # 默认 server_name "";
  # 配置块 server
  ```

  server_name后面可以跟多个主机名称，在处理HTTP请求时，Nginx会获取header中的Host，通过Host与server_name的匹配来决定那个server块来处理该请求。若有多个匹配那么有以下优先级：

  - 首先选择完全匹配，如：`www.nginx.com`

  - 其次选择后缀匹配，如：`*.nginx.com`

  - 再其次选择前缀匹配，如：`www.nginx.*`

  - 最后选择正则表达式匹配，如：`~^\.nginx\.com$`

- `server_names_hash_bucket_size`

  ```nginx
  server_names_hash_bucket_size size;
  # 默认 server_names_hash_bucket_size 32|64|128：
  # 配置块 http server location
  ```

  nginx使用哈希表的方式存储server_name，该参数指定了每个哈希桶占用内存大小。

- `server_names_hash_max_size`

  ```nginx
  server_names_hash_max_size size;
  # 默认 server_names_hash_max_sizs 512;
  # 配置块 http server location
  ```

  设置哈希表的bucket的数量，越大则冲突越低，但使用内存则会越多。

- 重定向主机名称处理

  ```nginx
  server_name_in_redirect on|off;
  # 默认 server_name_in_redirect on;
  # 配置块 http server location
  ```

  配合server_name使用，表示在重定向请求时会使用server_name里配置的第一个主机名代替原请求的Host头部，而off时，则在重定向时使用请求本身的Host头部。

- location

  ```nginx
  location [=|~|~*|^~|@] /uri/ { ... }
  # 配置块 server
  # uri 支持正则表达式
  ```

  location会根据用户请求的URI来匹配/uri表达式，如果匹配成功，则选择该{}处理请求。

  - = 表示需要完全匹配路径
  - ~ 表示大小写字母敏感
  - ~* 表示忽略大小写
  - ^~ 表示只需要前半部分匹配即可
  - @表示仅用于服务器内部重定向

#### 2. 文件路径定义

- 以root方式设置资源路径

  ```nginx
  # 定义资源文件相对http请求的根目录
  root path;
  # 默认 root html;
  # 配置块 http server location if
  ```

- 以alias方式设置资源路径

  ```nginx
  alias path;
  # 配置块: location
  
  # alias支持正则表达式，并且可以使用参数
  location ~ ^/test/(\w+)\.(\w+)$ {
      alias /usr/local/nginx/$2/$1.$2;
  }
  ```

- 访问首页

  ```nginx
  index file ...;
  # 默认 index index.html;
  # 配置块 http server location
  ```

- 根据HTTP状态码重定向页面

  ```nginx
  # 用户请求状态码重定向到指定的页面
  error_page code path;
  # 配置块 http server location if
  ```

- 是否允许递归使用error_page

  ```nginx
  recursive_error_page on|off;
  # 默认 recursive_error_page off;
  # 配置块 http server location
  ```

- try_files

  ```nginx
  # 如果所有的path路径都找不到，那么重定向到uri
  try_files path1 [path2] uri;
  # 配置块 server location
  ```

#### 3. 内存及磁盘资源分配

- HTTP包只存储到磁盘文件中

  ```nginx
  client_body_in_file_only on|clean|off;
  # 默认 client_body_in_file_only off;
  # 配置块 http server location
  ```

  当设置为on/clean时，用户请求的HTTP包存储到磁盘文件，即使0字节也会存储为文件。请求结束后，clean配置会删除这个文件。

- HTTP包尽量写到一个内存buffer中

  ```nginx
  client_body_in_single_buffer on|off;
  # 默认 client_body_in_single_buffer off;
  # 配置块 http server location
  ```

  用户请求HTTP包存储到内存buffer中，如超过了`client_body_buffer_size`则会写到磁盘中。

- 存储HTTP头部的内存buffer大小

  ```nginx
  client_header_buffer_size size;
  # 默认 client_header_buffer_size 1k;
  # 配置块 http server
  ```

  nginx接收用户请求中header部分时分配内存buffer的大小，当header超过`client_header_buffer_size`这个大小`large_client_header_buffers`这个参数会生效。

- 存储超大HTTP头部的内存buffer大小

  ```nginx
  large_client_header_buffers number size;
  # 默认 large_client_header_buffers 48k;
  # 配置块 http server
  ```

  定义了接收一个超大HTTP头部请求的buffer个数和每个buffer的大小，超过该大小则会出错。

- 存储HTTP包的内存buffer大小

  ```nginx
  client_body_buffer_size size;
  # 默认 client_body_buffer_size 8k/16k;
  # 配置块 http server location
  ```

  定义了Nginx接收HTTP包的内存缓冲区大小，如果头部Content-Length小于这个大小，那么会自动分配本次请求使用的内存，以降低内存消耗。

- HTTP包的临时存放目录

  ```nginx
  client_body_temp_path dir-path [level1[level2[level3]]];
  # 默认 client_body_temp_path
  # 配置块 http server location
  ```

  如果HTTP包过大，则会以整数命令递增的方式存储到该指定目录。

- `connection_pool_size`

  ```nginx
  connection_pool_size size;
  # 默认 connection_pool_size 256;
  # 配置块 http server
  ```

  Nginx对每个建立成功的TCP连接会预分配一个内存池，这个参数指定内存池的初始大小，TCP关闭时销毁。

- `request_pool_size`

  ```nginx
  request_pool_size size;
  # 默认 request_pool_size 4k;
  # 配置块 http server
  ```

  Nginx处理http请求时，会为每个请求分配一个内存池，该参数指定内存池的初始大小，用于减小小块内存的分配次数，请求结束时销毁。

#### 4. 网络连接设置

- 读取HTTP头部超时时间

  ```nginx
  client_header_timeout time(默认单位：秒)
  # 默认 client_header_timeout 60;
  # 配置块 http server location
  ```

- 读取HTTP包体超时时间

  ```nginx
  client_body_timeout time
  # 默认 client_header_timeout 60;
  # 配置块 http server location
  ```

- 发送响应的超时时间

  ```nginx
  send_timeout time;
  # 默认 send_timeout 60;
  # 配置块 http server location
  ```

- `reset_timeout_connection`

  ```nginx
  reset_timeout_connection on|off;
  # 默认 reset_timeout_connection off;
  # 配置块 http server location
  ```

  连接超时后直接发送客户端RESET包，而不是进行四次握手关闭TCP连接。

- `lingering_close`

  ```nginx
  lingering_close off|on|always;
  # 默认 lingering_close on;
  # 配置块 http server location
  ```

  用于设置nginx关闭用户连接的方式，always表示关闭前无条件处理连接上用户发送的数据，off表示完全不管连接上是否已经有准备就绪的来自用户的数据，on是中间值，一般情况会处理，但有些情况会忽略。

- `lingering_time`

  ```nginx
  lingering_time time;
  # 默认 lingering_time 30s;
  # 配置块 http server location
  ```

  lingering_close启用后，该配置对上传大文件有用，若用户上传文件超过设定的大小，nginx服务会发送413，但有些客户端不管该返回仍然上传，这时如果经过了lingering_time设置的时间，直接关闭连接。

- `lingering_timeout`

  ```nginx
  lingering_timeout time;
  # lingering_timeout 5s;
  # 配置块 http server location
  ```

  lingering_close生效后，在关闭连接前会检测是否有数据到达，如果该设定还未数据到达，那么关闭连接。

- 禁用浏览器keepalive功能

  ```nginx
  keepalive_disable [safair|none]
  # 配置块 http server location
  ```

- keepalive超时时间

  ```nginx
  keepalive_timeout time;
  # 配置块 http server location
  ```

- 一个keepalive长连接上允许承载最大请求数

  ```nginx
  keepalive_requests 100;
  # 配置块 http server location
  ```

- `tcp_nodelay`

  ```nginx
  tcp_nodelay on|off;
  # 默认 tcp_nodelay on;
  # 配置块 http server location
  ```

- `tcp_nopush`

  ```nginx
  tcp_nopush on|off;
  # 默认 tcp_nopush off;
  # 配置块 http server location
  ```

#### 5. MIME类型设置

- MIME type到文件扩展的映射

  ```nginx
  type {...};
  # 配置块 http server location
  
  types {
      text/html html;
      text/html conf;
      image/gif gif;
      image/jpeg jpg;
  }
  ```

- 默认MIME type

  ```nginx
  default_type MIME-type;
  # 默认 default_type text/plain;
  # 配置块 http server location
  ```

- `types_hash_bucket_size`

  ```nginx
  types_hash_bucket_size 32|64|128;
  # 配置块 http server location
  ```

  为了快速寻址到对应的MIME，nginx使用哈希表存储文件与MIME，该参数定义散列桶占用内存大小。

- `types_hash_max_size`

  ```nginx
  types_hash_max_size 1024;
  # 配置块 http server location
  ```

#### 6. 客户端请求限制

- 限制HTTP方法

  ```nginx
  limit_except GET {
      allow 192.168.1.0/32;
      deny all;
  }
  ```

- HTTP请求包最大值

  ```nginx
  client_max_body_size 1m;
  # 配置块 http server location
  ```

  用来限制HTTP头部中Content-Length的大小，如果超过则返回413("Request Entity Too Large")。

- 请求速度限制，每秒传输字节

  ```nginx
  limit_rate 0;
  # 配置块 http server location if
  ```

- `limit_rate_after`

  ```nginx
  limit_rate_after 1m;
  # 配置块 http server location if 
  ```

  表示nginx响应时间超过该参数后才进行限速。

#### 7. 文件操作优化

- sendfile系统调用

  ```nginx
  sendfile off|on;
  # 配置块 http server location
  ```

  可以使用Linux的sendfile系统调用发送文件，减少内核态与用户太的两次内存复制，提高发送效率。

- AIO系统调用

  ```nginx
  aio off|on;
  # 配置块 http server location
  ```

  FreeBSD或Linux系统上启用内核级别的异步文件I/O功能，与sendfile功能互斥。

- directio

  ```nginx
  directio off|size;
  # 配置块 http server location
  ```

  FreeBSD或Linux系统上使用O_DIRECT选项读取文件，缓冲区大小为size，通常对大文件读取速度有优化。

- `directio_alignment`

  ```nginx
  directio_alignment 512;
  # 配置块 http server location
  ```

  使用directio时对齐512字节。

- 打开文件缓存

  ```nginx
  open_file_cache off|max=N[inactive=time];
  # 配置块 http server location
  # max: 内存中存储元素的最大个数，超过会使用LRU算法进行替换
  # inactive: 指定时间内没有被访问过的元素淘汰，默认60s
  ```

  文件缓存会缓存文件句柄、文件大小、文件上次修改时间、已经打开过的目录结构、没有找到或没有权限的文件信息，这样减少了磁盘的操作。

- 是否缓存打开文件错误的信息

  ```nginx
  open_file_cache_errors off|on
  # 配置块 http server location
  ```

- 不被淘汰的最小访问次数

  ```nginx
  open_file_cache_min_uses 1;
  # 配置块 http server location
  ```

- 检验缓存中元素有效性的频率

  ```nginx
  open_file_cache_vaild 60s;
  # 配置块 http server location
  # 每60s检查一次缓存中的元素是否仍然有效
  ```

#### 8. 客户端请求特殊处理

- 忽略不合法的HTTP头部

  ```nginx
  ignore_invalid_headers on|off;
  # 配置块 http server
  ```

- HTTP头部是否允许带下划线

  ```nginx
  underscores_in_headers off|on;
  # 配置块 http server
  ```

- 对If-Modified-Since头部的处理策略

  ```nginx
  if_modified_since [exact|off|before];
  # 配置块 http server location
  # off: 忽略请求中If-Modified-Since头部
  # exact: 做精确比较，返回200或304
  # before: 只要文件修改时间小于头部，则返回304
  ```

- 文件未找到是否记录到error日志

  ```nginx
  log_not_found on|off;
  # 配置块 http server location
  ```

- `merge_slashes` 合并相邻的/

  ```nginx
  merge_slashes on|off;
  # 配置块 http server location
  ```

- DNS解析地址

  ```nginx
  resolver address ....;
  # 配置块 http server location
  ```

- DNS解析超时时间

  ```nginx
  resolver_timeout 30s;
  # 配置块 http server location
  ```

- 返回错误时注明Nginx版本

  ```nginx
  server_tokens on|off;
  # 配置块 http server location
  ```

#### 9. HTTP核心模块提供的变量

| 参数名                | 含义                                                         |
| --------------------- | ------------------------------------------------------------ |
| `$arg_PARAMETER`      | HTTP请求中某个参数的值                                       |
| `$args`               | HTTP请求中完整的参数                                         |
| `$binary_remote_addr` | 二进制格式的客户端地址                                       |
| `$body_bytes_sent`    | 回复客户端响应时包大小                                       |
| `$content_length`     | 请求头部Content-Length字段                                   |
| `$content_type`       | 请求头部Content-Type字段                                     |
| `$cookie_COOKIE`      | 请求头部cookie字段                                           |
| `$document_root`      | 当前请求所使用的root配置项的值                               |
| `$uri`                | 当前请求uri，不带任何参数，同`$document_uri`                 |
| `$request_uri`        | 客户端原始请求uri，带完整参数                                |
| `$host`               | 请求头部Host字段，如不存在则以处理server主机名代替           |
| `$hostname`           | nginx所在机器的名称，与gethostbyname值相同                   |
| `$http_HEADER`        | 请求头部相应的值，HEADER全小写                               |
| `$sent_http_HEADER`   | 相应头部相应的值，如`$sent_http_content_type`表示Content-Type |
| `$is_args`            | 请求uri中是否带参数，带参数则值为?，不带则是空字符串         |
| `$limit_rate`         | 连接的限速，0表示不限速                                      |
| `$nginx_version`      | nginx版本号                                                  |
| `$query_string`       | 请求URI的参数，与$args相同，但该参数只读                     |
| `$remote_addr`        | 客户端地址                                                   |
| `$remote_port`        | 客户端端口号                                                 |
| `$remote_user`        | 使用Auth Basic Module时定义的用户名                          |
| `$request_filename`   | 请求中URI经过root或alias转换后的文件路径                     |
| `$request_body`       | 请求包体                                                     |
| `$request_body_file`  | 请求存储的临时文件名                                         |
| `$request_completion` | 请求完成值为ok，若为完成就要返回客户端则为空字符串           |
| `$request_method`     | 请求方法名                                                   |
| `$scheme`             | 请求协议，如https                                            |
| `$server_addr`        | 服务器地址                                                   |
| `$server_name`        | 服务器名称                                                   |
| `$server_port`        | 服务器端口                                                   |
| `$server_protocol`    | 服务器响应协议，如HTTP/1.1                                   |

### 2.4 搭建反向代理服务器

Nginx作为反向代理服务器时，对于客户端的HTTP请求会先缓存的本地内存或磁盘，然后再发送到业务处理服务器，这样可以尽可能的减少业务服务器的压力。

#### 1. 负载均衡基本配置

```nginx
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
    server backend4.example.com;
}
server {
    location / {
        proxy_pass http://backend;
    }
}
```

- upstream块

  定义了一个上游服务器集群，便于反向代理`proxy_pass`使用，配置在http块下

- server

  server配置项指定了一台上游服务器的名称，名称可以是域名、IP地址端口、UNIX句柄等，还可跟一些参数

  - weight=number 设置这台上游服务器转发的权重，默认为1
  - max_fails=number 在指定`fail_timeout`时间内，转发次数超过该值，则认为`fail_timeout`时间段内该服务器不可用，默认为1，设置为0表示不检查
  - fail_timeout=time 默认10秒
  - down 上游服务器永久下线，只有使用ip_hash配置项时有效
  - backup 使用ip_hash配置项时无效，表示上游服务器是备份服务器，只有所有非备份失败时才会使用

- ip_hash

  根据客户端IP地址计算一个key，然后对上游集群进行取模，转发给指定服务器

- 记录日志时支持的变量

  ```nginx
  $upstream_addr  # 处理请求的上游服务器地址
  $upstream_cache_status # 表示是否命中缓存，取值：MISS, EXPIRED, UPDATING, STALE, HIT
  $upstream_status # 上游服务器响应状态码
  $upstream_resposne_time # 上游服务器响应时间，精确到毫秒
  $upstream_http_$HEADER # HTTP的头部
  ```

#### 2. 反向代理的基本配置

- `proxy_pass`

  ```nginx
  proxy_pass http://localhost:8000/uri;
  # 默认情况不转发Host，需要设置 proxy_set_header Host $host;
  ```

- `proxy_method`

  ```nginx
  proxy_method method;
  # 转发为设置的方法
  ```

- `proxy_hide_header`

  ```nginx
  proxy_hide_header HEADER;
  # 指定哪些HEADER不被转发
  ```

- `proxy_pass_header`

  ```nginx
  proxy_pass_header HEADER;
  # 允许转发指定的HEADER
  ```

- `proxy_pass_request_body`

  ```nginx
  proxy_pass_request_body on|off
  # 是否向上游服务器转发包体
  ```

- `proxy_pass_request_headers`

  ```nginx
  proxy_pass_request_headers on|off
  # 是否转发HTTP头部
  ```

- `proxy_redirect`

  ```nginx
  proxy_redirect [default|off|direct replacement];
  ```

  当上游服务器响应时重定向或刷新请求，该参数可以重设HTTP头部的location或refresh字段。

- `proxy_next_upstream`

  ```nginx
  proxy_next_upstream [error|timeout|invalid_header|http_500|http_502]
  ```

  当上游服务器转发请求出现错误，继续换一台处理这个请求。

