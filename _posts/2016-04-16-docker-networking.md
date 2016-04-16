Docker Networking知识点总结一
- 
    随着Docker容器技术越来越热，越来越多的人对它着迷，将它应用到生产环境中。本篇文章就主要介绍一些Docker容器网络方面的知识点。
先从本人遇到的一个Docker网络问题开始吧。scenario是这样的，ubuntu14.04的宿主机可以出外网，可以正常使用`apt-get install`在线安装软件。但是我在容器里使用`apt-get install`却死活无法download下来我要装的软件。后来这个问题在larry老师和tobe的帮助下得以解决。解决方法有两种，  
第一种，在起容器的时候指定网络类型为host，即`docker run -i -t --net=host ubuntu:14.04 /bin/bash`，这种方法容器和宿主机共用一个网络，不会产生容器本身的网络。  
第二种，清iptable,`iptables -F`。然后在起容器的时候指定DNS，即`docker run -i -t --dns=$DNSIP ubuntu:14.04 /bin/bash`。默认Docker给容器分配的DNS是8.8.8.8。  
**ok**，下面来进入正题，介绍一些docker网络的知识。  
**1**. docker的4种网络模式  
在docker起容器的时候可以用`--net`选项指定容器的网络模式，docker有以下4种网络模式：  
#### *host模式，使用--net=host指定。* ####
#### *container模式，使用--net=container:NAMEorID指定。* ####
#### *none模式，使用--net=none指定。* ####
#### *bridge模式，使用--net=bridge指定，默认设置。* 


1. 在host模式下，容器将不会获得一个独立的Network Namespace，而是和宿主机共用一个Network Namespace。容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口。  


1. 在container模式下，容器是和已经存在的一个容器共享一个Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享IP、端口范围等。两个容器的进程可以通过lo网卡设备通信。  


1. 在none模式下，Docker容器拥有自己的Network Namespace，但是，并不为Docker容器进行任何网络配置。也就是说，这个Docker容器没有网卡、IP、路由等信息。需要我们自己为Docker容器添加网卡、配置IP等。  


1. 在bridge模式下，这也是docker默认的网络模式。它会为每一个容器分配Network Namespace、设置IP等，并将一个主机上的Docker容器连接到一个虚拟网桥上。  
  
我们着重来看一下这个模式：  
当docker server启动时，会在主机上创建一个名为docker0的虚拟网桥。一般Docker会使用172.17.0.0/16这个网段，并将172.17.42.1/16分配给docker0网桥。你可以在宿主机上通过`ifconfig`命令查看。单机环境下的网络拓扑如下，  
![](http://i.imgur.com/Cj9C6vq.jpg)  

Docker0在宿主机桥接所有的容器网卡，你可以通过`brctl show`命令来查看网卡的桥接情况，如下图  
![](http://i.imgur.com/BP4Yvpw.jpg)
  
在容器里看到的地址一般是像下面这样的地址  
![](http://i.imgur.com/Lb2di7i.jpg)  
  
可以把这个网络看成是一个私有网络，通过`nat`的方式连接外网。它的原理是这样的，容器内的`eth0`将数据包发送到对应的`veth*`，也就是docker0上的某个接口，然后docker0再将这个数据包转发给宿主机的eth0，通过`sysctl -a | grep ip_forward`可以看到转发情况。如果容器要访问外网，必须将这个值设为true，也就是1。容器内也有这个参数的设置，是限制容器间通信的。这是容器间互访和容器访问外网的网络知识。  
  
#### 那么问题来了 ####，如果我的容器分布在不同的物理主机上，它们要怎么实现通信呢？haha~~,这也是有办法的。  
只要将docker0这个网桥桥接到我们指定的网卡上就行了。什么意思？如下图  
![](http://i.imgur.com/hvZmfjH.jpg)  
主机A和主机B的网卡一都连着物理交换机的同一个vlan 101,这样网桥一和网桥三就相当于在同一个物理网络中了，而容器一、容器三、容器四也在同一物理网络中了，他们之间可以相互通信。
主机A上的网卡二连接了vlan102 桥接网桥二，它不与其他物理主机和容器相通。当然你可以加入路由信息让它们通信。  
  
以上介绍的都是容器互访或者容器访问外网，最后如果外网要访问容器怎么办呢？很简单，运行容器的时候加上-p参数即可，就是做宿主机和容器间的端口映射。  
  
这就是part1的docker网络知识，感谢larry老师和tobe的友情帮助！



