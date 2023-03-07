---
layout: wiki  # 使用wiki布局模板
wiki: 深入理解Nginx：模块开发与架构解析 # 这是项目名
title: 第二章 Nginx的配置
order: 3
---
Nginx拥有大量官方发布的模块和第三方模块，这些已有的模块可以帮助我们实现Web服务器上很多的功能。使用这些模块时，仅仅需要增加、修改一些配置项即可。因此，本章的目的是熟悉Nginx的配置文件，包括配置文件的语法格式、运行所有Nginx服务必须具备的基础配置以及使用HTTP核心模块配置静态Web服务器的方法，最后还会介绍反向代理服务器。<br>
通过本章的学习，读者可以：熟练地配置一个静态Web服务器；对影响Web服务器性能的各个配置项有深入的理解；对配置语法有全面的了解。通过互联网或其他途径得到任意模块的配置说明，然后可通过修改nginx.conf文件来使用这些模块的功能。
## 2.1 运行中的Nginx进程间的关系
在正式提供服务的产品环境下，部署Nginx时都是使用一个master进程来管理多个worker进程，一般情况下，worker进程的数量与服务器上的CPU核心数相等。每一个worker进程都是繁忙的，它们在真正地提供互联网服务，master进程则很“清闲”，只负责监控管理worker进程。worker进程之间通过共享内存、原子操作等一些进程间通信机制来实现负载均衡等功能（第9章将会介绍负载均衡机制，第14章将会介绍负载均衡锁的实现）。<br>
部署后Nginx进程间的关系如图2-1所示。<br>
Nginx是支持单进程（master进程）提供服务的，那么为什么产品环境下要按照master-worker方式配置同时启动多个进程呢？这样做的好处主要有以下两点：
- 由于master进程不会对用户请求提供服务，只用于管理真正提供服务的worker进程，所以master进程可以是唯一的，它仅专注于自己的纯管理工作，为管理员提供命令行服务，包括诸如启动服务、停止服务、重载配置文件、平滑升级程序等。master进程需要拥有较大的权限，例如，通常会利用root用户启动master进程。worker进程的权限要小于或等于master进程，这样master进程才可以完全地管理worker进程。当任意一个worker进程出现错误从而导致coredump时，master进程会立刻启动新的worker进程继续服务。
- 多个worker进程处理互联网请求不但可以提高服务的健壮性（一个worker进程出错后，其他worker进程仍然可以正常提供服务），最重要的是，这样可以充分利用现在常见的SMP多核架构，从而实现微观上真正的多核并发处理。因此，用一个进程（master进程）来处理互联网请求肯定是不合适的。另外，为什么要把worker进程数量设置得与CPU核心数量一致呢？这正是Nginx与Apache服务器的不同之处。在Apache上每个进程在一个时刻只处理一个请求，因此，如果希望Web服务器拥有并发处理的请求数更多，就要把Apache的进程或线程数设置得更多，通常会达到一台服务器拥有几百个工作进程，这样大量的进程间切换将带来无谓的系统资源消耗。而Nginx则不然，一个worker进程可以同时处理的请求数只受限于内存大小，而且在架构设计上，不同的worker进程之间处理并发请求时几乎没有同步锁的限制，worker进通常不会进入睡眠状态，因此，当Nginx上的进程数与CPU核心数相等时（最好每一个worker进程都绑定特定的CPU核心），进程间切换的代价是最小的。
  {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/chapter_02/nginx-chapter02-001.png 图2-1 部署后Nginx进程间的关系 %}

举例来说，如果产品中的服务器CPU核心数为8，那么就需要配置8个worker进程（见图2-2）。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/chapter_02/nginx-chapter02-002.png 图2-2 worker进程的数量尽量与CPU核心数相等 %}
如果对路径部分都使用默认配置，那么Nginx运行目录为/usr/local/nginx，其目录结构如下。
```shell
|---sbin
| |---nginx
|---conf
| |---koi-win
| |---koi-utf
| |---win-utf
| |---mime.types
| |---mime.types.default
| |---fastcgi_params
| |---fastcgi_params.default
| |---fastcgi.conf
| |---fastcgi.conf.default
| |---uwsgi_params
| |---uwsgi_params.default
| |---scgi_params
| |---scgi_params.default
| |---nginx.conf
| |---nginx.conf.default
|---logs
| |---error.log
| |---access.log
| |---nginx.pid
|---html
| |---50x.html
| |---index.html
|---client_body_temp
|---proxy_temp
|---fastcgi_temp
|---uwsgi_temp
|---scgi_temp
```
## 2.2 Nginx配置的通用语法
Nginx的配置文件其实是一个普通的文本文件。下面来看一个简单的例子。
```text
user nobody;
worker_processes 8;
error_log /var/log/nginx/error.log error;
#pid logs/nginx.pid;
events {
     use epoll;
     worker_connections 50000;
}
http {
     include mime.types;
     default_type application/octet-stream;
     log_format main '$remote_addr [$time_local] "$request" '
     '$status $bytes_sent "$http_referer" '
     '"$http_user_agent" "$http_x_forwarded_for"';
     access_log logs/access.log main buffer=32k;
     ...
｝
```
在这段简短的配置代码中，每一行配置项的语法格式都将在2.2.2节介绍，出现的events和http块配置项将在2.2.1节介绍，以#符号开头的注释将在2.2.3节介绍，类似“buffer=32k”这样的配置项的单位将在2.2.4节介绍。
### 2.2.1 块配置项
块配置项由一个块配置项名和一对大括号组成。具体示例如下：
```text
events {... }
http {
   upstream backend {
      server 127.0.0.1:8080;
   }
   gzip on;
   server {
      ...
      location /webstatic {
          gzip off;
      }
   }
}
```
上面代码段中的events、http、server、location、upstream等都是块配置项，块配置项之后是否如“location/webstatic{...}”那样在后面加上参数，取决于解析这个块配置项的模块，不能一概而论，但块配置项一定会用大括号把一系列所属的配置项全包含进来，表示大括号内的配置项同时生效。所有的事件类配置都要在events块中，http、server等配置也遵循这个规定。<br>
块配置项可以嵌套。内层块直接继承外层块，例如，上例中，server块里的任意配置都是基于http块里的已有配置的。当内外层块中的配置发生冲突时，究竟是以内层块还是外层块的配置为准，取决于解析这个配置项的模块，第4章将会介绍http块内配置项冲突的处理方法。例如，上例在http模块中已经打开了“gzip on;”，但其下的location/webstatic又把gzip关闭了：gzip off;，最终，在/webstatic的处理模块中，gzip模块是按照gzip off来处理请求的。
### 2.2.2 配置项的语法格式
从上文的示例可以看出，最基本的配置项语法格式如下：
```text
配置项名 配置项值
1 配置项值
2 ... ;
```
下面解释一下配置项的构成部分。<br>
首先，在行首的是配置项名，这些配置项名必须是Nginx的某一个模块想要处理的，否则Nginx会认为配置文件出现了非法的配置项名。配置项名输入结束后，将以空格作为分隔符。<br>
其次是配置项值，它可以是数字或字符串（当然也包括正则表达式）。针对一个配置项，既可以只有一个值，也可以包含多个值，配置项值之间仍然由空格符来分隔。当然，一个配置项对应的值究竟有多少个，取决于解析这个配置项的模块。我们必须根据某个Nginx模块对一个配置项的约定来更改配置项，第4章将会介绍模块是如何约定一个配置项的格式。<br>
最后，每行配置的结尾需要加上分号。
{% note color:light 注意 如果配置项值中包括语法符号，比如空格符，那么需要使用单引号或双引号括住配置项值，否则Nginx会报语法错误。例如：log_format main '$remote_addr - $remote_user [$time_local] "$request" '; %}
### 2.2.3 配置项的注释
如果有一个配置项暂时需要注释掉，那么可以加“#”注释掉这一行配置。例如：
```text
#pid logs/nginx.pid;
```
### 2.2.4 配置项的单位
大部分模块遵循一些通用的规定，如指定空间大小时不用每次都定义到字节、指定时间时不用精确到毫秒。<br>
当指定空间大小时，可以使用的单位包括：
- K或者k千字节（KiloByte，KB）。 
- M或者m兆字节（MegaByte，MB）。 
  例如：
  ```text
  gzip_buffers 4 8k;
  client_max_body_size 64M;
  ```
当指定时间时，可以使用的单位包括：
- ms（毫秒），s（秒），m（分钟），h（小时），d（天），w（周，包含7天），M（月，包含30天），y（年，包含365天）。
  例如：
  ```text
  expires 10y;
  proxy_read_timeout 600;
  client_body_timeout 2m;
  ```

