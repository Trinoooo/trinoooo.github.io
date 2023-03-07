---
layout: wiki  # 使用wiki布局模板
wiki: 深入理解Nginx：模块开发与架构解析 # 这是项目名
title: 第一章 研究Nginx前的准备工作
order: 2
---
2012年，Nginx荣获年度云计算开发奖（2012 Cloud Award for Developer of the Year），并成长为世界第二大Web服务器。全世界流量最高的前1000名网站中，超过25%都使用Nginx来处理海量的互联网请求。Nginx已经成为业界高性能Web服务器的代名词。<br>
那么，什么是Nginx？它有哪些特点？我们选择Nginx的理由是什么？如何编译安装Nginx？这种安装方式背后隐藏的又是什么样的思想呢？本章将会回答上述问题。
## 1.1 Nginx是什么
人们在了解新事物时，往往习惯通过类比来帮助自己理解事物的概貌。那么，我们在学习Nginx时也采用同样的方式，先来看看Nginx的竞争对手——Apache、Lighttpd、Tomcat、Jetty、IIS，它们都是Web服务器，或者叫做WWW（World Wide Web）服务器，相应地也都具备Web服务器的基本功能：基于REST架构风格[1] ，以统一资源描述符（Uniform Resource Identifier，URI）或者统一资源定位符（Uniform Resource Locator，URL）作为沟通依据，通过HTTP为浏览器等客户端程序提供各种网络服务。然而，由于这些Web服务器在设计阶段就受到许多局限，例如当时的互联网用户规模、网络带宽、产品特点等局限，并且各自的定位与发展方向都不尽相同，使得每一款Web服务器的特点与应用场合都很鲜明。<br>
Tomcat和Jetty面向Java语言，先天就是重量级的Web服务器，它的性能与Nginx没有可比性，这里略过。<br>
IIS只能在Windows操作系统上运行。Windows作为服务器在稳定性与其他一些性能上都不如类UNIX操作系统，因此，在需要高性能Web服务器的场合下，IIS可能会被“冷落”。<br>
Apache的发展时期很长，而且是目前毫无争议的世界第一大Web服务器，图1-1中是12年来（2010~2012年）世界Web服务器的使用排名情况。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-001.png 图1-1 Netcraft对于644275754个站点31.4M个域名Web服务器使用情况的调查结果（2012年3月）%}
从图1-1中可以看出，Apache目前处于领先地位。<br>
Apache有许多优点，如稳定、开源、跨平台等，但它出现的时间太长了，在它兴起的年代，互联网的产业规模远远比不上今天，所以它被设计成了一个重量级的、不支持高并发的Web服务器。在Apache服务器上，如果有数以万计的并发HTTP请求同时访问，就会导致服务器上消耗大量内存，操作系统内核对成百上千的Apache进程做进程间切换也会消耗大量CPU资源，并导致HTTP请求的平均响应速度降低，这些都决定了Apache不可能成为高性能Web服务器，这也促使了Lighttpd和Nginx的出现。观察图1-1中Nginx成长的曲线，体会一下Nginx抢占市场时的“咄咄逼人”吧。<br>
Lighttpd和Nginx一样，都是轻量级、高性能的Web服务器，欧美的业界开发者比较钟爱Lighttpd，而国内的公司更青睐Nginx，Lighttpd使用得比较少。<br>
在了解了Nginx的竞争对手之后，相信大家对Nginx也有了直观感受，下面让我们来正式地认识一下Nginx吧。
{% note color:light 提示 Nginx发音：engineX %}
来自俄罗斯的Igor Sysoev在为Rambler Media（http://www.rambler.ru/ ）工作期间，使用C语言开发了Nginx。Nginx作为Web服务器，一直为俄罗斯著名的门户网站Rambler Media提供着出色、稳定的服务。<br>
Igor Sysoev将Nginx的代码开源，并且赋予其最自由的2-clause BSD-like license[2] 许可证。由于Nginx使用基于事件驱动的架构能够并发处理百万级别的TCP连接，高度模块化的设计和自由的许可证使得扩展Nginx功能的第三方模块层出不穷，而且优秀的设计带来了极佳的稳定性，因此其作为Web服务器被广泛应用到大流量的网站上，包括腾讯、新浪、网易、淘宝等访问量巨大的网站。<br>
2012年2月和3月Netcraft对Web服务器的调查如表1-1所示，可以看出，Nginx的市场份额越来越大。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-002.png 表1-1 Netcraft对于Web服务器市场占有率前4位软件的调查（2012年2月和3月） %}
Nginx是一个跨平台的Web服务器，可运行在Linux、FreeBSD、Solaris、AIX、Mac OS、Windows等操作系统上，并且它还可以使用当前操作系统特有的一些高效API来提高自己的性能。<br>
例如，对于高效处理大规模并发连接，它支持Linux上的epoll（epoll是Linux上处理大并发网络连接的利器，9.6.1节中将会详细说明epoll的工作原理）、Solaris上的event ports和FreeBSD上的kqueue等。<br>
又如，对于Linux，Nginx支持其独有的sendfile系统调用，这个系统调用可以高效地把硬盘中的数据发送到网络上（不需要先把硬盘数据复制到用户态内存上再发送），这极大地减少了内核态与用户态数据间的复制动作。种种迹象都表明，Nginx以性能为王。<br>
2011年7月，Nginx正式成立公司，由Igor Sysoev担任CTO，立足于提供商业级的Web服务器。<br>
[1] 参见Roy Fielding博士的论文《Architectural Styles and the Design of Network-based Software Architectures》，可在http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm查看原文。<br>
[2] BSD（Berkeley Software Distribution）许可协议是自由软件（开源 软件的一个子集）中使用最广泛的许可协议之一。与其他许可协议相 比，BSD许可协议从GNU通用公共许可协议（GPL）到限制重重的著 作权（copyright）都要宽松一些，事实上，它跟公有领域更为接近。BSD许可协议被认为是copycenter（中间版权），界于标准的copyright与GPL的copyleft之间。2-clause BSD-like license是BSD许可协议中最宽 松的一种，它对开发者再次使用BSD软件只有两个基本的要求：一是 如果再发布的产品中包含源代码，则在源代码中必须带有原来代码中 的BSD协议；二是如果再发布的只是二进制类库/软件，则需要在类库/软件的文档和版权声明中包含原来代码中的BSD协议。
## 1.2 为什么选择Nginx
为什么选择Nginx？因为它具有以下特点：
1. 更快
   这表现在两个方面：一方面，在正常情况下，单次请求会得到更快的响应；另一方面，在高峰期（如有数以万计的并发请求），Nginx可以比其他Web服务器更快地响应请求。<br>
   实际上，本书第三部分中大量的篇幅都是在说明Nginx是如何做到这两点的。
