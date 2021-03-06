# centos7-ocserv-cisco-vpn
记录一次在centos7上安装open connect server实现外网用户访问内网资源的过程，以及自己遇到的一些坑。

## 背景
* 公司希望员工可以在外网也能通过访问公司出口IP的方式登录到VPN，从而访问内网资源，并且访问外网不受影响。公司网络设备为华为。

## 原理简介（Cisco Anyconenct VPN vs Huawei Firewall SSL VPN）
假设IP为1.2.3.4的外网用户想要访问VPN，在公司的统一出口IP 1.1.1.1上打开端口6666做为VPN的监听端口。注：以下图片为原创，比较简单粗陋:laughing:，所有IP和端口信息为虚构，如有雷同，算你牛逼。
### Huawei SSL VPN
![Huawei](images/Huawei_VPN.png)
如上图所示，只需要在防火墙上开启SSL VPN特性即可。VPN会预留一个内网的IP池（图中为192.168.1.0/24）用于分配给VPN的客户端，这个池子里面的IP在公司内网不能被占用，并且需要有路由规则将IP池里面的IP连接到所有的内网资源。用户连接到VPN以后会被分配一个池子里面的IP（图中为192.168.1.2），然后通过该IP访问内网资源。

**优点**：配置简单；不需要额外的设备<br>
**缺点**：客户端为华为特有，不具有普适性；浪费内网IP资源

### Cisco Anyconnect VPN
![Cisco](images/Cisco_VPN.png)
如上图所示，内网需要一台专门的VPN服务器在特定的IP端口监听VPN请求（图中为192.168.1.2:10443），同时需要在防火墙上做一个NAT规则，将外网端口映射到VPN监听端口（图中为1.1.1.1:6666到192.168.1.2:10443）。用户连接到VPN以后会被分配VPN服务器指定的IP池子（图中为10.10.1.0/24）里面的IP（图中为10.10.1.3），**这个IP池子可以是任意的内网IP段**。VPN服务器上会创建一个虚拟的网卡（图中为10.10.1.1）和该客户端进行一对一的通信，同时会把所有流量路由到真实的物理网卡（图中为192.168.1.2）出去，并且在iptables的NAT表中加上MASQUERADE规则。

**优点**：不消耗内网IP；客户端多平台适用<br>
**缺点**：需要一台额外的VPN服务器

## 为什么不用华为防火墙自带的SSL VPN
* 公司网络采用华为的设备，如果采用华为防火墙自带的SSL VPN，必须用他们家自己的客户端Secoclient，安装会有windows10版本兼容问题而且必须在BIOS中关闭secure boot才可以用。公司希望能统一采用cisco anyconnect的客户端，这样电脑，手机都可以用，而且客户端安装也容易。
* 根据上面的原理也能看出来，华为防火墙自带的SSL VPN功能会分配给每个客户端一个公司内部空闲的内网IP。这种方式占用公司内网IP资源不说，更主要是因为公司有些权限是按照vlan来划分的，而分配给客户端的IP必须受限于防火墙到核心交换机的路由，所以分配的IP不一定能满足vlan要求。cisco的VPN和客户端之间采用一对一的连接，然后在VPN服务器的网卡做NAT。理论上分配给客户端的IP可以是任意内网网段的IP，然后VPN服务器上会为每一个客户端创建一个专属的虚拟网卡进行一对一通信，访问内网资源都从VPN服务器的真实物理网卡出去，所以只需要VPN服务器的IP满足vlan条件即可。


## 操作步骤
* centos7机器初始化

  1. 在vmware ESXI主机里面起一台linux虚拟机，iso镜像采用官网下载的centos7 minimum版本。选择虚拟机port group的时候注意选择符合要求vlan的group，如果没有就要考虑新起一个port group并且指定vlan，不过前提条件是ESXI机器的真实物理网卡上行连接的trunk口要包括该vlan。
  2. 参考https://github.com/Victor2Code/centos7 对centos7进行简单的初始化，主要包括关闭firewalld.service并且启用iptables，关闭selinux，同时下载一些常用工具。
  
* ocser一键安装脚本

  1. 采用网上找到的一键安装脚本ocserv-install-script-for-centos7.sh（出处：http://www.baerk.com/148.html ）进行安装即可。这个脚本要用root用户执行，并且系统只能是centos7。
  2. 安装过程中需要输入的信息如下：

      * （如果网卡多于一个）输入希望用来监听的网卡名字，直接回车默认第一个网卡
      * 用来监听的端口，直接回车默认10443
      * 用于登录VPN的用户名和密码，直接回车用户名user，密码随机生成10位数字大小写字母组合
  3. 安装过程中完成了下面操作：
  
      * 安装软件依赖包
      * 下载ocserv并编译安装
      * 复制ocserv.service到/usr/lib/systemd/system创建系统服务
      `systemctl daemon-reload`
      * 创建一个初始用户（默认用户名为user）以及密码
      * 优化ocserv配置文件，配置自定义的监听端口，路由规则以及DNS等信息
      * iptables打开监听端口，允许从IP池的forward，同时配置从IP池到物理网卡的MASQUERADE
      ```bash
      iptables -I INPUT -p tcp --dport 10443 -j ACCEPT
      iptables -I INPUT -p udp --dport 10443 -j ACCEPT
      iptables -A FORWARD -s 10.10.1.0/24 -j ACCEPT
      iptables -t nat -A POSTROUTING -s 10.10.1.0/24 -o xxx -j MASQUERADE
      ```
      * 关闭selinux
      ```bash
      sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
      setenforce 0
      ```
      * 允许ip forwarding
      ```bash
      sysctl -w net.ipv4.ip_forward=1
      echo net.ipv4.ip_forward = 1 >> "/etc/sysctl.conf"
      ```
      * 启动ocserv服务并允许开机启动
      ```bash
      systemctl start ocserv.service
      systemctl enable ocserv.service
      ```
* 防火墙设置

  1. 端口映射从1.1.1.1:6666到192.168.1.2:10443的NAT规则打开
  2. 从外网到内网192.168.1.2的规则要放开
* 检验安装结果
  
  1. 内网`telnet 192.168.1.2 10443`查看服务是否有在监听，如果上述安装过程没有报错基本都可以telnet到。
  2. 外网`telnet 1.1.1.1 6666`查看外网到内网的端口映射是否生效，如果失败检查上面的“防火墙设置”。
  3. Anyconnect客户端访问1.1.1.1:6666并且输入创建的用户名和密码查看是否成功连接。
  4. 连接成功以后查看本地的路由规则是否和配置文件一致，访问对应网站是否成功。
* 后续维护
  1. 配置文件位置为`/usr/local/etc/ocserv/ocserv.conf`，修改以后需要`systemctl restart ocserv`去重启ocserv使配置生效。主要涉及的修改就是路由规则添加或者修改。
  2. 添加用户`ocpasswd -c /usr/local/etc/ocserv/ocpasswd username`。
  3. 删除用户，修改`/usr/local/etc/ocserv/ocpasswd`文件，删除用户所在的那一行即可。
  4. 日志查询`journalctl -u ocserv -f`