{% note color:light 注意 配置项后的值究竟是否可以使用这些单位，取决于解析该配置项的模块。如果这个模块使用了Nginx框架提供的相应解析配置项方法，那么配置项值才可以携带单位。第4章中详细描述了Nginx框架提供的14种预设解析方法，其中一些方法将可以解析以上列出的单位。 %}
### 2.2.5 在配置中使用变量
有些模块允许在配置项中使用变量，如在日志记录部分，具体示例如下。
```text
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
     '$status $bytes_sent "$http_referer" '
     '"$http_user_agent" "$http_x_forwarded_for"';
```
其中，remote_addr是一个变量，使用它的时候前面要加上$符号。需要注意的是，这种变量只有少数模块支持，并不是通用的。<br>
许多模块在解析请求时都会提供多个变量（如本章后面提到的http core module、http proxy module、http upstream module等），以使其他模块的配置可以即时使用。我们在学习某个模块提供的配置说明时可以关注它是否提供变量。
{% note color:light 提示 在执行configure命令时，我们已经把许多模块编译进Nginx中，但是否启用这些模块，一般取决于配置文件中相应的配置项。换句话说，每个Nginx模块都有自己感兴趣的配置项，大部分模块都必须在nginx.conf中读取某个配置项后才会在运行时启用。例如，只有当配置http{...}这个配置项时，ngx_http_module模块才会在Nginx中启用，其他依赖ngx_http_module的模块也才能正常使用。 %}
## 2.3 Nginx服务的基本配置
Nginx在运行时，至少必须加载几个核心模块和一个事件类模块。这些模块运行时所支持的配置项称为基本配置——所有其他模块执行时都依赖的配置项。<br>
下面详述基本配置项的用法。由于配置项较多，所以把它们按照用户使用时的预期功能分成了以下4类：
- 用于调试、定位问题的配置项。 
- 正常运行的必备配置项。 
- 优化性能的配置项。 
- 事件类配置项（有些事件类配置项归纳到优化性能类，这是因为它们虽然也属于events{}块，但作用是优化性能）。

有这么一些配置项，即使没有显式地进行配置，它们也会有默认的值，如daemon，即使在nginx.conf中没有对它进行配置，也相当于打开了这个功能，这点需要注意。对于这样的配置项，作者会在下面相应的配置项描述上加入一行“默认：”来进行说明。
### 2.3.1 用于调试进程和定位问题的配置项
先来看一下用于调试进程、定位问题的配置项，如下所示。
1. 是否以守护进程方式运行Nginx
   **语法**： `daemon on|off;`
   **默认**： `daemon on;`
   守护进程（daemon）是脱离终端并且在后台运行的进程。它脱离终端是为了避免进程执行过程中的信息在任何终端上显示，这样一来，进程也不会被任何终端所产生的信息所打断。Nginx毫无疑问是一个需要以守护进程方式运行的服务，因此，默认都是以这种方式运行的。<br>
   不过Nginx还是提供了关闭守护进程的模式，之所以提供这种模式，是为了方便跟踪调试Nginx，毕竟用gdb调试进程时最烦琐的就是如何继续跟进fork出的子进程了。这在第三部分研究Nginx架构时很有用。
2. 是否以master/worker方式工作
   **语法**： `master_process on|off;`
   **默认**： `master_process on;`
   可以看到，在如图2-1所示的产品环境中，是以一个master进程管理多个worker进程的方式运行的，几乎所有的产品环境下，Nginx都以这种方式工作。<br>
   与daemon配置相同，提供master_process配置也是为了方便跟踪调试Nginx。如果用off关闭了master_process方式，就不会fork出worker子进程来处理请求，而是用master进程自身来处理请求。
3. error日志的设置
   **语法**： `error_log/path/file level;`
   **默认**： `error_log logs/error.log error;`
   error日志是定位Nginx问题的最佳工具，我们可以根据自己的需求妥善设置error日志的路径和级别。<br>
   /path/file参数可以是一个具体的文件，例如，默认情况下是logs/error.log文件，最好将它放到一个磁盘空间足够大的位置；/path/file也可以是/dev/null，这样就不会输出任何日志了，这也是关闭error日志的唯一手段；/path/file也可以是stderr，这样日志会输出到标准错误文件中。<br>
   level是日志的输出级别，取值范围是debug、info、notice、warn、error、crit、alert、emerg，从左至右级别依次增大。当设定为一个级别时，大于或等于该级别的日志都会被输出到/path/file文件中，小于该级别的日志则不会输出。例如，当设定为error级别时，error、crit、alert、emerg级别的日志都会输出。<br>
   如果设定的日志级别是debug，则会输出所有的日志，这样数据量会很大，需要预先确保/path/file所在磁盘有足够的磁盘空间。
   {% note color:light 注意 如果日志级别设定到debug，必须在configure时加入--with-debug配置项。 %} 
4. 是否处理几个特殊的调试点
   **语法**： `debug_points[stop|abort]`
   这个配置项也是用来帮助用户跟踪调试Nginx的。它接受两个参数：stop和abort。Nginx在一些关键的错误逻辑中（Nginx 1.0.14版本中有8处）设置了调试点。如果设置了debug_points为stop，那么Nginx的代码执行到这些调试点时就会发出SIGSTOP信号以用于调试。如果debug_points设置为abort，则会产生一个coredump文件，可以使用gdb来查看Nginx当时的各种信息。<br>
   通常不会使用这个配置项。 
5. 仅对指定的客户端输出debug级别的日志
   **语法**： `debug_connection[IP|CIDR]`
   这个配置项实际上属于事件类配置，因此，它必须放在events{...} 中才有效。它的值可以是IP地址或CIDR地址，例如：
   ```text
   events {
       debug_connection 10.224.66.14;
       debug_connection 10.224.57.0/24;
   }
   ```
   这样，仅仅来自以上IP地址的请求才会输出debug级别的日志，其他请求仍然沿用error_log中配置的日志级别。<br>
   上面这个配置对修复Bug很有用，特别是定位高并发请求下才会发生的问题。
   {% note color:light 注意 使用debug_connection前，需确保在执行configure时已经加入了--with-debug参数，否则不会生效。%}
6. 限制coredump核心转储文件的大小
   **语法**： `worker_rlimit_core size;`
   在Linux系统中，当进程发生错误或收到信号而终止时，系统会将进程执行时的内存内容（核心映像）写入一个文件（core文件），以作为调试之用，这就是所谓的核心转储（core dumps）。当Nginx进程出现一些非法操作（如内存越界）导致进程直接被操作系统强制结束时，会生成核心转储core文件，可以从core文件获取当时的堆栈、寄存器等信息，从而帮助我们定位问题。但这种core文件中的许多信息不一定是用户需要的，如果不加以限制，那么可能一个core文件会达到几GB，这样随便coredumps几次就会把磁盘占满，引发严重问题。通过worker_rlimit_core配置可以限制core文件的大小，从而有效帮助用户定位问题。
7. 指定coredump文件生成目录
   **语法**： `working_directory path;`
   worker进程的工作目录。这个配置项的唯一用途就是设置coredump文件所放置的目录，协助定位问题。因此，需确保worker进程有权限向working_directory指定的目录中写入文件。
### 2.3.2 正常运行的配置项
下面是正常运行的配置项的相关介绍。
1. 定义环境变量 
   **语法**： `env VAR|VAR=VALUE`
   这个配置项可以让用户直接设置操作系统上的环境变量。例如：
   ```text
   env TESTPATH=/tmp/;
   ```
2. 嵌入其他配置文件
   **语法**： `include/path/file;`
   include配置项可以将其他配置文件嵌入到当前的nginx.conf文件中，它的参数既可以是绝对路径，也可以是相对路径（相对于Nginx的配置目录，即nginx.conf所在的目录），例如：
   ```text
   include mime.types;
   include vhost/*.conf;
   ```
   可以看到，参数的值可以是一个明确的文件名，也可以是含有通配符*的文件名，同时可以一次嵌入多个配置文件。
3. pid文件的路径
   **语法**： `pid path/file;`
   **默认**： `pid logs/nginx.pid;`
   保存master进程ID的pid文件存放路径。默认与configure执行时的参数“--pid-path”所指定的路径是相同的，也可以随时修改，但应确保Nginx有权在相应的目标中创建pid文件，该文件直接影响Nginx是否可以运行。
4. Nginx worker进程运行的用户及用户组
   **语法**： `user username[groupname];`
   **默认**： `user nobody nobody;`
   user用于设置master进程启动后，fork出的worker进程运行在哪个用户和用户组下。当按照“user username;”设置时，用户组名与用户名相同。<br>
   若用户在configure命令执行时使用了参数--user=username和--group=groupname，此时nginx.conf将使用参数中指定的用户和用户组。
5. 指定Nginx worker进程可以打开的最大句柄描述符个数
   **语法**： `worker_rlimit_nofile limit;`
   设置一个worker进程可以打开的最大文件句柄数。
6. 限制信号队列
   **语法**： `worker_rlimit_sigpending limit;`
   设置每个用户发往Nginx的信号队列的大小。也就是说，当某个用户的信号队列满了，这个用户再发送的信号量会被丢掉。

### 2.3.3 优化性能的配置项
下面是优化性能的配置项的相关介绍。
1. Nginx worker进程个数
   **语法**： `worker_processes number;`
   **默认**： `worker_processes 1;`
   在master/worker运行方式下，定义worker进程的个数。<br>
   worker进程的数量会直接影响性能。那么，用户配置多少个worker进程才好呢？这实际上与业务需求有关。<br>
   每个worker进程都是单线程的进程，它们会调用各个模块以实现多种多样的功能。如果这些模块确认不会出现阻塞式的调用，那么，有多少CPU内核就应该配置多少个进程；反之，如果有可能出现阻塞式调用，那么需要配置稍多一些的worker进程。<br>
   例如，如果业务方面会致使用户请求大量读取本地磁盘上的静态资源文件，而且服务器上的内存较小，以至于大部分的请求访问静态资源文件时都必须读取磁盘（磁头的寻址是缓慢的），而不是内存中的磁盘缓存，那么磁盘I/O调用可能会阻塞住worker进程少量时间，进而导致服务整体性能下降。<br>
   多worker进程可以充分利用多核系统架构，但若worker进程的数量多于CPU内核数，那么会增大进程间切换带来的消耗（Linux是抢占式内核）。一般情况下，用户要配置与CPU内核数相等的worker进程，并且使用下面的worker_cpu_affinity配置来绑定CPU内核。
