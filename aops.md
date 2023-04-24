# 一、A-Ops服务介绍

A-Ops是用于提升主机整体安全性的服务，通过资产管理、CVE管理、异常检测、配置溯源等功能，识别并管理主机中的信息资产，监测主机中的软件漏洞、排查主机中遇到的系统故障，使得目标主机能够更加稳定和安全的运行。

下表是A-Ops服务涉及模块的说明：

| 模块 | 说明  |
| ---------- | ---------------------------------------------------- |
| aops-ceres | A-Ops服务的客户端，<br />提供采集主机数据与管理其它数据采集器（如gala-gopher）的功能。<br />响应管理中心下发的命令，处理管理中心的需求与操作。 |
| aops-zeus  | A-Ops管理中心，主要负责与其它模块的交互，默认端口：11111<br />提供基本主机管理功能，主机与主机组的添加、删除等功能依赖此模块实现。 |
| aops-diana | 为A-Ops提供异常诊断模块功能，默认端口：11112<br />通过分析目标主机中数据，鉴别主机遇到的故障，提供故障修复功能。 |
| aops-hermes | 为A-Ops提供可视化操作界面，向用户展示数据信息。 |
| aops-apollo | A-Ops的漏洞管理模块相关功能依赖此服务实现，默认端口：11116<br />识别客户机周期性获取openEuler社区发布的安全公告，并更新到漏洞库中。<br />通过与漏洞库比对，检测出系统和软件存在的漏洞。 |
| aops-vulcanus | A-Ops的基本工具库，除aops-ceres与aops-hermes模块外，其余模块须与此模块共同安装使用。 |
| aops-tools | 提供基础环境一键部署脚本, 安装后在/opt/aops/scripts目录下可见。<br /> |
| gala-ragdoll | A-Ops的配置溯源功能依赖此模块实现，通过git监测并记录配置文件的改动，默认端口：11114 |
| dnf-hotpatch-plugin | dnf插件，使得dnf工具可识别热补丁信息，提供热补丁扫描以及热补丁修复的功能 |

# 二、部署环境要求

建议采用3台openEuler 22.03 LTS SP1机器完成部署，具体用途以及部署方案如下：

+ 机器A用于部署mysql、elasticsearch、kafka、prometheus等，主要提供数据服务支持，同时部署诊断模式下运行的aops-diana，建议内存8G+。
+ 机器B用于部署A-Ops服务端，提供业务功能支持，建议内存6G+。
+ 机器C用于部署A-Ops客户端，用作一个被AOps服务纳管监控的主机，下述流程中该机器需同时部署aops-ceres和gala-gopher，建议内存4G+。

| 机器编号 | 配置IP      | 部署模块                                                     |
| -------- | ----------- | ------------------------------------------------------------ |
| 机器A    | 192.168.1.1 | mysql，elasticsearch，kafka，prometheus，aops-diana, redis  |
| 机器B    | 192.168.1.2 | aops-zeus，aops-apollo，aops-diana，aops-hermes，gala-ragdoll |
| 机器C    | 192.168.1.3 | aops-ceres，gala-gopher                                      |

# 三、服务端部署

## 3.1、 机器管理

使用机器管理功能需部署aops-zeus、aops-hermes以及mysql服务。

每台机器在部署前，请先关闭防火墙。

```
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld
```

### 3.1.1、节点信息

| 机器编号 |  配置IP|部署模块|
| -------- | -------- | -------- |
| 机器A | 192.168.1.1 |mysql，aops-tools, prometheus, redis|
| 机器B | 192.168.1.2 |aops-zeus，aops-vulcanus|
| 机器C | 192.168.1.3 |aops-ceres，gala-gopher|

### 3.1.2、部署步骤

#### 3.1.2.1、 部署mysql

安装mysql：

```
yum install mysql-server
```

修改mysql配置文件：

```bash
vim /etc/my.cnf
```

新增bind-address，值为本机ip 

```ini
[mysqld]
bind-address=192.168.1.1
```

重启mysql服务：

```bash
systemctl restart mysqld
```

连接数据库，设置root用户访问权限，创建aops数据库：

