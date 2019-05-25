# centos7-ocserv-cisco-vpn
记录一次在centos7上安装open connect server实现外网用户访问内网资源的过程，以及自己遇到的一些坑。

## 背景
* 公司希望员工可以在外网也能通过访问公司出口IP的方式登录到VPN，从而访问内网资源，并且访问外网不受影响。 

## 原理简介（Cisco Anyconenct VPN vs Huawei Firewall SSL VPN）


## 为什么不用华为防火墙自带的SSL VPN
* 公司网络采用华为的设备，如果采用华为防火墙自带的SSL VPN，必须用他们家自己的客户端Secoclient，安装会有windows10版本兼容问题而且必须在BIOS中关闭secure boot才可以用。公司希望能统一采用cisco anyconnect的客户端，这样电脑，手机都可以用，而且客户端安装也容易。
* 根据上面的原理也能看出来，华为防火墙自带的SSL VPN功能会分配给每个客户端一个公司内部空闲的内网IP。这种方式占用公司内网IP资源不说，更主要是因为公司有些权限是按照vlan来划分的，而分配给客户端的IP必须受限于防火墙到核心交换机的路由，所以分配的IP不一定能满足vlan要求。cisco的VPN和客户端之间采用一对一的连接，然后在VPN服务器的网卡做NAT。理论上分配给客户端的IP可以是任意内网网段的IP，然后VPN服务器上会为每一个客户端创建一个专属的虚拟网卡进行一对一通信，访问内网资源都从VPN服务器的真实物理网卡出去，所以只需要VPN服务器的IP满足vlan条件即可。


## 操作步骤
* centos7机器初始化

  1. 在vmware ESXI主机里面起一台linux虚拟机，iso镜像采用官网下载的centos7 minimum版本。选择虚拟机port group的时候注意选择符合要求vlan的group，如果没有就要考虑新起一个port group并且指定vlan，不过前提条件是ESXI机器的真实物理网卡上行连接的trunk口要包括该vlan。
  2. 参考https://github.com/Victor2Code/centos7 对centos7进行简单的初始化，主要包括