2. 高扩展性
   Nginx的设计极具扩展性，它完全是由多个不同功能、不同层次、不同类型且耦合度极低的模块组成。因此，当对某一个模块修复Bug或进行升级时，可以专注于模块自身，无须在意其他。而且在HTTP模块中，还设计了HTTP过滤器模块：一个正常的HTTP模块在处理完请求后，会有一串HTTP过滤器模块对请求的结果进行再处理。这样，当我们开发一个新的HTTP模块时，不但可以使用诸如HTTP核心模块、events模块、log模块等不同层次或者不同类型的模块，还可以原封不动地复用大量已有的HTTP过滤器模块。这种低耦合度的优秀设计，造就了Nginx庞大的第三方模块，当然，公开的第三方模块也如官方发布的模块一样容易使用。<br>
   Nginx的模块都是嵌入到二进制文件中执行的，无论官方发布的模块还是第三方模块都是如此。这使得第三方模块一样具备极其优秀的性能，充分利用Nginx的高并发特性，因此，许多高流量的网站都倾向于开发符合自己业务特性的定制模块。
3. 高可靠性
   高可靠性是我们选择Nginx的最基本条件，因为Nginx的可靠性是大家有目共睹的，很多家高流量网站都在核心服务器上大规模使用Nginx。Nginx的高可靠性来自于其核心框架代码的优秀设计、模块设计的简单性；另外，官方提供的常用模块都非常稳定，每个worker进程相对独立，master进程在1个worker进程出错时可以快速“拉起”新的worker子进程提供服务。
4. 低内存消耗
   一般情况下，10000个非活跃的HTTP Keep-Alive连接在Nginx中仅消耗2.5MB的内存，这是Nginx支持高并发连接的基础。<br>
   从第3章开始，我们会接触到Nginx在内存中为了维护一个HTTP连接所分配的对象，届时将会看到，实际上Nginx一直在为用户考虑（尤其是在高并发时）如何使得内存的消耗更少。
5. 单机支持10万以上的并发连接
   这是一个非常重要的特性！随着互联网的迅猛发展和互联网用户数量的成倍增长，各大公司、网站都需要应付海量并发请求，一个能够在峰值期顶住10万以上并发请求的Server，无疑会得到大家的青睐。理论上，Nginx支持的并发连接上限取决于内存，10万远未封顶。当然，能够及时地处理更多的并发请求，是与业务特点紧密相关的，本书第8~11章将会详细说明如何实现这个特点。
6. 热部署
   master管理进程与worker工作进程的分离设计，使得Nginx能够提供热部署功能，即可以在7×24小时不间断服务的前提下，升级Nginx的可执行文件。当然，它也支持不停止服务就更新配置项、更换日志文件等功能。
7. 最自由的BSD许可协议
   这是Nginx可以快速发展的强大动力。BSD许可协议不只是允许用户免费使用Nginx，它还允许用户在自己的项目中直接使用或修改Nginx源码，然后发布。这吸引了无数开发者继续为Nginx贡献自己的智慧。