```
[root@localhost ~]# mysql

mysql> show databases;
mysql> use mysql;
mysql> select user,host from user;

+---------------+-----------+
| user          | host      |
+---------------+-----------+
| root          | localhost | //此处出现host为localhost时，说明mysql只允许本机连接，外网和本地软件客户端则无法连接。
| mysql.session | localhost |
| mysql.sys     | localhost |
+---------------+-----------+
3 rows in set (0.00 sec)
```

```
mysql> update user set host = '%' where user='root'; //设置允许root用户任意IP访问。
mysql> flush privileges;//刷新权限
mysql> create database aops default character set utf8mb4 collate utf8mb4_unicode_ci;  //创建aops数据库
mysql> exit
```

#### 3.1.2.2、 部署prometheus

安装prometheus:

```
yum install prometheus2
```

修改配置文件：

```
vim /etc/prometheus/prometheus.yml
```

将所有客户端的gala-gopher地址新增到prometheus的监控节点中。

```
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090', '192.168.1.3:8888'] //本指南中机器C用于部署客户端，故添加机器C的gala-gopher地址
```

启动服务：

```
systemctl start prometheus
```

#### 3.1.2.3、 部署redis

安装redis：

```
yum install redis
```

修改配置文件：

```
vim /etc/redis.conf
```

添加绑定IP：

```
# It is possible to listen to just one or multiple selected interfaces using
# the "bind" configuration directive, followed by one or more IP addresses.
#
# Examples:
#
# bind 192.168.1.100 10.0.0.1
# bind 127.0.0.1 ::1
#
# ~~~ WARNING ~~~ If the computer running Redis is directly exposed to the
# internet, binding to all the interfaces is dangerous and will expose the
# instance to everybody on the internet. So by default we uncomment the
# following bind directive, that will force Redis to listen only into
# the IPv4 lookback interface address (this means Redis will be able to
# accept connections only from clients running into the same computer it
# is running).
#
# IF YOU ARE SURE YOU WANT YOUR INSTANCE TO LISTEN TO ALL THE INTERFACES
# JUST COMMENT THE FOLLOWING LINE.
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bind 127.0.0.1 192.168.1.1 //此处添加机器A的真实IP 
```

启动redis服务：

```
systemctl start redis
```



#### 3.1.2.4、 部署aops-zeus

安装aops-zeus：

```
yum install aops-zeus
```

修改配置文件：

```
vim /etc/aops/zeus.ini
```

将配置文件中各服务的地址修改为真实地址，本指南中aops-zeus部署于机器B，故需把IP地址配为机器B的ip地址。

```
[zeus]
ip=192.168.1.2  // 此处ip修改为机器B真实ip
port=11111
host_vault_dir=/opt/aops
host_vars=/opt/aops/host_vars

[uwsgi]
wsgi-file=manage.py
daemonize=/var/log/aops/uwsgi/zeus.log
http-timeout=600
harakiri=600
processes=2		// 生成指定数目的worker/进程
threads=4		// 每个worker中使用线程数目

[mysql]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=3306
database_name=aops
engine_format=mysql+pymysql://@%s:%s/%s
pool_size=100
pool_recycle=7200

[prometheus]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=9090
query_range_step=15s

[agent]
default_instance_port=8888

[redis]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=6379

[diana]
ip=192.168.1.2  // 此处ip修改为机器B真实ip
port=11112

[apollo]
ip=192.168.1.2  // 此处ip修改为机器B真实ip
port=11116
```

启动aops-zeus服务：

```
systemctl start aops-zeus
```

#### 3.1.2.5、 部署aops-hermes

安装aops-hermes：

```
yum install aops-hermes
```

修改配置文件，由于将所有服务都部署在机器B，故需将web访问的各服务地址配置成机器B的真实ip。

```
vim /etc/nginx/aops-nginx.conf
```

部分服务配置展示：

