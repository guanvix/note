#### BMC
  基板管理控制器，可以在机器未开机的状态下，对机器进行固件升级、查看机器设备等操作。

  - 主要功能 
    - 监控服务器健康
    - 系统信息查询
    - 记录日志
    - 本地或远程管理
    - 远程控制台
    - 固件升级
    - PCIE卡带外管理  

  - 常用BMC芯片型号
    - G2  
        AST2400
    - G3/G5  
        AST2500
    - G6  
        AST2600

#### HDM  
  H3C的BMC管理软件，兼容IPMI1.5/IMPI2.0规范，通过第三方工具(ipmitool)基于LPC通道或LAN通道实现对服务器的有效管理  
  > <small>LPC通道：运行KCS协议，ipmitool等工具必须运行在服务器本机的操作系统上</small>  
  > <small>LAN通道：运行UDP/IP协议，ipmitool等工具可以远程管理服务器</small>

#### IPMI  
  智能平台管理接口，是一项应用于服务器管理系统设计的标准  
<small>
  > - 在IPMI管理平台中，BMC是核心控制器，系统管理软件对各器件的管理都是通过与BMC通信来实现的  
  > - 与BMC进行已验证的IPMI通信是通过建立会话来实现的  
  > - KCS协议，键盘控制器方式，用于BMC与SMS(系统管理软件)的一种传输协议  
  > - IPMI消息传输机制都是基于请求/应答机制，即请求消息触发一个动作，比如获取数据、配置参数等，而应答消息则返回该请求的执行情况。几乎所有IPMI消息组成结构都是一样的，比如都包括源地址、目的地址、负载等字段，不同的只是里面的内容
  > - 用于和BMC通信的接口有：SMIC、KCS、BT、IPMB、ICMP、LAN、Serial/Moden

</small>

    