以上7个特点当然不是Nginx的全部，拥有无数个官方功能模块、第三方功能模块使得Nginx能够满足绝大部分应用场景，这些功能模块间可以叠加以实现更加强大、复杂的功能，有些模块还支持Nginx与Perl、Lua等脚本语言集成工作，大大提高了开发效率。这些特点促使用户在寻找一个Web服务器时更多考虑Nginx。当然，选择Nginx的核心理由还是它能在支持高并发请求的同时保持高效的服务。如果Web服务器的业务访问量巨大，就需要保证在数以百万计的请求同时访问服务时，用户可以获得良好的体验，不会出现并发访问量达到一个数字后，新的用户无法获取服务，或者虽然成功地建立起了TCP连接，但大部分请求却得不到响应的情况。通常，高峰期服务器的访问量可能是正常情况下的许多倍，若有热点事件的发生，可能会导致正常情况下非常顺畅的服务器直接“挂 死”。然而，如果在部署服务器时，就预先针对这种情况进行扩容，又会使得正常情况下所有服务器的负载过低，这会造成大量的资源浪费。因此，我们会希望在这之间取得平衡，也就是说，在低并发压力下，用户可以获得高速体验，而在高并发压力下，更多的用户都能接入，可能访问速度会下降，但这只应受制于带宽和处理器的速度，而不应该是服务器设计导致的软件瓶颈。事实上，由于中国互联网用户群体的数量巨大，致使对Web服务器的设计往往要比欧美公司更加困难。例如，对于全球性的一些网站而言，欧美用户分布在两个半球，欧洲用户活跃时，美洲用户通常在休息，反之亦然。而国内巨大的用户群体则对业界的程序员提出更高的挑战，早上9点和晚上20点到24点这些时间段的并发请求压力是非常巨大的。尤其节假日、寒暑假到来之时，更会对服务器提出极高的要求。<br>
另外，国内业务上的特性，也会引导用户在同一时间大并发地访问服务器。例如，许多SNS网页游戏会在固定的时间点刷新游戏资源或者允许“偷菜”等好友互动操作。这些会导致服务器处理高并发请求的压力增大。<br>
上述情形都对我们的互联网服务在大并发压力下是否还能够给予用户良好的体验提出了更高的要求。若要提供更好的服务，那么可以从多方面入手，例如，修改业务特性、引导用户从高峰期分流或者把服务分层分级、对于不同并发压力给用户提供不同级别的服务等。但最根本的是，Web服务器要能支持大并发压力下的正常服务，这才是关键。<br>
快速增长的互联网用户群以及业内所有互联网服务提供商越来越好的用户体验，都促使我们在大流量服务中用Nginx取代其他Web服务器。Nginx先天的事件驱动型设计、全异步的网络I/O处理机制、极少的进程间切换以及许多优化设计，都使得Nginx天生善于处理高并发压力下的互联网请求，同时Nginx降低了资源消耗，可以把服务器硬件资源“压榨”到极致。
## 1.3 准备工作
由于Linux具有免费、使用广泛、商业支持越来越完善等特点，本书将主要针对Linux上运行的Nginx来进行介绍。需要说明的是，本书不是使用手册，而是介绍Nginx作为Web服务器的设计思想，以及如何更有效地使用Nginx达成目的，而这些内容在各操作系统上基本是相通的（除了第9章关于事件驱动方式以及第14章的进程间同步方式在类UNIX操作系统上略有不同以外）。
### 1.3.1 Linux操作系统
首先我们需要一个内核为Linux 2.6及以上版本的操作系统，因为Linux 2.6及以上内核才支持epoll，而在Linux上使用select或poll来解决事件的多路复用，是无法解决高并发压力问题的。<br>
我们可以使用uname-a命令来查询Linux内核版本，例如：
```shell
:wehf2wng001:root > uname -a
Linux wehf2wng001 2.6.18-128.el5 #1 SMP Wed Jan 21 10:41:14 EST 2009 x86_64 x86_64 x86_64 GNU/Linux
```
执行结果表明内核版本是2.6.18，符合我们的要求。
### 1.3.2 使用Nginx的必备软件
如果要使用Nginx的常用功能，那么首先需要确保该操作系统上至少安装了如下软件。
1. GCC编译器
   GCC（GNU Compiler Collection）可用来编译C语言程序。Nginx不会直接提供二进制可执行程序（1.2.x版本中已经开始提供某些操作系统上的二进制安装包了，不过，本书探讨如何开发Nginx模块是必须通过直接编译源代码进行的），这有许多原因，本章后面会详述。我们可以使用最简单的yum方式安装GCC，例如：
   ```shell
   yum install -y gcc
   ```
   GCC是必需的编译工具。在第3章会提到如何使用C++来编写Nginx HTTP模块，这时就需要用到G++编译器了。G++编译器也可以用yum安装，例如：
   ```shell
   yum install -y gcc-c++
   ```
   Linux上有许多软件安装方式，yum只是其中比较方便的一种，其他方式这里不再赘述。
2. PCRE库
   PCRE（Perl Compatible Regular Expressions，Perl兼容正则表达式）是由Philip Hazel开发的函数库，目前为很多软件所使用，该库支持正则表达式。它由RegEx演化而来，实际上，Perl正则表达式也是源自于Henry Spencer写的RegEx。<br>
   如果我们在配置文件nginx.conf里使用了正则表达式，那么在编译Nginx时就必须把PCRE库编译进Nginx，因为Nginx的HTTP模块要靠它来解析正则表达式。当然，如果你确认不会使用正则表达式，就不必安装它。其yum安装方式如下：
   ```shell
   yum install -y pcre pcre-devel
   ```
   pcre-devel是使用PCRE做二次开发时所需要的开发库，包括头文件等，这也是编译Nginx所必须使用的。
3. zlib库
   zlib库用于对HTTP包的内容做gzip格式的压缩，如果我们在nginx.conf里配置了gzip on，并指定对于某些类型（content-type）的HTTP响应使用gzip来进行压缩以减少网络传输量，那么，在编译时就必须把zlib编译进Nginx。其yum安装方式如下：
   ```shell
   yum install -y zlib zlib-devel
   ```
   同理，zlib是直接使用的库，zlib-devel是二次开发所需要的库。
4. OpenSSL开发库
   如果我们的服务器不只是要支持HTTP，还需要在更安全的SSL协议上传输HTTP，那么就需要拥有OpenSSL了。另外，如果我们想使用MD5、SHA1等散列函数，那么也需要安装它。其yum安装方式如下：
   ```shell
   yum install -y openssl openssl-devel
   ```
   上面所列的4个库只是完成Web服务器最基本功能所必需的。Nginx是高度自由化的Web服务器，它的功能是由许多模块来支持的。而这些模块可根据我们的使用需求来定制，如果某些模块不需要使用则完全不必理会它。同样，如果使用了某个模块，而这个模块使用了一些类似zlib或OpenSSL等的第三方库，那么就必须先安装这些软件。

### 1.3.3 磁盘目录
要使用Nginx，还需要在Linux文件系统上准备以下目录。
1. Nginx源代码存放目录
   该目录用于放置从官网上下载的Nginx源码文件，以及第三方或我们自己所写的模块源代码文件。