2. 绑定Nginx worker进程到指定的CPU内核
   **语法**： `worker_cpu_affinity cpumask[cpumask...]`
   为什么要绑定worker进程到指定的CPU内核呢？假定每一个worker进程都是非常繁忙的，如果多个worker进程都在抢同一个CPU，那么这就会出现同步问题。反之，如果每一个worker进程都独享一个CPU，就在内核的调度策略上实现了完全的并发。<br>
   例如，如果有4颗CPU内核，就可以进行如下配置：
   ```text
   worker_processes 4;
   worker_cpu_affinity 1000 0100 0010 0001;
   ```
   {% note color:light 注意 worker_cpu_affinity配置仅对Linux操作系统有效。Linux操作系统使用sched_setaffinity()系统调用实现这个功能。 %}
3. SSL硬件加速
   **语法**： `ssl_engine device；`
   如果服务器上有SSL硬件加速设备，那么就可以进行配置以加快SSL协议的处理速度。用户可以使用OpenSSL提供的命令来查看是否有SSL硬件加速设备：
   ```text
   openssl engine -t
   ```
4. 系统调用gettimeofday的执行频率
   **语法**： `timer_resolution t;`
   默认情况下，每次内核的事件调用（如epoll、select、poll、kqueue等）返回时，都会执行一次gettimeofday，实现用内核的时钟来更新Nginx中的缓存时钟。在早期的Linux内核中，gettimeofday的执行代价不小，因为中间有一次内核态到用户态的内存复制。当需要降低gettimeofday的调用频率时，可以使用timer_resolution配置。例如，“timer_resolution 100ms；”表示至少每100ms才调用一次gettimeofday。<br>
   但在目前的大多数内核中，如x86-64体系架构，gettimeofday只是一次vsyscall，仅仅对共享内存页中的数据做访问，并不是通常的系统调用，代价并不大，一般不必使用这个配置。而且，如果希望日志文件中每行打印的时间更准确，也可以使用它。
5. Nginx worker进程优先级设置
   **语法**： `worker_priority nice;`
   **默认**： `worker_priority 0;`
   该配置项用于设置Nginx worker进程的nice优先级。<br>
   在Linux或其他类UNIX操作系统中，当许多进程都处于可执行状态时，将按照所有进程的优先级来决定本次内核选择哪一个进程执行。进程所分配的CPU时间片大小也与进程优先级相关，优先级越高，进程分配到的时间片也就越大（例如，在默认配置下，最小的时间片只有5ms，最大的时间片则有800ms）。这样，优先级高的进程会占有更多的系统资源。<br>
   优先级由静态优先级和内核根据进程执行情况所做的动态调整（目前只有±5的调整）共同决定。nice值是进程的静态优先级，它的取值范围是–20~+19，–20是最高优先级，+19是最低优先级。因此，如果用户希望Nginx占有更多的系统资源，那么可以把nice值配置得更小一些，但不建议比内核进程的nice值（通常为–5）还要小。

### 2.3.4 事件类配置项
下面是事件类配置项的相关介绍。
1. 是否打开accept锁
   **语法**： `accept_mutex[on|off]`
   **默认**： `accept_mutext on;`
   accept_mutex是Nginx的负载均衡锁，本书会在第9章事件处理框架中详述Nginx是如何实现负载均衡的。这里，读者仅需要知道accept_mutex这把锁可以让多个worker进程轮流地、序列化地与新的客户端建立TCP连接。当某一个worker进程建立的连接数量达到worker_connections配置的最大连接数的7/8时，会大大地减小该worker进程试图建立新TCP连接的机会，以此实现所有worker进程之上处理的客户端请求数尽量接近。<br>
   accept锁默认是打开的，如果关闭它，那么建立TCP连接的耗时会更短，但worker进程之间的负载会非常不均衡，因此不建议关闭它。
2. lock文件的路径
   **语法**： `lock_file path/file;`
   **默认**： `lock_file logs/nginx.lock;`
   accept锁可能需要这个lock文件，如果accept锁关闭，lock_file配置完全不生效。如果打开了accept锁，并且由于编译程序、操作系统架构等因素导致Nginx不支持原子锁，这时才会用文件锁实现accept锁 （14.8.1节将会介绍文件锁的用法），这样lock_file指定的lock文件才会生效。
   {% note color:light 注意 在基于i386、AMD64、Sparc64、PPC64体系架构的操作系统上，若使用GCC、Intel C++、SunPro C++编译器来编译Nginx，则可以肯定这时的Nginx是支持原子锁的，因为Nginx会利用CPU的特性并用汇编语言来实现它（可以参考14.3节x86架构下原子操作的实现）。这时的lock_file配置是没有意义的。 %}
3. 使用accept锁后到真正建立连接之间的延迟时间
   **语法**： `accept_mutex_delay Nms;`
   **默认**： `accept_mutex_delay 500ms;`
   在使用accept锁后，同一时间只有一个worker进程能够取到accept锁。这个accept锁不是阻塞锁，如果取不到会立刻返回。如果有一个worker进程试图取accept锁而没有取到，它至少要等accept_mutex_delay定义的时间间隔后才能再次试图取锁。
4. 批量建立新连接
   **语法**： `multi_accept[on|off];`
   **默认**： `multi_accept off;`
   当事件模型通知有新连接时，尽可能地对本次调度中客户端发起的所有TCP请求都建立连接。
5. 选择事件模型
   **语法**： `use[kqueue|rtsig|epoll|/dev/poll|select|poll|eventport];`
   **默认**： `Nginx会自动使用最适合的事件模型。`
   对于Linux操作系统来说，可供选择的事件驱动模型有poll、select、epoll三种。epoll当然是性能最高的一种，在9.6节会解释epoll为什么可以处理大并发连接。
6. 每个worker的最大连接数
   **语法**： `worker_connections number;`
   定义每个worker进程可以同时处理的最大连接数。

## 2.4 用HTTP核心模块配置一个静态Web服务器
静态Web服务器的主要功能由ngx_http_core_module模块（HTTP框架的主要成员）实现，当然，一个完整的静态Web服务器还有许多功能是由其他的HTTP模块实现的。本节主要讨论如何配置一个包含基本功能的静态Web服务器，文中会完整地说明ngx_http_core_module模块提供的配置项及变量的用法，但不会过多说明其他HTTP模块的配置项。在阅读完本节内容后，读者应当可以通过简单的查询相关模块（如ngx_http_gzip_filter_module、ngx_http_image_filter_module等）的配置项说明，方便地在nginx.conf配置文件中加入新的配置项，从而实现更多的Web服务器功能。<br>
除了2.3节提到的基本配置项外，一个典型的静态Web服务器还会包含多个server块和location块，例如：
```text
http {
    gzip on;
    upstream {
         ...
    }
    ...
    server {
          listen localhost:80;
          ...
          location /webstatic {
                if ... {
                     ...
                }
                root /opt/webresource;
                ...
          }
          location ~* .(jpg|jpeg|png|jpe|gif)$ {
               ...
          }
    }
    server {
         ...
    }
}
```
所有的HTTP配置项都必须直属于http块、server块、location块、upstream块或if块等（HTTP配置项自然必须全部在http{}块之内，这里的“直属于”是指配置项直接所属的大括号对应的配置块），同时，在描述每个配置项的功能时，会说明它可以在上述的哪个块中存在，因为有些配置项可以任意地出现在某一个块中，而有些配置项只能出现在特定的块中，在第4章介绍自定义配置项的读取时，相信读者就会体会到这种设计思路。<br>
Nginx为配置一个完整的静态Web服务器提供了非常多的功能，下面会把这些配置项分为以下8类进行详述：虚拟主机与请求的分发、文件路径的定义、内存及磁盘资源的分配、网络连接的设置、MIME类型的设置、对客户端请求的限制、文件操作的优化、对客户端请求的特殊处理。这种划分只是为了帮助大家从功能上理解这些配置项。<br>
在这之后会列出ngx_http_core_module模块提供的变量，以及简单说明它们的意义。
### 2.4.1 虚拟主机与请求的分发
由于IP地址的数量有限，因此经常存在多个主机域名对应着同一个IP地址的情况，这时在nginx.conf中就可以按照server_name（对应用户请求中的主机域名）并通过server块来定义虚拟主机，每个server块就是一个虚拟主机，它只处理与之相对应的主机域名请求。这样，一台服务器上的Nginx就能以不同的方式处理访问不同主机域名的HTTP请求了。
1. 监听端口
   **语法**： `listen address:port[default(deprecated in 0.8.21)|default_server|[backlog=num|rcvbuf=size|sndbuf=size|accept_filter=filter|deferred|bind|ipv6only=[on|off]|ssl]];`
   **默认**： `listen 80;`
   **配置块**： `server`
   listen参数决定Nginx服务如何监听端口。在listen后可以只加IP地址、端口或主机名，非常灵活，例如：
   ```text
   listen 127.0.0.1:8000;
   listen 127.0.0.1; #注意：不加端口时，默认监听80端口
   listen 8000;
   listen *:8000;
   listen localhost:8000;
   ```
   如果服务器使用IPv6地址，那么可以这样使用：
   ```text
   listen [::]:8000;
   listen [fe80::1];
   listen [:::a8c9:1234]:80;
   ```
   在地址和端口后，还可以加上其他参数，例如：
   ```text
   listen 443 default_server ssl;
   listen 127.0.0.1 default_server accept_filter=dataready backlog=1024;
   ```
   下面说明listen可用参数的意义。
   - default：将所在的server块作为整个Web服务的默认server块。如果没有设置这个参数，那么将会以在nginx.conf中找到的第一个server块作为默认server块。为什么需要默认虚拟主机呢？当一个请求无法匹配配置文件中的所有主机域名时，就会选用默认的虚拟主机（在11.3节介绍默认主机的使用）。 
   - default_server：同上。 
   - backlog=num：表示TCP中backlog队列的大小。默认为–1，表示不予设置。在TCP建立三次握手过程中，进程还没有开始处理监听句柄，这时backlog队列将会放置这些新连接。可如果backlog队列已满，还有新的客户端试图通过三次握手建立TCP连接，这时客户端将会建立连接失败。
   - rcvbuf=size：设置监听句柄的SO_RCVBUF参数。 
   - sndbuf=size：设置监听句柄的SO_SNDBUF参数。 
   - accept_filter：设置accept过滤器，只对FreeBSD操作系统有用。 
   - deferred：在设置该参数后，若用户发起建立连接请求，并且完成了TCP的三次握手，内核也不会为了这次的连接调度worker进程来处理，只有用户真的发送请求数据时（内核已经在网卡中收到请求数据包），内核才会唤醒worker进程处理这个连接。这个参数适用于大并发的情况下，它减轻了worker进程的负担。当请求数据来临时，worker进程才会开始处理这个连接。只有确认上面所说的应用场景符合自己的业务需求时，才可以使用deferred配置。
   - bind：绑定当前端口/地址对，如127.0.0.1:8000。只有同时对一个端口监听多个地址时才会生效。
   - ssl：在当前监听的端口上建立的连接必须基于SSL协议。 