```

        # 保证前端路由变动时nginx仍以index.html作为入口
        location / {
            try_files $uri $uri/ /index.html;
            if (!-e $request_filename){
                rewrite ^(.*)$ /index.html last;
            }
        }

        location /api/ {
            proxy_pass http://192.168.1.2:11111/; //此处修改为aops-zeus部署机器真实IP
        }

        location /api/domain {
            proxy_pass http://192.168.1.2:11114/; //此处IP对应gala-ragdoll的IP地址
            rewrite ^/api/(.*) /$1 break;
        }
        
        location /api/check {
            proxy_pass http://192.168.1.2:11112/; //此处IP对应configurable模式运行的aops-diana的IP地址
            rewrite ^/api/(.*) /$1 break;
        }
		
        location /api/vulnerability {
            proxy_pass http://192.168.1.2:11116/; //此处IP对应aops-apollo的IP地址
            rewrite ^/api/(.*) /$1 break;
        }
```

开启aops-hermes服务：

```
systemctl start aops-hermes
```

## 3.2、 CVE管理

CVE管理功能在机器管理的基础上实现，故在部署CVE管理功能前须完成机器管理部分的部署，然后再部署aops-apollo。

数据服务部分aops-apollo服务的运行需要mysql、elasticsearch数据库的支持。

### 3.2.1、 节点信息

| 机器编号 | 配置IP      | 部署模块                                           |
| -------- | ----------- | -------------------------------------------------- |
| 机器A    | 192.168.1.1 | mysql，elasticsearch，redis                        |
| 机器B    | 192.168.1.2 | aops-zeus，aops-apollo，aops-hermes，aops-vulcanus |
| 机器C    | 192.168.1.3 | aops-ceres                                         |

### 3.2.2、部署步骤

#### 3.2.2.1、 部署基础服务

见 3.1 

#### 3.2.2.2、 部署elasticsearch

生成elasticsearch的repo源：

```shell
echo "[aops_elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md" > "/etc/yum.repos.d/aops_elascticsearch.repo"
```

通过yum命令安装：

```shell
yum install elasticsearch-7.14.0-1
```

修改elasticsearch配置文件：

```shell
vim /etc/elasticsearch/elasticsearch.yml
```

```
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node-1	
```

```
# ---------------------------------- Network -----------------------------------
#
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
# 此处修改为机器A真实ip
network.host: 192.168.1.1 
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
http.port: 9200		
#
# For more information, consult the network module documentation.
#
```

```
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
#discovery.seed_hosts: ["host1", "host2"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
cluster.initial_master_nodes: ["node-1", "node-2"]
#
```

重启elasticsearch服务：

```
systemctl restart elasticsearch
```

#### 3.2.2.3、 部署apollo

安装aops-apollo：

```
yum install aops-apollo
```

修改配置文件：

```
vim /etc/aops/apollo.ini
```

将apollo.ini配置文件中各服务的地址修改为真实地址：

```
[apollo]
ip=192.168.1.2//此处修改为机器B的真实IP
port=11116
host_vault_dir=/opt/aops
host_vars=/opt/aops/host_vars

[zeus]
ip=192.168.1.2	//此处修改为机器B的真实IP
port=11111

; herems info is used to send mail.
[hermes]
ip=192.168.1.2	//此处修改为部署aops-hermes的真实IP,以机器B的IP地址为例
port=54795

[cve]
cve_fix_function=yum
# value between 0-23, for example, 2 means 2:00 in a day.
cve_scan_time=2

[mysql]
ip=192.168.1.1 //此处修改为机器A的真实IP
port=3306
database_name=aops
engine_format=mysql+pymysql://@%s:%s/%s
pool_size=100
pool_recycle=7200

[elasticsearch]
ip=192.168.1.1 //此处修改为机器A的真实IP
port=9200
max_es_query_num=10000000

[redis]
ip=192.168.1.1 //此处修改为机器A的真实IP
port=6379

[uwsgi]
wsgi-file=manage.py
daemonize=/var/log/aops/uwsgi/apollo.log
http-timeout=600
harakiri=600
processes=2
threads=4

```
启动aops-apollo服务：

```
systemctl start aops-apollo
```

## 3.3、 异常检测

异常检测功能离不开机器管理服务的支持，故在部署异常检测功能前须完成机器管理部分的部署，然后再部署aops-diana。

基于分布式部署考虑，aops-diana服务需在机器A与机器B同时部署，分别扮演消息队列中的生产者与消费者角色。