2. Nginx编译阶段产生的中间文件存放目录
   该目录用于放置在configure命令执行后所生成的源文件及目录，以及make命令执行后生成的目标文件和最终连接成功的二进制文件。默认情况下，configure命令会将该目录命名为objs，并放在Nginx源代码目录下。
3. 部署目录
   该目录存放实际Nginx服务运行期间所需要的二进制文件、配置文件等。默认情况下，该目录为/usr/local/nginx。
4. 日志文件存放目录
   日志文件通常会比较大，当研究Nginx的底层架构时，需要打开debug级别的日志，这个级别的日志非常详细，会导致日志文件的大小增长得极快，需要预先分配一个拥有更大磁盘空间的目录。

### 1.3.4 Linux内核参数的优化
由于默认的Linux内核参数考虑的是最通用的场景，这明显不符合用于支持高并发访问的Web服务器的定义，所以需要修改Linux内核参数，使得Nginx可以拥有更高的性能。<br>
在优化内核时，可以做的事情很多，不过，我们通常会根据业务特点来进行调整，当Nginx作为静态Web内容服务器、反向代理服务器或是提供图片缩略图功能（实时压缩图片）的服务器时，其内核参数的调整都是不同的。这里只针对最通用的、使Nginx支持更多并发请求的TCP网络参数做简单说明。<br>
首先，需要修改/etc/sysctl.conf来更改内核参数。例如，最常用的配置：
```plain
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
net.ipv4.tcp_max_syn.backlog=1024
```
然后执行sysctl-p命令，使上述修改生效。<br>
上面的参数意义解释如下：
- file-max：这个参数表示进程（比如一个worker进程）可以同时打开的最大句柄数，这个参数直接限制最大并发连接数，需根据实际情况配置。
- tcp_tw_reuse：这个参数设置为1，表示允许将TIME-WAIT状态的socket重新用于新的TCP连接，这对于服务器来说很有意义，因为服务器上总会有大量TIME-WAIT状态的连接。
- tcp_keepalive_time：这个参数表示当keepalive启用时，TCP发送keepalive消息的频度。默认是2小时，若将其设置得小一些，可以更快地清理无效的连接。
- tcp_fin_timeout：这个参数表示当服务器主动关闭连接时，socket保持在FIN-WAIT-2状态的最大时间。
- tcp_max_tw_buckets：这个参数表示操作系统允许TIME_WAIT套接字数量的最大值，如果超过这个数字，TIME_WAIT套接字将立刻被清除并打印警告信息。该参数默认为180000，过多的TIME_WAIT套接字会使Web服务器变慢。
- tcp_max_syn_backlog：这个参数表示TCP三次握手建立阶段接收SYN请求队列的最大长度，默认为1024，将其设置得大一些可以使出现Nginx繁忙来不及accept新连接的情况时，Linux不至于丢失客户端发起的连接请求。
- ip_local_port_range：这个参数定义了在UDP和TCP连接中本地（不包括连接的远端）端口的取值范围。
- net.ipv4.tcp_rmem：这个参数定义了TCP接收缓存（用于TCP接收滑动窗口）的最小值、默认值、最大值。
- net.ipv4.tcp_wmem：这个参数定义了TCP发送缓存（用于TCP发送滑动窗口）的最小值、默认值、最大值。
- netdev_max_backlog：当网卡接收数据包的速度大于内核处理的速度时，会有一个队列保存这些数据包。这个参数表示该队列的最大值。
- rmem_default：这个参数表示内核套接字接收缓存区默认的大小。
- wmem_default：这个参数表示内核套接字发送缓存区默认的大小。
- rmem_max：这个参数表示内核套接字接收缓存区的最大大小。
- wmem_max：这个参数表示内核套接字发送缓存区的最大大小。 
  {% note color:light 注意 滑动窗口的大小与套接字缓存区会在一定程度上影响并发连接的数目。每个TCP连接都会为维护TCP滑动窗口而消耗内存，这个窗口会根据服务器的处理速度收缩或扩张。参数wmem_max的设置，需要平衡物理内存的总大小、Nginx并发处理的最大连接数量（由nginx.conf中的worker_processes和worker_connections参数决定）而确定。当然，如果仅仅为了提高并发量使服务器不出现Out Of Memory问题而去降低滑动窗口大小，那么并不合适，因为滑动窗口过小会影响大数据量的传输速度。rmem_default、wmem_default、rmem_max、wmem_max这4个参数的设置需要根据我们的业务特性以及实际的硬件成本来综合考虑。 %}
- tcp_syncookies：该参数与性能无关，用于解决TCP的SYN攻击。

