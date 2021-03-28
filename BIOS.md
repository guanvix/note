### BIOS

**BIOS**,基本输入输出系统，加载在计算机系统上最基本的软件代码，保存在主板上的一块EPROM或EEPROM芯片中，里面装有系统的重要信息和设置系统参数的设置程序-BIOS Setup程序。

**功能**
  - 上电
  - 自检及初始化
  - 程序服务处理
  - 硬件中断处理

#### BIOS启动过程
  - Legacy BIOS启动过程
    > <small>初始化--->开机自检--->记录计算机系统的配置--->提供中断服务--->启动操作系</small>  
    > <small>初始化：CPU初始化，执行BIOS程序。</small>  
    > <small>开机自检：POST,对各种硬件如CPU、RAM、键盘、鼠标等进行检测</small>
    > <small></small>  
  - UEFI BIOS启动过程
    > <small>SEC--->PEI--->DXE--->BDS--->TSL--->RT--->AL</small>  
    > <small>SEC：安全性，通电，内存初始化，CPU只能使用Cache来验证CPU、芯片组和主机驱动。</small>  
    > <small>PEI：EFI前初始化，初始化一小部分低地址内存空间，CPU开始使用此内存初始化CPU、芯片组和主板，随后EFI驱动载入内存。</small>  
    > <small>DXE：驱动执行环境，此阶段CPU、内存、PCI、USB、SATAheShell都会被初始化。</small>  
    > <small>BDS：开机设备选择，此阶段用户可以从检测到的启动设备中选择启动设备。</small>  
    > <small>TSL：临时系统载入，此阶段将由启动设备上的系统接手正式进入操作系统，若BDS阶段选择UEFI Shell则会进入简单命令行阶段。  
  