数据服务部分aops-diana服务的运行需要mysql、elasticsearch、kafka以及prometheus的支持。

### 3.3.1、 节点信息

| 机器编号 | 配置IP      | 部署模块                                            |
| -------- | ----------- | --------------------------------------------------- |
| 机器A    | 192.168.1.1 | mysql，elasticsearch，kafka，prometheus，aops-diana |
| 机器B    | 192.168.1.2 | aops-zeus，aops-diana，aops-hermes，aops-vulcanus   |
| 机器C    | 192.168.1.3 | aops-ceres，gala-gopher                             |

### 3.3.2、 部署步骤

#### 3.3.2.1、 部署基础服务

见 3.1

#### 3.3.2.2、 部署elasticsearch

见 3.2.2.2

#### 3.3.2.3、 部署kafka

kafka使用zooKeeper用于管理、协调代理，所以部署kafka时还需部署zookeeper服务。

##### 3.3.2.3.1、 安装zookeeper

安装：

```
yum install zookeeper
```

启动服务：

```
systemctl start zookeeper
```

##### 3.3.2.3.2、 安装kafka

安装：

```
yum install kafka
```

修改配置文件：

```
vim /opt/kafka/config/server.properties
```

将listener 改为本机ip

```
############################# Socket Server Settings #############################

# The address the socket server listens on. It will get the value returned from
# java.net.InetAddress.getCanonicalHostName() if not configured.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
listeners=PLAINTEXT://192.168.1.1:9092
```

启动kafka服务: 

```
cd /opt/kafka/bin
nohup ./kafka-server-start.sh ../config/server.properties &
tail -f ./nohup.out  # 查看nohup所有的输出出现A本机ip 以及 kafka启动成功INFO；
```

#### 3.3.2.4、 部署diana

机器A与机器B中aops-diana安装流程没有任何差别。

安装aops-diana：

```
yum install aops-diana
```

修改配置文件，机器A与机器B中aops-diana分别扮演不同的角色，通过配置文件的差异来区分两者扮演角色的不同。

```
vim /etc/aops/diana.ini
```

（1）机器A中aops-diana以executor模式启动，扮演kafka消息队列中的消费者角色，配置文件需修改部分如下所示：

```
[diana]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=11112
mode=executor  // 该模式为executor模式，用于常规诊断模式下的执行器，扮演kafka中消费者角色。
timing_check=on

[default_mode]
period=60
step=60

[elasticsearch]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=9200
max_es_query_num=10000000

[mysql]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=3306
database_name=aops
engine_format=mysql+pymysql://@%s:%s/%s
pool_size=100
pool_recycle=7200

[redis]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=6379

[prometheus]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=9090
query_range_step=15s

[agent]
default_instance_port=8888

[zeus]
ip=192.168.1.2  // 此处ip修改为机器B真实ip
port=11111

[consumer]
kafka_server_list=192.168.1.1:9092  // 此处ip修改为机器A真实ip
enable_auto_commit=False
auto_offset_reset=earliest
timeout_ms=5
max_records=3
task_name=CHECK_TASK
task_group_id=CHECK_TASK_GROUP_ID
result_name=CHECK_RESULT

[producer]
kafka_server_list = 192.168.1.1:9092  // 此处ip修改为机器A真实ip
api_version = 0.11.5
acks = 1
retries = 3
retry_backoff_ms = 100
task_name=CHECK_TASK
task_group_id=CHECK_TASK_GROUP_ID

[uwsgi]
wsgi-file=manage.py
daemonize=/var/log/aops/uwsgi/diana.log
http-timeout=600
harakiri=600
processes=2
threads=2
```

（2）机器B中diana以configurable模式启动，扮演kafka消息队列中的生产者角色，aops-hermes中关于aops-diana的端口配置以该机器ip与端口为准，配置文件需修改部分如下所示：