### 1.3.5 获取Nginx源码
可以在Nginx官方网站（http://nginx.org/en/download.html ）获取Nginx源码包。将下载的nginx-1.0.14.tar.gz源码压缩包放置到准备好的Nginx源代码目录中，然后解压。例如：
```shell
tar -zxvf nginx-1.0.14.tar.gz
```
本书编写时的Nginx最新稳定版本为1.0.14（如图1-2所示），本书后续部分都将以此版本作为基准。当然，本书将要说明的Nginx核心代码一般不会有改动（否则大量第三方模块的功能就无法保证了），即使下载其他版本的Nginx源码包也不会影响阅读本书。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-003.png 图1-2 Nginx的不同版本 %}
## 1.4 编译安装Nginx
安装Nginx最简单的方式是，进入nginx-1.0.14目录后执行以下3行命令：
```shell
./configure
make
make install
```
configure命令做了大量的“幕后”工作，包括检测操作系统内核和已经安装的软件，参数的解析，中间目录的生成以及根据各种参数生成一些C源码文件、Makefile文件等。<br>
make命令根据configure命令生成的Makefile文件编译Nginx工程，并生成目标文件、最终的二进制文件。<br>
make install命令根据configure执行时的参数将Nginx部署到指定的安装目录，包括相关目录的建立和二进制文件、配置文件的复制。
## 1.5 configure详解
可以看出，configure命令至关重要，下文将详细介绍如何使用configure命令，并分析configure到底是如何工作的，从中我们也可以看出Nginx的一些设计思想。
### 1.5.1 configure的命令参数
#### 1.路径相关的参数
表1-2列出了Nginx在编译期、运行期中与路径相关的各种参数。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-004.png 表1-2 configure支持的路径相关参数 %}
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-005.png %}
#### 2.编译相关的参数
表1-3列出了编译Nginx时与编译器相关的参数。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-006.png 表1-3 configure支持的编译相关参数 %}
#### 3.依赖软件的相关参数
表1-4~表1-8列出了Nginx依赖的常用软件支持的参数。
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-007.png 表1-4 PCRE的设置参数 %}
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-008.png 表1-5 OpenSSL的设置参数 %}
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-009.png 表1-6 原子库的设置参数 %}
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-010.png 表1-7 散列函数库的设置参数 %}
{% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-011.png 表1-8 zlib库的设置参数 %}
#### 4.模块相关的参数
除了少量核心代码外，Nginx完全是由各种功能模块组成的。这些模块会根据配置参数决定自己的行为，因此，正确地使用各个模块非常关键。在configure的参数中，我们把它们分为五大类。
- 事件模块。 
- 默认即编译进入Nginx的HTTP模块。 
- 默认不会编译进入Nginx的HTTP模块。 
- 邮件代理服务器相关的mail模块。 
- 其他模块。

1. 事件模块
   表1-9中列出了Nginx可以选择哪些事件模块编译到产品中。
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-012.png 表1-9 configure支持的事件模块参数 %}
2. 默认即编译进入Nginx的HTTP模块
   表1-10列出了默认就会编译进Nginx的核心HTTP模块，以及如何把这些HTTP模块从产品中去除。
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-013.png 表1-10 configure中默认编译到Nginx中的HTTP模块参数 %}
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-014.png %}
3. 默认不会编译进入Nginx的HTTP模块
   表1-11列出了默认不会编译至Nginx中的HTTP模块以及把它们加入产品中的方法。
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-015.png 表1-11 configure中默认不会编译到Nginx中的HTTP模块参数 %}
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-016.png %}
4. 邮件代理服务器相关的mail模块
   表1-12列出了把邮件模块编译到产品中的参数。
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-017.png 表1-12 configure提供的邮件模块参数 %}
5. 其他参数
   configure还接收一些其他参数，表1-13中列出了相关参数的说明。
   {% image https://hexo-duration.oss-cn-hangzhou.aliyuncs.com/understanding-nginx/understanding-nginx/nginx-chapter01-018.png 表1-13 configure提供的其他参数 %}
### 1.5.2 configure执行流程
我们看到configure命令支持非常多的参数，读者可能会好奇它在执行时到底做了哪些事情，本节将通过解析configure源码来对它有一个感性的认识。configure由Shell脚本编写，中间会调用<nginx-source>/auto/目录下的脚本。这里将只对configure脚本本身做分析，对于它所调用的auto目录下的其他工具脚本则只做功能性的说明。<br>
configure脚本的内容如下：
{% codeblock lang:shell %}
#!/bin/sh
# Copyright (C) Igor Sysoev
# Copyright (C) Nginx, Inc.
#auto/options脚本处理configure命令的参数。例如，如果参数是--help，那么显示支持的所有参数格式。options脚本会定义后续工作将要用到的变量，然后根据本次参数以及默认值设置这些变量
. auto/options

#auto/init脚本初始化后续将产生的文件路径。例如，Makefile、ngx_modules.c等文件默认情况下将会在<nginx-source>/objs/
. auto/init

#auto/sources脚本将分析Nginx的源码结构，这样才能构造后续的Makefile文件
. auto/sources

#编译过程中所有目标文件生成的路径由—builddir=DIR参数指定，默认情况下为<nginx-source>/objs，此时这个目录将会被创建
test -d $NGX_OBJS || mkdir $NGX_OBJS

#开始准备建立ngx_auto_headers.h、autoconf.err等必要的编译文件
echo > $NGX_AUTO_HEADERS_H
echo > $NGX_AUTOCONF_ERR

#向objs/ngx_auto_config.h写入命令行带的参数
echo "#define NGX_CONFIGURE \"$NGX_CONFIGURE\"" > $NGX_AUTO_CONFIG_H

#判断DEBUG标志，如果有，那么在objs/ngx_auto_config.h文件中写入DEBUG宏
if [ $NGX_DEBUG = YES ]; then
 have=NGX_DEBUG . auto/have
fi

#现在开始检查操作系统参数是否支持后续编译
if test -z "$NGX_PLATFORM"; then
 echo "checking for OS"
 NGX_SYSTEM=`uname -s 2>/dev/null`
 NGX_RELEASE=`uname -r 2>/dev/null`
 NGX_MACHINE=`uname -m 2>/dev/null`

#屏幕上输出OS名称、内核版本、32位/64位内核
 echo " + $NGX_SYSTEM $NGX_RELEASE $NGX_MACHINE"
 NGX_PLATFORM="$NGX_SYSTEM:$NGX_RELEASE:$NGX_MACHINE";
 case "$NGX_SYSTEM" in
 MINGW32_*)
 NGX_PLATFORM=win32
 ;;
 esac
else
 echo "building for $NGX_PLATFORM"
 NGX_SYSTEM=$NGX_PLATFORM
fi

#检查并设置编译器，如GCC是否安装、GCC版本是否支持后续编译nginx
. auto/cc/conf