2. 主机名称 
   **语法**： `server_name name[...];`
   **默认**： `server_name"";`
   **配置块**： `server`
   server_name后可以跟多个主机名称，如server_name www.testweb.com 、download.testweb.com;。<br>
   在开始处理一个HTTP请求时，Nginx会取出header头中的Host，与每个server中的server_name进行匹配，以此决定到底由哪一个server块来处理这个请求。有可能一个Host与多个server块中的server_name都匹配，这时就会根据匹配优先级来选择实际处理的server块。server_name与Host的匹配优先级如下：
   1. 首先选择所有字符串完全匹配的server_name，如www.testweb.com 。 
   2. 其次选择通配符在前面的server_name，如*.testweb.com。 
   3. 再次选择通配符在后面的server_name，如www.testweb.* 。 
   4. 最后选择使用正则表达式才匹配的server_name，如~^\.testweb\.com$。
   实际上，这个规则正是7.7节中介绍的带通配符散列表的实现依据，同时，在10.4节也介绍了虚拟主机配置的管理。如果Host与所有的server_name都不匹配，这时将会按下列顺序选择处理的server块。
   

   1. 优先选择在listen配置项后加入[default|default_server]的server块。
   2. 找到匹配listen端口的第一个server块。
      如果server_name后跟着空字符串（如server_name"";），那么表示匹配没有Host这个HTTP头部的请求。
      {% note color:light 注意 Nginx正是使用server_name配置项针对特定Host域名的请求提供不同的服务，以此实现虚拟主机功能。 %}
3. server_names_hash_bucket_size
   **语法**： `server_names_hash_bucket_size size;`
   **默认**： `server_names_hash_bucket_size 32|64|128;`
   **配置块**： `http、server、location`
   为了提高快速寻找到相应server name的能力，Nginx使用散列表来存储server name。server_names_hash_bucket_size设置了每个散列桶占用的内存大小。
4. server_names_hash_max_size
   **语法**： `server_names_hash_max_size size;`
   **默认**： `server_names_hash_max_size 512;`
   **配置块**： `http、server、location`
   server_names_hash_max_size会影响散列表的冲突率。 server_names_hash_max_size越大，消耗的内存就越多，但散列key的冲突率则会降低，检索速度也更快。server_names_hash_max_size越小，消耗的内存就越小，但散列key的冲突率可能增高。
5. 重定向主机名称的处理 
   **语法**： `server_name_in_redirect on|off;`
   **默认**： `server_name_in_redirect on;`
   **配置块**： `http、server或者location`
   该配置需要配合server_name使用。在使用on打开时，表示在重定向请求时会使用server_name里配置的第一个主机名代替原先请求中的Host头部，而使用off关闭时，表示在重定向请求时使用请求本身的Host头部。
6. location 
   **语法**： `location[=|~|~*|^~|@]/uri/{...}`
   **配置块**： `server`
   location会尝试根据用户请求中的URI来匹配上面的/uri表达式，如果可以匹配，就选择location{}块中的配置来处理用户请求。当然，匹配方式是多样的，下面介绍location的匹配规则。
   1. =表示把URI作为字符串，以便与参数中的uri做完全匹配。例如：
      ```text
      location = / {
            # 只有当用户请求是 /时，才会使用该location下的配置 
            ...
      } 
      ```
   2. ~表示匹配URI时是字母大小写敏感的。
   3. ~*表示匹配URI时忽略字母大小写问题。 
   4. ^~表示匹配URI时只需要其前半部分与uri参数匹配即可。例如：
      ```text
      location ^~ /images/ {
            # 以/images/开始的请求都会匹配上
            ...
      } 
      ```
   5. @表示仅用于Nginx服务内部请求之间的重定向，带有@的location不直接处理用户请求。当然，在uri参数里是可以用正则表达式的，例如：
      ```text
      location ~* \.(gif|jpg|jpeg)$ {
            # 匹配以.gif、.jpg、.jpeg结尾的请求 
            ...
      }
      ```
      注意，location是有顺序的，当一个请求有可能匹配多个location时，实际上这个请求会被第一个location处理。在以上各种匹配方式中，都只能表达为“如果匹配...则...”。如果需要表达“如果不匹配...则...”，就很难直接做到。有一种解决方法是在最后一个location中使用/作为参数，它会匹配所有的HTTP请求，这样就可以表示如果不能匹配前面的所有location，则由“/”这个location处理。
      例如：
      ```text
      location / {
            # /可以匹配所有请求
            ...
      }
      ```

### 2.4.2 文件路径的定义
下面介绍一下文件路径的定义配置项。
1. 以root方式设置资源路径
   **语法**： `root path;`
   **默认**： `root html;`
   **配置块**： `http、server、location、if`
   例如，定义资源文件相对于HTTP请求的根目录。
   ```text
   location /download/ {
       root /opt/web/html/;
   }
   ```
   在上面的配置中，如果有一个请求的URI是/download/index/test.html，那么Web服务器将会返回服务器上/opt/web/html/download/index/test.html文件的内容。
2. 以alias方式设置资源路径
   **语法**： `alias path;`
   **配置块**： `location`
   alias也是用来设置文件资源路径的，它与root的不同点主要在于如何解读紧跟location后面的uri参数，这将会致使alias与root以不同的方式将用户请求映射到真正的磁盘文件上。例如，如果有一个请求的URI是/conf/nginx.conf，而用户实际想访问的文件在/usr/local/nginx/conf/nginx.conf，那么想要使用alias来进行设置的话，可以采用如下方式：
   ```text
   location /conf {
       alias /usr/local/nginx/conf/;
   }
   ```
   如果用root设置，那么语句如下所示：
   ```text
   location /conf {
       root /usr/local/nginx/;
   }
   ```
   使用alias时，在URI向实际文件路径的映射过程中，已经把location后配置的/conf这部分字符串丢弃掉，因此，/conf/nginx.conf请求将根据alias path映射为path/nginx.conf。root则不然，它会根据完整的URI请求来映射，因此，/conf/nginx.conf请求会根据root path映射为path/conf/nginx.conf。这也是root可以放置到http、server、location或if块中，而alias只能放置到location块中的原因。alias后面还可以添加正则表达式，例如：
   ```text
   location ~ ^/test/(\w+)\.(\w+)$ {
       alias /usr/local/nginx/$2/$1.$2;
   }
   ```
   这样，请求在访问/test/nginx.conf时，Nginx会返回/usr/local/nginx/conf/nginx.conf文件中的内容。
3. 访问首页
   **语法**： `index file...;`
   **默认**： `index index.html;`
   **配置块**： `http、server、location`
   有时，访问站点时的URI是/，这时一般是返回网站的首页，而这与root和alias都不同。这里用ngx_http_index_module模块提供的index配置实现。index后可以跟多个文件参数，Nginx将会按照顺序来访问这些文件，例如：
   ```text
   location / {
      root path;
      index /index.html /html/index.php /index.php;
   }
   ```
   接收到请求后，Nginx首先会尝试访问path/index.php文件，如果可以访问，就直接返回文件内容结束请求，否则再试图返回path/html/index.php文件的内容，依此类推。