```
[diana]
ip=192.168.1.2  // 此处ip修改为机器B真实ip
port=11112
mode=configurable  // 该模式为configurable模式，用于常规诊断模式下的调度器，充当生产者角色。
timing_check=on

[default_mode]
period=60
step=60

[elasticsearch]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=9200
max_es_query_num=10000000

[mysql]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=3306
database_name=aops
engine_format=mysql+pymysql://@%s:%s/%s
pool_size=100
pool_recycle=7200

[redis]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=6379

[prometheus]
ip=192.168.1.1  // 此处ip修改为机器A真实ip
port=9090
query_range_step=15s

[agent]
default_instance_port=8888

[zeus]
ip=192.168.1.2  // 此处ip修改为机器B真实ip
port=11111

[consumer]
kafka_server_list=192.168.1.1:9092  // 此处ip修改为机器A真实ip
enable_auto_commit=False
auto_offset_reset=earliest
timeout_ms=5
max_records=3
task_name=CHECK_TASK
task_group_id=CHECK_TASK_GROUP_ID
result_name=CHECK_RESULT

[producer]
kafka_server_list = 192.168.1.1:9092  // 此处ip修改为机器A真实ip
api_version = 0.11.5
acks = 1
retries = 3
retry_backoff_ms = 100
task_name=CHECK_TASK
task_group_id=CHECK_TASK_GROUP_ID

[uwsgi]
wsgi-file=manage.py
daemonize=/var/log/aops/uwsgi/diana.log
http-timeout=600
harakiri=600
processes=2
threads=2
```

启动aops-diana服务：

```
systemctl start aops-diana
```

## 3.4、 配置溯源

A-Ops配置溯源在机器管理的基础上依赖gala-ragdoll实现，同样在部署gala-ragdoll服务之前，须完成机器管理部分的部署。

### 3.4.1、 节点信息

| 机器编号 | 配置IP      | 部署模块           |
| -------- | ----------- | ------------------ |
| 机器A    | 192.168.1.1 | mysql，aops-tools  |
| 机器B    | 192.168.1.2 | aops-zeus，gala-ragdoll，aops-hermes，aops-vulcanus |
| 机器C    | 192.168.1.3 | aops-ceres         |

### 3.4.2、 部署步骤

#### 3.4.2.1、 部署基础服务

见 3.1

#### 3.4.2.2、 部署gala-ragdoll

安装gala-ragdoll：

```shell
yum install gala-ragdoll python3-gala-ragdoll
```

修改配置文件：

```shell
vim /etc/ragdoll/gala-ragdoll.conf
```

将collect节点collect_address中IP地址修改为机器B的地址，collect_api与collect_port修改为实际接口地址。

```
[git]
git_dir = "/home/confTraceTest"
user_name = "user_name"
user_email = "user_email"

[collect]
collect_address = "http://192.168.1.2"    //此处修改为机器B的真实IP
collect_api = "/manage/config/collect"    //此处接口原为示例值，需修改为实际接口值/manage/config/collect
collect_port = 11111                      //此处修改为aops-zeus服务的实际端口

[sync]
sync_address = "http://192.168.1.2"
sync_api = "/demo/syncConf"
sync_port = 11114


[ragdoll]
port = 11114

```

启动gala-ragdoll服务：

```shell
systemctl start gala-ragdoll
```

## 3.5、一键化部署

A-Ops服务也支持一键化部署，这部分依赖aops-tools中的脚本实现，具体使用见[A-Ops一键化部署指南](AOps一键化部署指南.md)

# 四、客户端部署

客户端依赖aops-ceres模块实现，部分数据采集依赖gala-gopher完成，关于这部分的部署，见[aops-ceres部署指南](aops-ceres部署指南.md)

# FAQ：

1. 管理中心（aops-zeus）与各模块之间通过http协议进行连接通信。
2. 若防火墙不方便关闭，请设置放行服务部署过程涉及的所有接口，否则会造成服务不可访问，进而影响A-Ops的正常使用。
3. 基于安全角度考虑，机器管理中添加新机器的方式，由客户端主动发起，具体操作请参考[aops-ceres部署指南](aops-ceres部署指南.md)关于注册部分。
4. 异常检测，配置溯源，CVE管理模块相互独立，用户实际使用中可根据需求选择部署。
5. 所有服务部署分为服务端与客户端两部分，服务端部署参考具体功能需求进行部署即可，客户端只需部署aops-ceres和gala-gopher即可。
