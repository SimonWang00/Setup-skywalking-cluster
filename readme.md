# 分布式追踪系统skywalking集群搭建
SkyWalking 是一个开源 APM 系统，包括针对 Cloud Native 体系结构中的分布式系统的监视，跟踪，诊断功能。

----------
## 目录
* [1.概述](## 1.概述)
	* 1.1 系统依赖
    * 1.2 安装包
    * 1.3 collector集群
* [2.快速部署](## 2.快速部署)
    * 2.1 集群依赖安装
    * 2.2 下载安装程序
    * 2.3 解压collector到/applications目录
    * 2.4 配置collector连接zookeeper
    * 2.5 配置collector连接zookeeper
    * 2.6 修改collector启动脚本
    * 2.7 远程拷贝程序到其它节点
    * 2.8 启动collecoter
* [3.部署UI](## 3.部署UI)
    * 3.1 远程拷贝安装程序到可视化服务器
    * 3.2 配置UI
    * 3.3 修改UI启动脚本
    * 3.4 启动UI
    * 3.5 验证
* [4.验证服务](## 4.验证服务)
	* 4.1 maven构建SpringBootDemo
    * 4.2 部署javaagent
    * 4.3 配置agent
    * 4.4 配置agent
    * 4.5 请求服务
    * 4.6 UI查看结果
* [参考链接](## 参考链接) 

----------

### 1. 概述：
SkyWalking 创建与2015年，提供分布式追踪功能。从5.x开始，项目进化为一个完成功能的Application Performance Management系统。他被用于追踪、监控和诊断分布式系统，特别是使用微服务架构，云原生或容积技术。提供以下主要功能：
- 分布式追踪和上下文传输
- 应用、实例、服务性能指标分析
- 根源分析
- 应用拓扑分析
- 应用和服务依赖分析
- 慢服务检测
- 性能优化

[点击了解skywalking技术原理](https://github.com/apache/skywalking)

![分布式架构图](./pictures/jgt.jpg)

#### 1.1 系统依赖
- Java 8
- apache-skywalking-apm-es7
- Centos 7
- zookeeper
- elasticsearch 7（官方推荐）
- maven

#### 1.2 安装包
[点击获取skywalking安装程序](./install/apache-skywalking-apm-es7-8.0.1.tar.gz)，大小约136M左右。

#### 1.3 collector集群
CPU： 8  Intel(R) Xeon(R) CPU E5-2670 0 @ 2.60GHz
内存： 16G 
磁盘空间：245G

| 服务器         | function      |install path    |
| ------------- |:-------------:|:-------------:|
| 10.101.3.107  |  collector  |/applications |
| 10.101.3.108  |  collector  |/applications |
| 10.101.3.109  |  collector  |/applications |
| 10.101.3.106  |  UI可视化	  |/applications |

### 2.快速部署：
> 安装思路：先安装好zookeeper集群，再安装好elasticsearch集群，方可正式进入skywalking集群的部署。部署skywalking集群的方式是先逐台安装，然后再远程拷贝即可。单台安装策略：先安装collector，启动collector，然后对ui部署，启动ui部分，最后加入agent验证测试。

#### 2.1 集群依赖安装
skywalking集群主要依赖如下：
1. jdk1.8，centos7上预先已经集成jdk 1.8.0_262-b10,所以这里不在单独安装。
1. zookeeper集群，详细安装方法，**请参考**[zookeeper集群搭建](https://github.com/SimonWang00/Setup-zookeeper-cluster/blob/main/readme.md)。
1. elasticsearch集群，详细安装方法，**请参考**[elasticsearch集群搭建](https://github.com/SimonWang00/Setup-elasticsearch-cluster)。


#### 2.2 下载安装程序
> 安装文档步骤，安装好2.1中的依赖后，请下载1.2中链接的安装包程序。

#### 2.3 解压collector到/applications目录
解压命令：

	tar zxvf apache-skywalking-apm-es7-8.0.1.tar.gz -C /applications

#### 2.4 配置collector连接zookeeper
> cluster下注释掉standalone，打开zookeeper相关,并配置zookeeper集群地址。

备份默认配置：

	# 备份默认配置
	cp apache-skywalking-apm-bin-es7/config/application.yml apache-skywalking-apm-bin-es7/config/application.yml.bak
	# 修改
	vim apache-skywalking-apm-bin-es7/config/application.yml

修改如下：
	
	cluster:
	  #selector: ${SW_CLUSTER:standalone}
	  #standalone:
	  # Please check your ZooKeeper is 3.5+, However, it is also compatible with ZooKeeper 3.4.x. Replace the ZooKeeper 3.5+
	  # library the oap-libs folder with your ZooKeeper 3.4.x library.
	  zookeeper:
	    nameSpace: ${SW_NAMESPACE:"SW_CLUSTER_ZK_NAMESPACE"}
	    #hostPort: ${SW_CLUSTER_ZK_HOST_PORT:localhost:2181}
	    hostPort: ${SW_CLUSTER_ZK_HOST_PORT:10.101.3.107:2181,10.101.3.108:2181,10.101.3.109:2181}
	    # Retry Policy
	    baseSleepTimeMs: ${SW_CLUSTER_ZK_SLEEP_TIME:1000} # initial amount of time to wait between retries
	    maxRetries: ${SW_CLUSTER_ZK_MAX_RETRIES:3} # max number of times to retry
	    # Enable ACL
	    enableACL: ${SW_ZK_ENABLE_ACL:false} # disable ACL in default
	    schema: ${SW_ZK_SCHEMA:digest} # only support digest schema
	    expression: ${SW_ZK_EXPRESSION:skywalking:skywalking}

本人项目的配置文件，[点击下载获取](./conf/application.yml)。

> 选择zookeeper集群，就要注释掉kubernetes、consul和etcd。

#### 2.5 配置collector连接elasticsearch
> storage下注释掉h2相关，打开elasticsearch相关。

继续编辑：

	# 修改
	vim apache-skywalking-apm-bin-es7/config/application.yml

修改如下：
	
	storage:
		elasticsearch7:
	    nameSpace: ${SW_NAMESPACE:""}
	    #clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
	    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:10.101.3.107:9200,10.101.3.108:9200,10.101.3.109:9200}
	    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
	    trustStorePath: ${SW_STORAGE_ES_SSL_JKS_PATH:""}
	    trustStorePass: ${SW_STORAGE_ES_SSL_JKS_PASS:""}
	    dayStep: ${SW_STORAGE_DAY_STEP:1} # Represent the number of days in the one minute/hour/day index.
	    user: ${SW_ES_USER:""}
	    password: ${SW_ES_PASSWORD:""}
	    secretsManagementFile: ${SW_ES_SECRETS_MANAGEMENT_FILE:""} # Secrets management file in the properties format includes the username, password, which are managed by 3rd party tool.
	    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1} # The index shards number is for store metrics data rather than basic segment record
	    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5} # Super data set has been defined in the codes, such as trace segments. This factor provides more shards for the super data set, shards number = indexShardsNumber * superDatasetIndexShardsFactor. Also, this factor effects Zipkin and Jaeger traces.
	    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
	    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:2000} # Execute the bulk every 2000 requests
	    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
	    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
	    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
	    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
	    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
	    profileTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_PROFILE_TASK_SIZE:200}
	    advanced: ${SW_STORAGE_ES_ADVANCED:""}


#### 2.6 修改collector启动脚本
修改初始化内存2G， 最大可用6G
	
	vim bin/oapService.sh
    JAVA_OPTS=”-Xms2G -Xmx6G”

注释掉webapp的启动

	 vim bin/startup.sh
     #!/usr/bin/env sh
     PRG=”$0”
     PRGDIR='dirname "$PRG"'
     OAP_EXE=oapService.sh
     #WEBAPP_EXE=webappService.sh
     "$PRGDIR/$OAP_EXE"
     #"$ PRGDIR/$ WEBAPP_EXE"

#### 2.7 远程拷贝程序到其它节点
启动指令：

	# 拷贝到108
	scp -r /applications/apache-skywalking-apm-bin-es7 root@10.101.3.108:/applications
	# 拷贝到109
	scp -r /applications/apache-skywalking-apm-bin-es7 root@10.101.3.109:/applications


#### 2.8 启动collector
> 在三台节点上分别进行启动操作。

启动指令：

	bin/startup.sh

启动成功后打印
![sucess](./pictures/sucess.png)

查看进程：

	ps -ax | grep  skywalking

![pid](./pictures/pid.png)

好了，collector服务启动就完成，后面就是可视化的部署安装。


### 3 部署UI
> 在UI部署在10.101.3.106上面，部署完成后访问http://10.101.3.106:8080/可视化界面。
> （在服务器上安装jdk1.8，由于系统模板已经集成jdk 1.8.0_221,这里不在单独安装）

#### 3.1 远程拷贝安装程序到可视化服务器
选择其中一个节点的安装目录，拷贝到10.101.3.106的指定目录，执行指令：

	scp -r /applications/apache-skywalking-apm-bin-es7 root@10.101.3.106:/applications

#### 3.2 配置UI
修改方式如下：
	
	vim apache-skywalking-apm-bin-es7/webapp/webapp.yml
	# 修改listOfServers选项
	listOfServers:10.101.3.107:12800,10.101.3.108:12800,10.101.3.109:12800

#### 3.3 修改UI启动脚本
注释掉oap服务的启动

	vim apache-skywalking-apm-bin-es7/bin/startup.sh

做如下修改：

	#!/usr/bin/env sh

	PRG="$0"
	PRGDIR=`dirname "$PRG"`
	OAP_EXE=oapService.sh
	#WEBAPP_EXE=webappService.sh
	
	"$PRGDIR"/"$OAP_EXE"
	
	#"$PRGDIR"/"$WEBAPP_EXE"

#### 3.4 启动UI
启动UI
	
	./startup.sh

#### 3.5 验证
查看UI启动日志

	 tail -f apache-skywalking-apm-bin-es7/logs/webapp.log

查看日志：
![UI-log](./pictures/UI-log.png)

### 4.验证服务：
> 选取Java的应用程序作为demo，进行分布式链路追踪的配置实例。

#### 4.1 maven构建SpringBootDemo

构建方法参考如下：
![maven_build](./pictures/maven_build.png)

#### 4.2 部署javaagent
> 具体过程：将apache-skywalking-apm-es7-8.0.1.tar.gz解压到target目录下，修改agent配置，然后启动服务。

如下：
![agent](./pictures/agent.png)

### 4.3 配置agent
> 配置在目录apache-skywalking-apm-bin-es7/agent/config下的agent.config。

修改配置如下：
	
	# 服务名称
	agent.service_name=${SW_AGENT_NAME:Java_app1_test}
	# collector的地址
	collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:10.101.3.107:11800,10.101.3.108:11800,10.101.3.109:11800}

### 4.4 启动服务
在target目录中，执行如下指令：
	java -javaagent:./apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar -jar ./spring-boot-demo-0.0.1-SNAPSHOT.jar

![app](./pictures/app.png)


### 4.5 请求服务
> 对应用发起请求，多刷几次。

### 4.6 UI查看结果
查看UI追踪如下：
![UI](./pictures/UI.png)

## 参考链接
- Skywalking的存储配置与调优：https://smooth.blog.csdn.net/article/details/96479544。
- Skywalking官方文档：https://github.com/apache/skywalking/blob/5.x/docs/README_ZH.md。
- 分布式skywalking安装部署总结：https://blog.csdn.net/CSDN_niuniu/article/details/104007423?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param