4. 根据HTTP返回码重定向页面
   **语法**： `error_page code[code...][=|=answer-code]uri|@named_location`
   **配置块**： `http、server、location、if`
   当对于某个请求返回错误码时，如果匹配上了error_page中设置的code，则重定向到新的URI中。例如：
   ```text
   error_page 404 /404.html;
   error_page 502 503 504 /50x.html;
   error_page 403 http://example.com/forbidden.html;
   error_page 404 = @fetch;
   ```
   注意，虽然重定向了URI，但返回的HTTP错误码还是与原来的相同。用户可以通过“=”来更改返回的错误码，例如：
   ```text
   error_page 404 =200 /empty.gif;
   error_page 404 =403 /forbidden.gif;
   ```
   也可以不指定确切的返回错误码，而是由重定向后实际处理的真实结果来决定，这时，只要把“=”后面的错误码去掉即可，例如：
   ```text
   error_page 404 = /empty.gif;
   ```
   如果不想修改URI，只是想让这样的请求重定向到另一个location中进行处理，那么可以这样设置：
   ```text
   location / (
       error_page 404 @fallback;
   )
   location @fallback (
       proxy_pass http://backend;
   )
   ```
   这样，返回404的请求会被反向代理到http://backend 上游服务器中处理。
5. 是否允许递归使用error_page
   **语法**： `recursive_error_pages[on|off];`
   **默认**： `recursive_error_pages off;`
   **配置块**： `http、server、location`
   确定是否允许递归地定义error_page。 
6. try_files
   **语法**： `try_files path1[path2]uri;`
   **配置块**： `server、location`
   try_files后要跟若干路径，如path1 path2...，而且最后必须要有uri参数，意义如下：尝试按照顺序访问每一个path，如果可以有效地读取，就直接向用户返回这个path对应的文件结束请求，否则继续向下访问。如果所有的path都找不到有效的文件，就重定向到最后的参数uri上。因此，最后这个参数uri必须存在，而且它应该是可以有效重定向的。例如：
   ```text
   try_files /system/maintenance.html $uri $uri/index.html $uri.html @other;
   location @other {
       proxy_pass http://backend;
   }
   ```
   上面这段代码表示如果前面的路径，如/system/maintenance.html等，都找不到，就会反向代理到http://backend 服务上。还可以用指定错误码的方式与error_page配合使用，例如：
   ```text
   location / {
   try_files $uri $uri/ /error.phpc=404 =404;
   }
   ```
   
### 2.4.3 内存及磁盘资源的分配
下面介绍处理请求时内存、磁盘资源分配的配置项。
1. HTTP包体只存储到磁盘文件中
   **语法**： `client_body_in_file_only on|clean|off;`
   **默认**： `client_body_in_file_only off;`
   **配置块**： `http、server、location`
   当值为非off时，用户请求中的HTTP包体一律存储到磁盘文件中，即使只有0字节也会存储为文件。当请求结束时，如果配置为on，则这个文件不会被删除（该配置一般用于调试、定位问题），但如果配置为clean，则会删除该文件。
2. HTTP包体尽量写入到一个内存buffer中
   **语法**： `client_body_in_single_buffer on|off;`
   **默认**： `client_body_in_single_buffer off;`
   **配置块**： `http、server、location`
   用户请求中的HTTP包体一律存储到内存buffer中。当然，如果HTTP包体的大小超过了下面client_body_buffer_size设置的值，包体还是会写入到磁盘文件中。
3. 存储HTTP头部的内存buffer大小
   **语法**： `client_header_buffer_size size;`
   **默认**： `client_header_buffer_size 1k;`
   **配置块**： `http、server`
   上面配置项定义了正常情况下Nginx接收用户请求中HTTP header部分（包括HTTP行和HTTP头部）时分配的内存buffer大小。有时，请求中的HTTP header部分可能会超过这个大小，这时large_client_header_buffers定义的buffer将会生效。
4. 存储超大HTTP头部的内存buffer大小
   **语法**： `large_client_header_buffers number size;`
   **默认**： `large_client_header_buffers 48k;`
   **配置块**： `http、server`
   large_client_header_buffers定义了Nginx接收一个超大HTTP头部请求的buffer个数和每个buffer的大小。如果HTTP请求行（如GET/index HTTP/1.1）的大小超过上面的单个buffer，则返回"Request URI too large"(414)。请求中一般会有许多header，每一个header的大小也不能超过单个buffer的大小，否则会返回"Bad request"(400)。当然，请求行和请求头部的总和也不可以超过buffer个数*buffer大小。
5. 存储HTTP包体的内存buffer大小
   **语法**： `client_body_buffer_size size;`
   **默认**： `client_body_buffer_size 8k/16k;`
   **配置块**： `http、server、location`
   上面配置项定义了Nginx接收HTTP包体的内存缓冲区大小。也就是说，HTTP包体会先接收到指定的这块缓存中，之后才决定是否写入磁盘。
   { % note color:light 注意 如果用户请求中含有HTTP头部Content-Length，并且其标识的长度小于定义的buffer大小，那么Nginx会自动降低本次请求所使用的内存buffer，以降低内存消耗。 %}
6. HTTP包体的临时存放目录
   **语法**： `client_body_temp_path dir-path[level1[level2[level3]]]`
   **默认**： `client_body_temp_path client_body_temp;`
   **配置块**： `http、server、location`
   上面配置项定义HTTP包体存放的临时目录。在接收HTTP包体时，如果包体的大小大于client_body_buffer_size，则会以一个递增的整数命名并存放到client_body_temp_path指定的目录中。后面跟着的level1、level2、level3，是为了防止一个目录下的文件数量太多，从而导致性能下降，因此使用了level参数，这样可以按照临时文件名最多再加三层目录。例如：
   ```text
   client_body_temp_path /opt/nginx/client_temp 1 2;
   ```
   如果新上传的HTTP包体使用00000123456作为临时文件名，就会被存放在这个目录中。/opt/nginx/client_temp/6/45/00000123456
7. connection_pool_size
   **语法**： `connection_pool_size size;`
   **默认**： `connection_pool_size 256;`
   **配置块**： `http、server`
   Nginx对于每个建立成功的TCP连接会预先分配一个内存池，上面的size配置项将指定这个内存池的初始大小（即ngx_connection_t结构体中的pool内存池初始大小，9.8.1节将介绍这个内存池是何时分配的），用于减少内核对于小块内存的分配次数。需慎重设置，因为更大的size会使服务器消耗的内存增多，而更小的size则会引发更多的内存分配次数。
8. request_pool_size
   **语法**： `request_pool_size size;`
   **默认**： `request_pool_size 4k;`
   **配置块**： `http、server`
   Nginx开始处理HTTP请求时，将会为每个请求都分配一个内存池，size配置项将指定这个内存池的初始大小（即ngx_http_request_t结构体中的pool内存池初始大小，11.3节将介绍这个内存池是何时分配的），用于减少内核对于小块内存的分配次数。TCP连接关闭时会销毁connection_pool_size指定的连接内存池，HTTP请求结束时会销毁request_pool_size指定的HTTP请求内存池，但它们的创建、销毁时间并不一致，因为一个TCP连接可能被复用于多个HTTP请求。8.7节会详述内存池原理。

### 2.4.4 网络连接的设置
下面介绍网络连接的设置配置项。
1. 读取HTTP头部的超时时间
   **语法**： `client_header_timeout time（默认单位：秒）;`
   **默认**： `client_header_timeout 60;`
   **配置块**： `http、server、location`
   客户端与服务器建立连接后将开始接收HTTP头部，在这个过程中，如果在一个时间间隔（超时时间）内没有读取到客户端发来的字节，则认为超时，并向客户端返回408("Request timed out")响应。
2. 读取HTTP包体的超时时间
   **语法**： `client_body_timeout time`（默认单位：秒）；
   **默认**： `client_body_timeout 60;`
   **配置块**： `http、server、location`
   此配置项与client_header_timeout相似，只是这个超时时间只在读取HTTP包体时才有效。
3. 发送响应的超时时间
   **语法**： `send_timeout time;`
   **默认**： `send_timeout 60;`
   **配置块**： `http、server、location`
   这个超时时间是发送响应的超时时间，即Nginx服务器向客户端发送了数据包，但客户端一直没有去接收这个数据包。如果某个连接超过send_timeout定义的超时时间，那么Nginx将会关闭这个连接。
4. reset_timeout_connection
   **语法**： `reset_timeout_connection on|off;`
   **默认**： `reset_timeout_connection off;`
   **配置块**： `http、server、location`
   连接超时后将通过向客户端发送RST包来直接重置连接。这个选项打开后，Nginx会在某个连接超时后，不是使用正常情形下的四次握手关闭TCP连接，而是直接向用户发送RST重置包，不再等待用户的应答，直接释放Nginx服务器上关于这个套接字使用的所有缓存（如TCP滑动窗口）。相比正常的关闭方式，它使得服务器避免产生许多处于FIN_WAIT_1、FIN_WAIT_2、TIME_WAIT状态的TCP连接。
   {% note color:light 注意，使用RST重置包关闭连接会带来一些问题，默认情况下不会开启。 %}
5. lingering_close
   **语法**： `lingering_close off|on|always;`
   **默认**： `lingering_close on;`
   **配置块**： `http、server、location`
   该配置控制Nginx关闭用户连接的方式。always表示关闭用户连接前必须无条件地处理连接上所有用户发送的数据。off表示关闭连接时完全不管连接上是否已经有准备就绪的来自用户的数据。on是中间值，一般情况下在关闭连接前都会处理连接上的用户发送的数据，除了有些情况下在业务上认定这之后的数据是不必要的。