#对非Windows操作系统定义一些必要的头文件，并检查其是否存在，以此决定configure后续步骤是否可以成功[1]
if [ "$NGX_PLATFORM" != win32 ]; then
 . auto/headers
fi

#对于当前操作系统，定义一些特定的操作系统相关的方法并检查当前环境是否支持。例如，对于Linux，在这里使用sched_setaffinity设置进程优先级，使用Linux特有的sendfile系统调用来加速向网络中发送文件块
. auto/os/conf

#定义类UNIX 操作系统中通用的头文件和系统调用等，并检查当前环境是否支持
if [ "$NGX_PLATFORM" != win32 ]; then
 . auto/unix
fi

#最核心的构造运行期modules的脚本。它将会生成ngx_modules.c文件，这个文件会被编译进Nginx中，其中它所做的唯一的事情就是定义了ngx_modules数组。ngx_modules指明Nginx运行期间有哪些模块会参与到请求的处理中，包括HTTP请求可能会使用哪些模块处理，因此，它对数组元素的顺序非常敏感，也就是说，绝大部分模块在ngx_modules数组中的顺序其实是固定的。例如，一个请求必须先执行ngx_http_gzip_filter_module模块重新修改HTTP响应中的头部后，才能使用ngx_http_header_filter模块按照headers_in结构体里的成员构造出以TCP流形式发送给客户端的HTTP响应头部。注意，我们在--add-module=参数里加入的第三方模块也在此步骤写入到ngx_modules.c文件中了
. auto/modules

#conf脚本用来检查Nginx在链接期间需要链接的第三方静态库、动态库或者目标文件是否存在
. auto/lib/conf

#处理Nginx安装后的路径
case ".$NGX_PREFIX" in
   .)
   NGX_PREFIX=${NGX_PREFIX:-/usr/local/nginx}
   have=NGX_PREFIX value="\"$NGX_PREFIX/\"" . auto/define
   ;;
   .!)
   NGX_PREFIX=
   ;;
   *)
   have=NGX_PREFIX value="\"$NGX_PREFIX/\"" . auto/define
   ;;
esac

#处理Nginx安装后conf文件的路径
if [ ".$NGX_CONF_PREFIX" != "." ]; then
 have=NGX_CONF_PREFIX value="\"$NGX_CONF_PREFIX/\"" . auto/define
fi

#处理Nginx安装后，二进制文件、pid、lock等其他文件的路径可参见configure参数中路径类选项的说明
have=NGX_SBIN_PATH value="\"$NGX_SBIN_PATH\"" . auto/define
have=NGX_CONF_PATH value="\"$NGX_CONF_PATH\"" . auto/define
have=NGX_PID_PATH value="\"$NGX_PID_PATH\"" . auto/define
have=NGX_LOCK_PATH value="\"$NGX_LOCK_PATH\"" . auto/define
have=NGX_ERROR_LOG_PATH value="\"$NGX_ERROR_LOG_PATH\"" . auto/define
have=NGX_HTTP_LOG_PATH value="\"$NGX_HTTP_LOG_PATH\"" . auto/define
have=NGX_HTTP_CLIENT_TEMP_PATH value="\"$NGX_HTTP_CLIENT_TEMP_PATH\"" . auto/define
have=NGX_HTTP_PROXY_TEMP_PATH value="\"$NGX_HTTP_PROXY_TEMP_PATH\"" . auto/define
have=NGX_HTTP_FASTCGI_TEMP_PATH value="\"$NGX_HTTP_FASTCGI_TEMP_PATH\"" . auto/define
have=NGX_HTTP_UWSGI_TEMP_PATH value="\"$NGX_HTTP_UWSGI_TEMP_PATH\"" . auto/define
have=NGX_HTTP_SCGI_TEMP_PATH value="\"$NGX_HTTP_SCGI_TEMP_PATH\"" . auto/define

#创建编译时使用的objs/Makefile文件
. auto/make

#为objs/Makefile加入需要连接的第三方静态库、动态库或者目标文件
. auto/lib/make

#为objs/Makefile加入install功能，当执行make install时将编译生成的必要文件复制到安装路径，建立必要的目录
. auto/install

# 在ngx_auto_config.h文件中加入NGX_SUPPRESS_WARN宏、NGX_SMP宏
. auto/stubs

#在ngx_auto_config.h文件中指定NGX_USER和NGX_GROUP宏，如果执行configure时没有参数指定，默认两者皆为nobody（也就是默认以nobody用户运行进程）
have=NGX_USER value="\"$NGX_USER\"" . auto/define
have=NGX_GROUP value="\"$NGX_GROUP\"" . auto/define

