# centos7-ocserv-cisco-vpn
记录一次在centos7上安装open connect server实现外网用户访问内网资源的过程，以及自己遇到的一些坑。

## 背景
* 公司希望员工可以在外网也能通过访问公司出口IP的方式登录到VPN，从而访问内网资源，并且访问外网不受影响。 

## 原理简介
* 

## 一些限制条件 
* 公司网络采用华为的设备，如果采用华为防火墙自带的SSL VPN，必须用他们家自己的客户端Secoclient，安装有windows10版本兼容问题而且必须在BIOS中关闭secure boot才可以用。公司希望能统一采用cisco anyconnect的客户端，这样电脑，手机都可以用，而且客户端安装也容易。
* 公司采用vlan去做一些权限的隔离，所以VPN的出口IP