6. lingering_time
   **语法**： `lingering_time time;`
   **默认**： `lingering_time 30s;`
   **配置块**： `http、server、location`
   lingering_close启用后，这个配置项对于上传大文件很有用。上文讲过，当用户请求的Content-Length大于max_client_body_size配置时，Nginx服务会立刻向用户发送413（Request entity too large）响应。但是，很多客户端可能不管413返回值，仍然持续不断地上传HTTP body，这时，经过了lingering_time设置的时间后，Nginx将不管用户是否仍在上传，都会把连接关闭掉。
7. lingering_timeout
   **语法**： `lingering_timeout time;`
   **默认**： `lingering_timeout 5s;`
   **配置块**： `http、server、location`
   lingering_close生效后，在关闭连接前，会检测是否有用户发送的数据到达服务器，如果超过lingering_timeout时间后还没有数据可读，就直接关闭连接；否则，必须在读取完连接缓冲区上的数据并丢弃掉后才会关闭连接。
8. 对某些浏览器禁用keepalive功能
   **语法**： `keepalive_disable[msie6|safari|none]...`
   **默认**： `keepalive_disablemsie6 safari`
   **配置块**： `http、server、location`
   HTTP请求中的keepalive功能是为了让多个请求复用一个HTTP长连接，这个功能对服务器的性能提高是很有帮助的。但有些浏览器，如IE 6和Safari，它们对于使用keepalive功能的POST请求处理有功能性问题。因此，针对IE 6及其早期版本、Safari浏览器默认是禁用keepalive功能的。
9. keepalive超时时间
   **语法**： `keepalive_timeout time（默认单位：秒）;`
   **默认**： `keepalive_timeout 75;`
   **配置块**： `http、server、location`
   一个keepalive连接在闲置超过一定时间后（默认的是75秒），服务器和浏览器都会去关闭这个连接。当然，keepalive_timeout配置项是用来约束Nginx服务器的，Nginx也会按照规范把这个时间传给浏览器，但每个浏览器对待keepalive的策略有可能是不同的。
10. 一个keepalive长连接上允许承载的请求最大数
    **语法**： `keepalive_requests n;`
    **默认**： `keepalive_requests 100;`
    **配置块**： `http、server、location`
    一个keepalive连接上默认最多只能发送100个请求。
11. tcp_nodelay
    **语法**： `tcp_nodelay on|off;`
    **默认**： `tcp_nodelay on;`
    **配置块**： `http、server、location`
    确定对keepalive连接是否使用TCP_NODELAY选项。
12. tcp_nopush
    **语法**： `tcp_nopush on|off;`
    **默认**： `tcp_nopush off;`
    **配置块**： `http、server、location`
    在打开sendfile选项时，确定是否开启FreeBSD系统上的TCP_NOPUSH或Linux系统上的TCP_CORK功能。打开tcp_nopush后，将会在发送响应时把整个响应包头放到一个TCP包中发送。

### 2.4.5 MIME类型的设置
下面是MIME类型的设置配置项。
- MIME type与文件扩展的映射
  **语法**： `type{...};`
  **配置块**： `http、server、location`
  定义MIME type到文件扩展名的映射。多个扩展名可以映射到同一个MIME type。例如：
   ```text
   types {
      text/html html;
      text/html conf;
      image/gif gif;
      image/jpeg jpg;
   }
   ```
- 默认MIME type
  **语法**： `default_type MIME-type;`
  **默认**： `default_type text/plain;`
  **配置块**： `http、server、location`
  当找不到相应的MIME type与文件扩展名之间的映射时，使用默认的MIME type作为HTTP header中的Content-Type。
- types_hash_bucket_size
  **语法**： `types_hash_bucket_size size;`
  **默认**： `types_hash_bucket_size 32|64|128;`
  **配置块**： `http、server、location`
  为了快速寻找到相应MIME type，Nginx使用散列表来存储MIMEtype与文件扩展名。types_hash_bucket_size设置了每个散列桶占用的内存大小。
- types_hash_max_size
  **语法**： `types_hash_max_size size;`
  **默认**： `types_hash_max_size 1024;`
  **配置块**： `http、server、location`
  types_hash_max_size影响散列表的冲突率。types_hash_max_size越大，就会消耗更多的内存，但散列key的冲突率会降低，检索速度就更快。types_hash_max_size越小，消耗的内存就越小，但散列key的冲突率可能上升。
### 2.4.6 对客户端请求的限制
下面介绍对客户端请求的限制的配置项。
1. 按HTTP方法名限制用户请求
   **语法**： `limit_except method...{...}`
   **配置块**： `location`
   Nginx通过limit_except后面指定的方法名来限制用户请求。方法名可取值包括：GET、HEAD、POST、PUT、DELETE、MKCOL、COPY、MOVE、OPTIONS、PROPFIND、PROPPATCH、LOCK、UNLOCK或者PATCH。例如：
   ```text
   limit_except GET {
      allow 192.168.1.0/32;
      deny all;
   }
   ```
   {% note color:light 注意，允许GET方法就意味着也允许HEAD方法。因此，上面这段代码表示的是禁止GET方法和HEAD方法，但其他HTTP方法是允许的。 %}
2. HTTP请求包体的最大值
   **语法**： `client_max_body_size size;`
   **默认**： `client_max_body_size 1m;`
   **配置块**： `http、server、location`
   浏览器在发送含有较大HTTP包体的请求时，其头部会有一个Content-Length字段，client_max_body_size是用来限制Content-Length所示值的大小的。因此，这个限制包体的配置非常有用处，因为不用等Nginx接收完所有的HTTP包体——这有可能消耗很长时间——就可以告诉用户请求过大不被接受。例如，用户试图上传一个10GB的文件，Nginx在收完包头后，发现Content-Length超过client_max_body_size定义的值，就直接发送413("Request Entity Too Large")响应给客户端。
3. 对请求的限速
   **语法**： `limit_rate speed;`
   **默认**： `limit_rate 0;`
   **配置块**： `http、server、location、if`
   此配置是对客户端请求限制每秒传输的字节数。speed可以使用2.2.4节中提到的多种单位，默认参数为0，表示不限速。
   针对不同的客户端，可以用$limit_rate参数执行不同的限速策略。
   例如：
   ```text
   server {
      if ($slow) {
          set $limit_rate 4k;
      }
   } 
   ```
4. limit_rate_after
   **语法**： `limit_rate_after time;`
   **默认**： `limit_rate_after 1m;`
   **配置块**： `http、server、location、if`
   此配置表示Nginx向客户端发送的响应长度超过limit_rate_after后才开始限速。例如：
   ```text
   limit_rate_after 1m;
   limit_rate 100k;
   ```
   11.9.2节将从源码上介绍limit_rate_after与limit_rate的区别，以及HTTP框架是如何使用它们来限制发送响应速度的。

### 2.4.7 文件操作的优化
下面介绍文件操作的优化配置项。
1. sendfile系统调用
   **语法**： `sendfile on|off;`
   **默认**： `sendfile off;`
   **配置块**： `http、server、location`
   可以启用Linux上的sendfile系统调用来发送文件，它减少了内核态与用户态之间的两次内存复制，这样就会从磁盘中读取文件后直接在内核态发送到网卡设备，提高了发送文件的效率。
2. AIO系统调用
   **语法**： `aio on|off;`
   **默认**： `aio off;`
   **配置块**： `http、server、location`
   此配置项表示是否在FreeBSD或Linux系统上启用内核级别的异步文件I/O功能。注意，它与sendfile功能是互斥的。
3. directio
   **语法**： `directio size|off;`
   **默认**： `directio off;`
   **配置块**： `http、server、location`
   此配置项在FreeBSD和Linux系统上使用O_DIRECT选项去读取文件，缓冲区大小为size，通常对大文件的读取速度有优化作用。注意，它与sendfile功能是互斥的。
4. directio_alignment
   **语法**： `directio_alignment size;`
   **默认**： `directio_alignment 512;`
   **配置块**： `http、server、location`
   它与directio配合使用，指定以directio方式读取文件时的对齐方式。一般情况下，512B已经足够了，但针对一些高性能文件系统，如Linux下的XFS文件系统，可能需要设置到4KB作为对齐方式。
5. 打开文件缓存
   **语法**： `open_file_cache max=N[inactive=time]|off;`
   **默认**： `open_file_cache off;`
   **配置块**： `http、server、location`
   文件缓存会在内存中存储以下3种信息：
   - 文件句柄、文件大小和上次修改时间。 
   - 已经打开过的目录结构。 
   - 没有找到的或者没有权限操作的文件信息。
   这样，通过读取缓存就减少了对磁盘的操作。<br>
   该配置项后面跟3种参数。
   - max：表示在内存中存储元素的最大个数。当达到最大限制数量后，将采用LRU（Least Recently Used）算法从缓存中淘汰最近最少使用的元素。
   - inactive：表示在inactive指定的时间段内没有被访问过的元素将会被淘汰。默认时间为60秒。
   - off：关闭缓存功能。
   例如：
   ```text
   open_file_cache max=1000 inactive=20s;
   ```