#显示configure执行的结果，如果失败，则给出原因
. auto/summary
{% endcodeblock %}
（注：在configure脚本里检查某个特性是否存在时，会生成一个最简单的只包含main函数的C程序，该程序会包含相应的头文件。然后，通过检查是否可以编译通过来确认特性是否支持，并将结果记录在objs/autoconf.err文件中。后续检查头文件、检查特性的脚本都用了类似的方法。）
### 1.5.3 configure生成的文件
当configure执行成功时会生成objs目录，并在该目录下产生以下目录和文件：
```shell
|---ngx_auto_headers.h
|---autoconf.err
|---ngx_auto_config.h
|---ngx_modules.c
|---src
| |---core
| |---event
| | |---modules
| |---os
| | |---unix
| | |---win32
| |---http
| | |---modules
| | | |---perl
| |---mail
| |---misc
|---Makefile
```
上述目录和文件介绍如下。
1. src目录用于存放编译时产生的目标文件。
2. Makefile文件用于编译Nginx工程以及在加入install参数后安装Nginx。
3. autoconf.err保存configure执行过程中产生的结果。
4. ngx_auto_headers.h和ngx_auto_config.h保存了一些宏，这两个头文件会被src/core/ngx_config.h及src/os/unix/ngx_linux_config.h文件（可将“linux”替换为其他UNIX操作系统）引用。
5. ngx_modules.c是一个关键文件，我们需要看看它的内部结构。一个默认配置下生成的ngx_modules.c文件内容如下：
   ```c
   #include <ngx_config.h>
   #include <ngx_core.h>…
   ngx_module_t *ngx_modules[] = {
   &ngx_core_module,
   &ngx_errlog_module,
   &ngx_conf_module,
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
   &ngx_http_auth_basic_module,
   &ngx_http_access_module,
   &ngx_http_limit_zone_module,
   &ngx_http_limit_req_module,
   &ngx_http_geo_module,
   &ngx_http_map_module,
   &ngx_http_split_clients_module,
   &ngx_http_referer_module,
   &ngx_http_rewrite_module,
   &ngx_http_proxy_module,
   &ngx_http_fastcgi_module,
   &ngx_http_uwsgi_module,
   &ngx_http_scgi_module,
   &ngx_http_memcached_module,
   &ngx_http_empty_gif_module,
   &ngx_http_browser_module,
   &ngx_http_upstream_ip_hash_module,
   &ngx_http_write_filter_module,
   &ngx_http_header_filter_module,
   &ngx_http_chunked_filter_module,
   &ngx_http_range_header_filter_module,
   &ngx_http_gzip_filter_module,
   &ngx_http_postpone_filter_module,
   &ngx_http_ssi_filter_module,
   &ngx_http_charset_filter_module,
   &ngx_http_userid_filter_module,
   &ngx_http_headers_filter_module,
   &ngx_http_copy_filter_module,
   &ngx_http_range_body_filter_module,
   &ngx_http_not_modified_filter_module,
   NULL
   };
   ```
   ngx_modules.c文件就是用来定义ngx_modules数组的。<br>
   ngx_modules是非常关键的数组，它指明了每个模块在Nginx中的优先级，当一个请求同时符合多个模块的处理规则时，将按照它们在ngx_modules数组中的顺序选择最靠前的模块优先处理。对于HTTP过滤模块而言则是相反的，因为HTTP框架在初始化时，会在ngx_modules数组中将过滤模块按先后顺序向过滤链表中添加，但每次都是添加到链表的表头，因此，对HTTP过滤模块而言，在ngx_modules数组中越是靠后的模块反而会首先处理HTTP响应（参见第6章及第11章的11.9节）。<br>
   因此，ngx_modules中模块的先后顺序非常重要，不正确的顺序会导致Nginx无法工作，这是auto/modules脚本执行后的结果。读者可以体会一下上面的ngx_modules中同一种类型下（第8章会介绍模块类型，第10章、第11章将介绍的HTTP框架对HTTP模块的顺序是最敏感的）各个模块的顺序以及这种顺序带来的意义。<br>
   可以看出，在安装过程中，configure做了大量的幕后工作，我们需要关注在这个过程中Nginx做了哪些事情。configure除了寻找依赖的软件外，还针对不同的UNIX操作系统做了许多优化工作。这是Nginx跨平台的一种具体实现，也体现了Nginx追求高性能的一贯风格。configure除了生成Makefile外，还生成了ngx_modules.c文件，它决定了运行时所有模块的优先级（在编译过程中而不是编码过程中）。对于不需要的模块，既不会加入ngx_modules数组，也不会编译进Nginx产品中，这也体现了轻量级的概念。<br>
   [1] 在configure脚本里检查某个特性是否存在时，会生成一个最简单的只 包含main函数的C程序，该程序会包含相应的头文件。然后，通过检查是否可以编译通过来确认特性是否支持，并将结果记录在objs/autoconf.err文件中。后续检查头文件、检查特性的脚本都用了类似的方法。
## 1.6 Nginx的命令行控制
在Linux中，需要使用命令行来控制Nginx服务器的启动与停止、重载配置文件、回滚日志文件、平滑升级等行为。默认情况下，Nginx被安装在目录/usr/local/nginx/中，其二进制文件路径为/usr/local/nginc/sbin/nginx，配置文件路径为/usr/local/nginx/conf/nginx.conf。当然，在configure执行时是可以指定把它们安装在不同目录的。为了简单起见，本节只说明默认安装情况下的命令行的使用情况，如果读者安装的目录发生了变化，那么替换一下即可。
1. 默认方式启动
   直接执行Nginx二进制程序。例如：
   ```shell
   /usr/local/nginx/sbin/nginx
   ```
   这时，会读取默认路径下的配置文件：/usr/local/nginx/conf/nginx.conf。<br>
   实际上，在没有显式指定nginx.conf配置文件路径时，将打开在configure命令执行时使用--conf-path=PATH指定的nginx.conf文件（参见1.5.1节）。
2. 另行指定配置文件的启动方式
   使用-c参数指定配置文件。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -c /tmp/nginx.conf
   ```
   这时，会读取-c参数后指定的nginx.conf配置文件来启动Nginx。
3. 另行指定安装目录的启动方式
   使用-p参数指定Nginx的安装目录。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -p /usr/local/nginx/
   ```
