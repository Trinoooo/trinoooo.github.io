---
layout: wiki  # 使用wiki布局模板
wiki: Kubernetes in Action # 这是项目名
title: 关于本书
order: 4
---
本书旨在让你能够熟练使用Kubernetes。它介绍了在Kubernetes中有效地开发和运行应用所需的几乎所有概念。<br>
在深入研究Kubernetes之前，本书概述了Docker等容器技术，包括如何构建容器，以便即使以前没有使用过这些技术的读者也可以使用它们。然后，它会慢慢带你从基本概念到实现原理了解大部分的Kubernetes知识。
## 本书适合谁
本书主要关注应用开发人员，但也从操作的角度概述了应用的管理。它适合任何对在多服务器上运行和管理容器化应用感兴趣的人。<br>
对于希望学习容器技术以及大规模的容器编排的人，无论是初学者还是高级软件工程师都将得到在Kubernetes环境中开发、容器化和运行应用所需的专业知识。<br>
阅读本书不需要预先了解容器或Kubernetes技术。本书以渐进的方式展开主题，不会使用让非专家开发者难以理解的应用源代码。<br>
读者至少应该具备编程、计算机网络和运行Linux基本命令的基础知识，并了解常用的计算机协议，如HTTP协议。<br>
本书分为三个部分，涵盖18个章节。<br>
第一部分简要地介绍Docker和Kubernetes、如何设置Kubernetes集群，以及如何在集群中运行一个简单的应用。它包括两章：
- 第1章解释了什么是Kubernetes、Kubernetes的起源，以及它如何帮助解决当今大规模应用管理的问题。
- 第2章是关于如何构建容器镜像并在Kubernetes集群中运行的实践教程。还解释了如何运行本地单节点Kubernetes集群，以及在云上运行适当的多节点集群。

第二部分介绍了在Kubernetes中运行应用必须理解的关键概念。本章内容如下：
- 第3章介绍了Kubernetes的基本构建模块——pod，并解释了如何通过标签组织pod和其他Kubernetes对象。
- 第4章将向你介绍Kubernetes如何通过自动重启容器来保持应用程序的健康。还展示了如何正确地运行托管的pod，水平伸缩它们，使它们能够抵抗集群节点的故障，并在未来或定期运行它们。
- 第5章介绍了pod如何向运行在集群内外的客户端暴露它们提供的服务，还展示了运行在集群中的pod是如何发现和访问集群内外的服务的。
- 第6章解释了在同一个pod中运行的多个容器如何共享文件，以及如何管理持久化存储并使得pod可以访问。
- 第7章介绍了如何将配置数据和敏感信息（如凭据）传递给运行在pod中的应用。
- 第8章描述了应用如何获得正在运行的Kubernetes环境的信息，以及如何通过与Kubernetes通信来更改集群的状态。
- 第9章介绍了Deployment的概念，并解释了在Kubernetes环境中运行和更新应用的正确方法。

第三部分深入研究了Kubernetes集群的内部，介绍了一些额外的概念，并从更高的角度回顾了在前两部分中所学到的所有内容。这是最后一组章节：
- 第11章深入Kubernetes的底层，解释了组成Kubernetes集群的所有组件，以及每个组件的作用。它还解释了pod如何通过网络进行通信，以及服务如何跨多个pod形成负载平衡。
- 第12章解释了如何保护Kubernetes API服务器，以及通过扩展集群使用身份验证和授权。
- 第13章介绍了pod如何访问节点的资源，以及集群管理员如何防止pod访问节点的资源。
- 第14章深入了解限制每个应用程序允许使用的计算资源，配置应用的QoS（Quality of Service）保证，以及监控各个应用的资源使用情况。 还会介绍如何防止用户消耗太多资源。
- 第15章讨论了如何通过配置Kubernetes来自动伸缩应用运行的副本数，以及在当前集群节点数量不能接受任何新增应用时，如何对集群进行扩容。
- 第16章介绍了如何确保pod只被调度到特定的节点，或者如何防止它们被调度到其他节点。还介绍了如何确保pod被调度在一起，或者如何防止它们调度在一起。
- 第17章介绍了如何开发应用程序并部署在集群中。还介绍了如何配置开发和测试工作流来提高开发效率。
- 第18章介绍了如何使用自己的自定义对象扩展Kubernetes，以及其他人是如何开发并创建企业级应用平台的。

随着章节的深入，不仅可以了解单个构建Kubernetes的模块，还可以逐步增加对使用kubectl命令行工具的理解。
## 关于代码
虽然这本书没有包含很多实际的源代码，但是包含了许多YAML格式的Kubernetes资源清单，以及shell命令及其输出。所有这些都是用等宽字体显示的，以便与普通文本相互区分。<br>
shell命令大部分以粗体文字展示，以便与命令的输出清晰地区分，但为了表示强调有时只有命令的最重要的部分或是命令输出的一部分是粗体的。在大多数情况下，为了适应图书有限的版面空间，命令输出会被重新格式化。此外，由于Kubernetes的命令行工具kubectl不断发展更新，新版本输出的信息可能会比书中显示的更多。如果输出结果不完全一致，请不要感到困惑。<br>
代码清单有时会包括续行标记（➥）表示一行文字延续到下一行。代码清单还可能包括注释，这些注释用于解释最重要的部分。<br>
在文本段落包含一些常见的元素，如Pod、ReplicationController、ReplicaSet、DaemonSet等，为了避免代码字体的过度复杂，便于阅读，都以常规字体显示。在某些地方，大写字母开头的“Pod”表示Pod资源，小写则表示实际运行的容器组。<br>
本书中的所有示例都使用谷歌Kubernetes引擎（Google Kubernetes Engine），在Kubernetes 1.8版本和Minikube的本地集群进行了运行测试。完整的源代码和YAML清单可以在https://github.com/luksa/kubernetes-in-action找到，或者从出版商的网站www.manning.com/books/kubernetes-in-action中下载。<br>
其他线上资源<br>
可以在下面的网址中找到许多额外的Kubernetes信息：
- Kubernetes官方网站：https://kubernetes.io 
- 经常发布有用信息的Kubernetes博客：http://blog.kubernetes.io 
- Kubernetes社区的Slack频道：http://slack.k8s.io 
- Kubernetes以及Cloud Native Computing Foundation的YouTube频道： 
  - https://www.youtube.com/channel/UCZ2bu0qutTOM0tHYa_jkIwg
  - https://www.youtube.com/channel/UCvqbFHwN-nwalWPjPUKpvTA
- 为了获得对于单个话题更深入的理解或者贡献Kubernetes项目，可以查看Kubernetes特别兴趣小组（SIGs）:https://github.com/kubernetes/kubernetes/wiki/Special-InterestGroups-(SIGs)。

最后，由于Kubernetes是一个开源项目，Kubernetes源码本身包含大量的信息。可以访问Kubernetes和相关的仓库：https://github.com/kubernetes/。
## 读者服务
轻松注册成为博文视点社区用户( www.broadview.com.cn )，扫码直达本书页面。<br>
提交勘误：您对书中内容的修改意见可在 提交勘误 处提交，若被采纳，将获赠博文视点社区积分（在您购买电子书时，积分可用来抵扣相应金额）。<br>
交流互动：在页面下方读者评论 处留下您的疑问或观点，与我们和其他读者一同学习交流。<br>
页面入口：http://www.broadview.com.cn/34995