6. 是否缓存打开文件错误的信息
**语法**： `open_file_cache_errors on|off;`
**默认**： `open_file_cache_errors off;`
**配置块**： `http、server、location`
此配置项表示是否在文件缓存中缓存打开文件时出现的找不到路径、没有权限等错误信息。
7. 不被淘汰的最小访问次数
   **语法**： `open_file_cache_min_uses number;`
   **默认**： `open_file_cache_min_uses 1;`
   **配置块**： `http、server、location`
   它与open_file_cache中的inactive参数配合使用。如果在inactive指定的时间段内，访问次数超过了open_file_cache_min_uses指定的最小次数，那么将不会被淘汰出缓存。
8. 检验缓存中元素有效性的频率
   **语法**： `open_file_cache_valid time;`
   **默认**： `open_file_cache_valid 60s;`
   **配置块**： `http、server、location`
   默认为每60秒检查一次缓存中的元素是否仍有效。

### 2.4.8 对客户端请求的特殊处理
下面介绍对客户端请求的特殊处理的配置项。
1. 忽略不合法的HTTP头部
   **语法**： `ignore_invalid_headers on|off;`
   **默认**： `ignore_invalid_headers on;`
   **配置块**： `http、server`
   如果将其设置为off，那么当出现不合法的HTTP头部时，Nginx会拒绝服务，并直接向用户发送400（Bad Request）错误。如果将其设置为on，则会忽略此HTTP头部。
2. HTTP头部是否允许下划线
   **语法**： `underscores_in_headers on|off;`
   **默认**： `underscores_in_headers off;`
   **配置块**： `http、server`
   默认为off，表示HTTP头部的名称中不允许带“_”（下划线）。
3. 对If-Modified-Since头部的处理策略
   **语法**： `if_modified_since[off|exact|before];`
   **默认**： `if_modified_since exact;`
   **配置块**： `http、server、location`
   出于性能考虑，Web浏览器一般会在客户端本地缓存一些文件，并存储当时获取的时间。这样，下次向Web服务器获取缓存过的资源时，就可以用If-Modified-Since头部把上次获取的时间捎带上，而if_modified_since将根据后面的参数决定如何处理If-Modified-Since头部。<br>
   相关参数说明如下。
   - off：表示忽略用户请求中的If-Modified-Since头部。这时，如果获取一个文件，那么会正常地返回文件内容。HTTP响应码通常是200。 
   - exact：将If-Modified-Since头部包含的时间与将要返回的文件上次修改的时间做精确比较，如果没有匹配上，则返回200和文件的实际内容，如果匹配上，则表示浏览器缓存的文件内容已经是最新的了，没有必要再返回文件从而浪费时间与带宽了，这时会返回304 Not Modified，浏览器收到后会直接读取自己的本地缓存。 
   - before：是比exact更宽松的比较。只要文件的上次修改时间等于或者早于用户请求中的If-Modified-Since头部的时间，就会向客户端返回304 Not Modified。 
4. 文件未找到时是否记录到error日志
   **语法**： `log_not_found on|off;`
   **默认**： `log_not_found on;`
   **配置块**： `http、server、location`
   此配置项表示当处理用户请求且需要访问文件时，如果没有找到文件，是否将错误日志记录到error.log文件中。这仅用于定位问题。
5. merge_slashes
   **语法**： `merge_slashes on|off;`
   **默认**： `merge_slashes on;`
   **配置块**： `http、server、location`
   此配置项表示是否合并相邻的“/”，例如，//test///a.txt，在配置为on时，会将其匹配为location/test/a.txt；如果配置为off，则不会匹配，URI将仍然是//test///a.txt。 
6. DNS解析地址
   **语法**： `resolver address...;`
   **配置块**： `http、server、location`
   设置DNS名字解析服务器的地址，例如：
   ```text
   resolver 127.0.0.1 192.0.2.1; 
   ```
7. DNS解析的超时时间
   **语法**： `resolver_timeout time;`
   **默认**： `resolver_timeout 30s;`
   **配置块**： `http、server、location`
   此配置项表示DNS解析的超时时间。 
8. 返回错误页面时是否在Server中注明Nginx版本
   **语法**： `server_tokens on|off;`
   **默认**： `server_tokens on;`
   **配置块**： `http、server、location`
   表示处理请求出错时是否在响应的Server头部中标明Nginx版本，这是为了方便定位问题。

### 2.4.9 ngx_http_core_module模块提供的变量
在记录access_log访问日志文件时，可以使用ngx_http_core_module模块处理请求时所产生的丰富的变量，当然，这些变量还可以用于其他HTTP模块。例如，当URI中的某个参数满足设定的条件时，有些HTTP模块的配置项可以使用类似$arg_PARAMETER这样的变量。又如，若想把每个请求中的限速信息记录到access日志文件中，则可以使用$limit_rate变量。<br>
表2-1列出了ngx_http_core_module模块提供的这些变量。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/chapter_02/nginx-chapter02-003.png 表2-1 ngx_http_core_module模块提供的变量 %}
## 2.5 用HTTP proxy module配置一个反向代理服务器
反向代理（reverse proxy）方式是指用代理服务器来接受Internet上的连接请求，然后将请求转发给内部网络中的上游服务器，并将从上游服务器上得到的结果返回给Internet上请求连接的客户端，此时代理服务器对外的表现就是一个Web服务器。充当反向代理服务器也是Nginx的一种常见用法（反向代理服务器必须能够处理大量并发请求），本节将介绍Nginx作为HTTP反向代理服务器的基本用法。<br>
由于Nginx具有“强悍”的高并发高负载能力，因此一般会作为前端的服务器直接向客户端提供静态文件服务。但也有一些复杂、多变的业务不适合放到Nginx服务器上，这时会用Apache、Tomcat等服务器来处理。于是，Nginx通常会被配置为既是静态Web服务器也是反向代理服务器（如图2-3所示），不适合Nginx处理的请求就会直接转发到上游服务器中处理。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/chapter_02/nginx-chapter02-004.png 图2-3 作为静态Web服务器与反向代理服务器的Nginx%}
与Squid等其他反向代理服务器相比，Nginx的反向代理功能有自己的特点，如图2-4所示。<br>
当客户端发来HTTP请求时，Nginx并不会立刻转发到上游服务器，而是先把用户的请求（包括HTTP包体）完整地接收到Nginx所在服务器的硬盘或者内存中，然后再向上游服务器发起连接，把缓存的客户端请求转发到上游服务器。而Squid等代理服务器则采用一边接收客户端请求，一边转发到上游服务器的方式。<br>
Nginx的这种工作方式有什么优缺点呢？很明显，缺点是延长了一个请求的处理时间，并增加了用于缓存请求内容的内存和磁盘空间。而优点则是降低了上游服务器的负载，尽量把压力放在Nginx服务器上。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/chapter_02/nginx-chapter02-005.png 图2-4 Nginx作为反向代理服务器时转发请求的流程 %}
Nginx的这种工作方式为什么会降低上游服务器的负载呢？通常，客户端与代理服务器之间的网络环境会比较复杂，多半是“走”公网，网速平均下来可能较慢，因此，一个请求可能要持续很久才能完成。而代理服务器与上游服务器之间一般是“走”内网，或者有专线连接，传输速度较快。Squid等反向代理服务器在与客户端建立连接且还没有开始接收HTTP包体时，就已经向上游服务器建立了连接。例如，某个请求要上传一个1GB的文件，那么每次Squid在收到一个TCP分包（如2KB）时，就会即时地向上游服务器转发。在接收客户端完整HTTP包体的漫长过程中，上游服务器始终要维持这个连接，这直接对上游服务器的并发处理能力提出了挑战。<br>
Nginx则不然，它在接收到完整的客户端请求（如1GB的文件）后，才会与上游服务器建立连接转发请求，由于是内网，所以这个转发过程会执行得很快。这样，一个客户端请求占用上游服务器的连接时间就会非常短，也就是说，Nginx的这种反向代理方案主要是为了降低上游服务器的并发压力。<br>
Nginx将上游服务器的响应转发到客户端有许多种方法，第12章将介绍其中常见的两种方式。
### 2.5.1 负载均衡的基本配置
作为代理服务器，一般都需要向上游服务器的集群转发请求。这里的负载均衡是指选择一种策略，尽量把请求平均地分布到每一台上游服务器上。下面介绍负载均衡的配置项。
1. upstream块 
   **语法**： `upstream name{...}`
   **配置块**： `http`
   upstream块定义了一个上游服务器的集群，便于反向代理中的proxy_pass使用。例如：
   ```text
   upstream backend {
       server backend1.example.com;
       server backend2.example.com;
       server backend3.example.com;
   }
   server {
       location / {
           proxy_pass http://backend;
       }
   } 
   ```
2. server
   **语法**： `server name[parameters];`
   **配置块**： `upstream`
   server配置项指定了一台上游服务器的名字，这个名字可以是域名、IP地址端口、UNIX句柄等，在其后还可以跟下列参数。
   - weight=number：设置向这台上游服务器转发的权重，默认为1。 
   - max_fails=number：该选项与fail_timeout配合使用，指在fail_timeout时间段内，如果向当前的上游服务器转发失败次数超过number，则认为在当前的fail_timeout时间段内这台上游服务器不可用。max_fails默认为1，如果设置为0，则表示不检查失败次数。
   - fail_timeout=time：fail_timeout表示该时间段内转发失败多少次后就认为上游服务器暂时不可用，用于优化反向代理功能。它与向上游服务器建立连接的超时时间、读取上游服务器的响应超时时间等完全无关。fail_timeout默认为10秒。
   - down：表示所在的上游服务器永久下线，只在使用ip_hash配置项时才有用。 
   - backup：在使用ip_hash配置项时它是无效的。它表示所在的上游服务器只是备份服务器，只有在所有的非备份上游服务器都失效后，才会向所在的上游服务器转发请求。 
     例如：
     ```text
     upstream backend {
         server backend1.example.com weight=5;
         server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
         server unix:/tmp/backend3;
     } 
     ```