4. 另行指定全局配置项的启动方式
   可以通过-g参数临时指定一些全局配置项，以使新的配置项生效。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -g "pid /var/nginx/test.pid;"
   ```
   上面这行命令意味着会把pid文件写到/var/nginx/test.pid中。-g参数的约束条件是指定的配置项不能与默认路径下的nginx.conf中的配置项相冲突，否则无法启动。就像上例那样，类似这样的配置项：pid logs/nginx.pid，是不能存在于默认的nginx.conf中的。<br>
   另一个约束条件是，以-g方式启动的Nginx服务执行其他命令行时，需要把-g参数也带上，否则可能出现配置项不匹配的情形。例如，如果要停止Nginx服务，那么需要执行下面代码：
   ```shell
   /usr/local/nginx/sbin/nginx -g "pid /var/nginx/test.pid;" -s stop
   ```
   如果不带上-g"pid/var/nginx/test.pid;"，那么找不到pid文件，也会出现无法停止服务的情况。
5. 测试配置信息是否有错误
   在不启动Nginx的情况下，使用-t参数仅测试配置文件是否有错误。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -t
   ```
   执行结果中显示配置是否正确。
6. 在测试配置阶段不输出信息
   测试配置选项时，使用-q参数可以不把error级别以下的信息输出到屏幕。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -t -q
   ```
7. 显示版本信息
   使用-v参数显示Nginx的版本信息。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -v
   ```
8. 显示编译阶段的参数
   使用-V参数除了可以显示Nginx的版本信息外，还可以显示配置编译阶段的信息，如GCC编译器的版本、操作系统的版本、执行configure时的参数等。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -V
   ```
9. 快速地停止服务
   使用-s stop可以强制停止Nginx服务。-s参数其实是告诉Nginx程序向正在运行的Nginx服务发送信号量，Nginx程序通过nginx.pid文件中得到master进程的进程ID，再向运行中的master进程发送TERM信号来快速地关闭Nginx服务。例如：
   ```shell
   /usr/local/nginx/sbin/nginx -s stop
   ```
   实际上，如果通过kill命令直接向nginx master进程发送TERM或者INT信号，效果是一样的。例如，先通过ps命令来查看nginx master的进程ID：
   ```shell
   :ahf5wapi001:root > ps -ef | grep nginx
   root 10800 1 0 02:27 ? 00:00:00 nginx: master process ./nginx
   root 10801 10800 0 02:27 ? 00:00:00 nginx: worker process
   ```
   接下来直接通过kill命令来发送信号：
   ```shell
   kill -s SIGTERM 10800
   ```
   或者：
   ```shell
   kill -s SIGINT 10800
   ```
   上述两条命令的效果与执行/usr/local/nginx/sbin/nginx-s stop是完全一样的。
10. “优雅”地停止服务
    如果希望Nginx服务可以正常地处理完当前所有请求再停止服务，那么可以使用-s quit参数来停止服务。例如：
    ```shell
    /usr/local/nginx/sbin/nginx -s quit
    ```
    该命令与快速停止Nginx服务是有区别的。当快速停止服务时，worker进程与master进程在收到信号后会立刻跳出循环，退出进程。而“优雅”地停止服务时，首先会关闭监听端口，停止接收新的连接，然后把当前正在处理的连接全部处理完，最后再退出进程。 <br>
    与快速停止服务相似，可以直接发送QUIT信号给master进程来停止服务，其效果与执行-s quit命令是一样的。例如：
    ```shell
    kill -s SIGQUIT <nginx master pid>
    ```
    如果希望“优雅”地停止某个worker进程，那么可以通过向该进程发送WINCH信号来停止服务。例如：
    ```shell
    kill -s SIGWINCH <nginx worker pid>
    ```
11. 使运行中的Nginx重读配置项并生效
    使用-s reload参数可以使运行中的Nginx服务重新加载nginx.conf文件。例如：
    ```shell
    /usr/local/nginx/sbin/nginx -s reload
    ```
    事实上，Nginx会先检查新的配置项是否有误，如果全部正确就以“优雅”的方式关闭，再重新启动Nginx来实现这个目的。类似的，-s是发送信号，仍然可以用kill命令发送HUP信号来达到相同的效果。
    ```shell
    kill -s SIGHUP <nginx master pid>
    ```
12. 日志文件回滚
    使用-s reopen参数可以重新打开日志文件，这样可以先把当前日志文件改名或转移到其他目录中进行备份，再重新打开时就会生成新的日志文件。这个功能使得日志文件不至于过大。例如：
    ```shell
    /usr/local/nginx/sbin/nginx -s reopen
    ```
    当然，这与使用kill命令发送USR1信号效果相同。
    ```shell
    kill -s SIGUSR1 <nginx master pid>
    ```
13. 平滑升级Nginx
    当Nginx服务升级到新的版本时，必须要将旧的二进制文件Nginx替换掉，通常情况下这是需要重启服务的，但Nginx支持不重启服务来完成新版本的平滑升级。<br>
    升级时包括以下步骤：
    1. 通知正在运行的旧版本Nginx准备升级。通过向master进程发送USR2信号可达到目的。例如：
       ```shell
       kill -s SIGUSR2 <nginx master pid>
       ```
       这时，运行中的Nginx会将pid文件重命名，如将/usr/local/nginx/logs/nginx.pid重命名为/usr/local/nginx/logs/nginx.pid.oldbin，这样新的Nginx才有可能启动成功。
    2. 启动新版本的Nginx，可以使用以上介绍过的任意一种启动方法。这时通过ps命令可以发现新旧版本的Nginx在同时运行。
    3. 通过kill命令向旧版本的master进程发送SIGQUIT信号，以“优雅”的方式关闭旧版本的Nginx。随后将只有新版本的Nginx服务运行，此时平滑升级完毕。
14. 显示命令行帮助
    使用-h或者-?参数会显示支持的所有命令行参数。
## 1.7 小结
本章介绍了Nginx的特点以及在什么场景下需要使用Nginx，同时介绍了如何获取Nginx以及如何配置、编译、安装运行Nginx。本章还深入介绍了最为复杂的configure过程，这部分内容是学习本书第二部分和第三部分的基础。