3. ip_hash 
   **语法**： `ip_hash; `
   **配置块**： `upstream`
   在有些场景下，我们可能会希望来自某一个用户的请求始终落到固定的一台上游服务器中。例如，假设上游服务器会缓存一些信息，如果同一个用户的请求任意地转发到集群中的任一台上游服务器中，那么每一台上游服务器都有可能会缓存同一份信息，这既会造成资源的浪费，也会难以有效地管理缓存信息。ip_hash就是用以解决上述问题的，它首先根据客户端的IP地址计算出一个key，将key按照upstream集群里的上游服务器数量进行取模，然后以取模后的结果把请求转发到相应的上游服务器中。这样就确保了同一个客户端的请求只会转发到指定的上游服务器中。
   ip_hash与weight（权重）配置不可同时使用。如果upstream集群中有一台上游服务器暂时不可用，不能直接删除该配置，而是要down参数标识，确保转发策略的一贯性。例如：
   ```text
   upstream backend {
       ip_hash;
       server backend1.example.com;
       server backend2.example.com;
       server backend3.example.com down;
       server backend4.example.com;
   }
   ```
4. 记录日志时支持的变量
   如果需要将负载均衡时的一些信息记录到access_log日志中，那么在定义日志格式时可以使用负载均衡功能提供的变量，见表2-2。
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/chapter_02/nginx-chapter02-006.png 表2-2 访问上游服务器时可以使用的变量 %}
   例如，可以在定义access_log访问日志格式时使用表2-2中的变量。
   ```text
   log_format timing '$remote_addr - $remote_user [$time_local] $request '
             'upstream_response_time $upstream_response_time '
             'msec $msec request_time $request_time';
   log_format up_head '$remote_addr - $remote_user [$time_local] $request '
             'upstream_http_content_type $upstream_http_content_type';
   ```
          
### 2.5.2 反向代理的基本配置
下面介绍反向代理的基本配置项。 
1. proxy_pass 
   **语法**： `proxy_pass URL;`
   **配置块**： `location、if`
   此配置项将当前请求反向代理到URL参数指定的服务器上，URL可以是主机名或IP地址加端口的形式，例如：
   ```text
   proxy_pass http://localhost:8000/uri/;
   ```
   也可以是UNIX句柄：
   ```text
   proxy_pass http://unix:/path/to/backend.socket:/uri/ ;
   ```
   还可以如上节负载均衡中所示，直接使用upstream块，例如：
   ```text
   upstream backend {
         ... 
   }
   server {
         location / {
               proxy_pass http://backend;
         }
   }
   ```
   用户可以把HTTP转换成更安全的HTTPS，例如：
   ```text
   proxy_pass https://192.168.0.1;
   ```
   默认情况下反向代理是不会转发请求中的Host头部的。如果需要转发，那么必须加上配置：
   ```text
   proxy_set_header Host $host;
   ```
2. proxy_method 
   **语法**： `proxy_method method;`
   **配置块**： `http、server、location`
   此配置项表示转发时的协议方法名。例如设置为：
   ```text
   proxy_method POST;
   ```
   那么客户端发来的GET请求在转发时方法名也会改为POST。 
3. proxy_hide_header
   **语法**： `proxy_hide_header the_header;`
   **配置块**： `http、server、location`
   Nginx会将上游服务器的响应转发给客户端，但默认不会转发以下HTTP头部字段：Date、Server、X-Pad和X-Accel-*。使用proxy_hide_header后可以任意地指定哪些HTTP头部字段不能被转发。
   例如：
   ```text
   proxy_hide_header Cache-Control;
   proxy_hide_header MicrosoftOfficeWebServer;
   ```
4. proxy_pass_header 
   **语法**： `proxy_pass_header the_header;`
   **配置块**： `http、server、location`
   与proxy_hide_header功能相反，proxy_pass_header会将原来禁止转发的header设置为允许转发。例如：
   ```text
   proxy_pass_header X-Accel-Redirect;
   ```
5. proxy_pass_request_body 
   **语法**： `proxy_pass_request_body on|off;`
   **默认**： `proxy_pass_request_body on;`
   **配置块**： `http、server、location`
   作用为确定是否向上游服务器发送HTTP包体部分。
6. proxy_pass_request_headers 
   **语法**： `proxy_pass_request_headers on|off;`
   **默认**： `proxy_pass_request_headers on;`
   **配置块**： `http、server、location`
   作用为确定是否转发HTTP头部。
7. proxy_redirect
   **语法**： `proxy_redirect[default|off|redirect replacement];`
   **默认**： `proxy_redirect default;`
   **配置块**： `http、server、location`
   当上游服务器返回的响应是重定向或刷新请求（如HTTP响应码是301或者302）时，proxy_redirect可以重设HTTP头部的location或refresh字段。例如，如果上游服务器发出的响应是302重定向请求，location字段的URI是http://localhost:8000/two/some/uri/ ，那么在下面的配置情况下，实际转发给客户端的location是http://frontend/one/some/uri/ 。
   ```text
   proxy_redirect http://localhost:8000/two/
   http://frontend/one/;
   ```
   这里还可以使用ngx-http-core-module提供的变量来设置新的location字段。例如：
   ```text
   proxy_redirect http://localhost:8000/
   http://$host:$server_port/;
   ```
   也可以省略replacement参数中的主机名部分，这时会用虚拟主机名称来填充。例如：
   ```text
   proxy_redirect http://localhost:8000/two//one/;
   ```
   使用off参数时，将使location或者refresh字段维持不变。例如：
   ```text
   proxy_redirect off;
   ```
   使用默认的default参数时，会按照proxy_pass配置项和所属的location配置项重组发往客户端的location头部。例如，下面两种配置效果是一样的：
   ```text
   location /one/ {
         proxy_pass http://upstream:port/two/;
         proxy_redirect default;
   }
   location /one/ {
         proxy_pass http://upstream:port/two/;
         proxy_redirect http://upstream:port/two//one/;
   } 
   ```
8. proxy_next_upstream
   **语法**： `proxy_next_upstream[error|timeout|invalid_header|http_500|http_502|http_503|http_504|http_404|off];`
   **默认**： `proxy_next_upstream error timeout;`
   **配置块**： `http、server、location`
   此配置项表示当向一台上游服务器转发请求出现错误时，继续换一台上游服务器处理这个请求。前面已经说过，上游服务器一旦开始发送应答，Nginx反向代理服务器会立刻把应答包转发给客户端。因此，一旦Nginx开始向客户端发送响应包，之后的过程中若出现错误也是不允许换下一台上游服务器继续处理的。这很好理解，这样才可以更好地保证客户端只收到来自一个上游服务器的应答。 proxy_next_upstream的参数用来说明在哪些情况下会继续选择下一台上游服务器转发请求。
   - error：当向上游服务器发起连接、发送请求、读取响应时出错。 
   - timeout：发送请求或读取响应时发生超时。 
   - invalid_header：上游服务器发送的响应是不合法的。 
   - http_500：上游服务器返回的HTTP响应码是500。 
   - http_502：上游服务器返回的HTTP响应码是502。 
   - http_503：上游服务器返回的HTTP响应码是503。 
   - http_504：上游服务器返回的HTTP响应码是504。 
   - http_404：上游服务器返回的HTTP响应码是404。 
   - off：关闭proxy_next_upstream功能—出错就选择另一台上游服务器再次转发。

Nginx的反向代理模块还提供了很多种配置，如设置连接的超时时间、临时文件如何存储，以及最重要的如何缓存上游服务器响应等功能。这些配置可以通过阅读ngx_http_proxy_module模块的说明了解，只有深入地理解，才能实现一个高性能的反向代理服务器。本节只是介绍反向代理服务器的基本功能，在第12章中我们将会深入地探索upstream机制，到那时，读者也许会发现ngx_http_proxy_module模块只是使用upstream机制实现了反向代理功能而已。
## 2.6 小结
Nginx由少量的核心框架代码和许多模块组成，每个模块都有它独特的功能。因此，读者可以通过查看每个模块实现了什么功能，来了解Nginx可以帮我们做些什么。<br>
Nginx的Wiki网站（http://wiki.nginx.org/Modules ）上列出了官方提供的所有模块及配置项，仔细观察就会发现，这些配置项的语法与本章的内容都是很相近的，读者只需要弄清楚模块说明中每个配置项的意义即可。另外，网页http://wiki.nginx.org/3rdPartyModules 中列出了Wiki上已知的几十个第三方模块，同时读者还可以从搜索引擎上搜索到更多的第三方模块。了解每个模块的配置项用法，并在Nginx中使用这些模块，可以让Nginx做到更多。<br>
随着对本书的学习，读者会对Nginx模块的设计思路有深入的了解，也会渐渐熟悉如何编写一个模块。如果某个模块的实现与你的想法有出入，可以更改这个模块的源码，实现你期望的业务功能。如果所有的模块都没有你想要的功能，不妨自己重写一个定制的模块，也可以申请发布到Nginx网站上供大家分享。