NiFi使用说明书
=========
#1.安装说明#
##1.1. 准备##
1. Java 版本至少为1.8。
2. 安装包：前往 [http://nifi.apache.org/download.html](http://nifi.apache.org/download.html "下载链接")下载，以下以1.0.0为例。
##1.2. 安装##
1. nifi默认web端口是8080，可以前往./conf/nifi.properties修改，以8082为例 ：
	`nifi.web.http.port=8082` 
2. ./bin下有nifi.sh 可以启动服务
	- start: 在后台启动 NiFi
	- 	stop: 停止在后台运行的 NiFi
	- 	status:提供 NiFi 的当前状态
	- 	run: 在前台运行 NiFi 并等待关机 NiFi Ctrl + C
	- 	install: 安装 NiFi 作为一种服务，然后可以通过控制服务来启停nifi
		- 	service nifi start
		- 	service nifi stop
		- 	service nifi status
	
	例如：
	- 启动nifi服务：nifi.sh start
	- 停止nifi服务：nifi.sh stop

		 
#2. 界面介绍
![](http://i.imgur.com/KRWj31u.png)
## 2.1. Components Toolbar(组件工具栏)
1. 从左向右依次是处理器，输入端口，输出端口，处理器组，远程处理器组，漏斗，模板，标签
2. 这些基本组件可以组成数据流，下面会详细的介绍。
## 2.2. Status Bar(状态栏）
 状态栏提供画布上的多个处理器的信息，如果是集群，也可以看到集群的连接状态等信息。
## 2.3. Operate Palette(操作台）
1. 查看组件的配置信息
2. 启停 processor/group 的快捷方式
## 2.4. search(查找栏）
可以通过搜索查找画布的上组件
## 2.5. Global Menu(菜单)
现有组件的更多选项菜单

# 3. 优化方式#
## 设置缓存文件##
为预防队列缓存中数据过大，导致内存全部消耗完，可以将nifi的缓存文件指向大容量的目录下，将nifi的安装目录`./conf/nifi.properties` 的  `nifi.content.repository.directory.default` 属性（默认是 `./content_repository`）改为
`/data/nifi_repository/content_repository`

# 4.  简单 dataflow 实例
## 4.1. 案例1：将FTP （172.10.1.102）上指定文件上传到测试集群的 HDFS 上
描述：本实例只涉及到简单的 processor 之间的连接

步骤：

###1. 第一步：新建Proecss Group 组件：在nifi的画布上新加 Process Group,拖拽![](http://i.imgur.com/wu368Lx.png)，将group命名为 GETFTP->PUTHDFS
![](http://i.imgur.com/QDdKtVy.png)


###2. 第二步：在Group 中新建 GETFTP processor 组件：双击 group 进入，拖拽 processor ![](http://i.imgur.com/ZRcEoVm.png) 按钮，添加 GETFTP processor
1. 搜索getftp组件（或 单击右边的关键字筛选），将其 ADD 到 画布上，添加的processor 是无效的，因为 GETFTP processor 必须参数还未配置
![](http://i.imgur.com/9W6zBZx.png)

2. 右击添加的processor 可以看到 processor的配置，历史数据，数据出处，使用方法，改变颜色，复制，删除等选项
![](http://i.imgur.com/zmwyWj8.png)


3. 点击configure，对GETFTP进行设置，设置选项有4项：
	1.  SETTINGS 基本设置
![](http://i.imgur.com/P8Q9fTc.png)

	2.  SCHEDULING schedule的方式选项，我们使用默认的driven方式
![](http://i.imgur.com/jb1HRuP.png)

	3.   PROPERTIES 参数配置，加粗的参数是必填项，有些有默认值，具体用法可以参考，processor 的 usage，有些支持java 正则表达式
![](http://i.imgur.com/sLi8qY6.png)

	4.   COMMENTS 
![](http://i.imgur.com/LL5nGxi.png)

###3. 第三步：在Group 中新建 PUTHDFS processor 组件：拖拽 processor ![](http://i.imgur.com/ZRcEoVm.png) 按钮，添加 PUTHDFS processor
添加PUTHDFS 的方法 同第二步添加GETFTP 的方法相同，通过配置信息，使processor 有效，PUTHDFS 的配置如下：默认是本机的hdfs，这里只需要把测试集群的配置文件拷贝到装nifi的机器上就可以
![](http://i.imgur.com/wT3VBji.png)

###4. 第四步：拖拽GETFTP 中的连接 到 PUTHDFS 上
图一：![](http://i.imgur.com/JbRVrOB.png)
图二：![](http://i.imgur.com/EBJ8OBt.png)

###5. 第五步：flow 中传输的数据即为 flowfile，每个flowfile 都有去处：传输到下游组件和删除
###6. 第六步：更改 flow 中最后一个 processor 的 config 的 Auto terminate relationships（自动终止关系），勾选 failure 和 success。处理器每个关系必须连接到下游组件（即连接指向的组件）或自动终止。如果关系是自动终止，当 processor 完成处理时，则会自动删除当前 flow 中的 FlowFile。任何已连接到下游组件的关系不能自动终止。如：
1. 传输下游： GETFTP 的 终止关系为 success，如果不选中，则表示processor继续向下游的 processor 进行传输 flowfile 
2. 终止关系： PUTHDFS 的终止关系有 failure 和 success ，如果选中 两者，表示 该processor 不能向下游的进行 传输 flowfile，因为 PUTHDFS 没有下游 组件，所以这里要选中终止关系 failure 和 success ，即删除flow中的flowfile，否则报错。
![](http://i.imgur.com/7GxLSk6.png)
 在跳板机上的hosts 添加以下内容，因为Hadoop 的配置文件是由172.10.1.230 上面拷贝过来的，需要知道 HDP1-NN01 的ip是多少。

       ` 172.10.1.2 HDP1-NN01.panodata.cn HDP1-NN01
        172.10.1.3 HDP1-NN02.panodata.cn HDP1-NN02
		172.10.1.230    vHDP-NN01
		172.10.1.231    vHDP-DN01
		172.10.1.232    vHDP-DN02`
###7. 第七步：启动flow 进行数据传输，方式有两种
1. 	 选中 group ，点击 左下角的控制台中的启动group
![](http://i.imgur.com/oqyPcvm.png)

2. 	 进入group，选中画布，点击控制台中的启动group，也可以选中其中的processor 进行启停。

###本案例需要注意的问题
1. 在putHDFS processor 中 config 的Hadoop config Resourse:  values, 这个值是hdfs 的配置文件，路径是安装nifi这台机器的路径，可以将装有hdfs 机器上的两个配置文件core-site.xml 和hdfs-site.xml 拷贝到当前机器的路径中，例如：/nifi/nifi-1.0.0/hadoopconfig/core-site.xml,/nifi/nifi-1.0.0/hadoopconfig/hdfs-site.xml
2. nifi 中使用hive 可以不用拷贝hive的配置文件，因为在PutHiveQL 的cofig 中的hive connection中有主机ip。

## 4.2. 案例2：QueryDatabaseTable->PutHiveStreaming

描述：从mysql库中，取出指定表的数据，Query result will be converted to Avro format，可以直接将这个结果 输入到PutHiveStreaming processor中

###用到的driver

	1. mysqldriver name  :   com.mysql.jdbc.Driver
	2. mysql Database Connection URL   :   jdbc:mysql://192.168.178.51:3306/openser
	3. hive database connection url   :    jdbc:hive2://172.10.1.230:10000/default
	4. hive metastore url   :    thrift://172.10.1.230:9083
	

###1. add QueryDatabaseTable processor， QueryDatabaseTable config
1. Database connection pooling service
		* ![](http://i.imgur.com/9DOsJw1.png)
		* DBCPConnectionPool,如下图
		 ![](http://i.imgur.com/uVCf7nw.png)
		*
		* Database connection url:jdbc:mysql://192.168.178.51:3306/openser, mysql://host:port/database。
		* Driver: com.mysql.jdbc.Driver
		* Driver location :/nifi/jdbcDriver/mysql-connector-java-5.1.40-bin.jar，将jar放入目录中，设置jar的路径
		* Database user :ro_maoyu
		* password:***
2. Table name:数据库中的表
###2. add PutHiveStreaming processor，PutHiveStreaming config
1. ![](http://i.imgur.com/rQTHNpF.png)
1. Hive metastore : url:thrift://172.10.1.230:9083,可以在测试集群的ambari中查看
2. databasse name: hive 中默认的数据库是default
3. Max Open Connections: 这是buckets 个数，默认情况是10，
3. table name：这是先前在hive中建好的表，建表的语句如下
4. ![](http://i.imgur.com/ofZkA7c.png)

###总结
 注意仔细看文档，在puthiveStreaming中注意使用 hive streaming 的使用情况，可以去hive （[https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest#StreamingDataIngest-StreamingRequirements](https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest#StreamingDataIngest-StreamingRequirements "hive streaming 使用")）官网查。
##4.3. 案例3：GetFTP->解压-> PUTHDFS
描述： 从ftp获取数据，解压后，通过筛选将csv文件上传到hdfs。流程 processor：GetFTP -> CompressContent -> UnpackContent -> PUTHDFS ，每个processor 的详细配置信息，可以通过右击-usage来查看

GetFTP :从ftp获取想要的数据

CompressContent ：用来压缩或解压的processor

UnpackContent ：用来filter文件

PUTHDFS：将文件上传至hdfs

1. GetPFT config
	1. ![](http://i.imgur.com/vtWtlNj.png)

2. CompressContent config
	1. ![](http://i.imgur.com/qHT6ap2.png)

3. UnpackContent config
	1. ![](http://i.imgur.com/dZQ2aqS.png)
	
4. PUTHDFS config
	1. ![](http://i.imgur.com/a3cXlOd.png)

##4.4. 案例4：GetFTP -> PutHiveQL
描述：从ftp获取hive_ddl.txt，这个文件保存着sql语句，获取的语句可以放到PutHiveQL中执行。

1. putHiveQL:The content of an incoming FlowFile is expected to be the HiveQL command to execute,

2. putHiveQl config：  Hive Database Connection Pooling Service:Hive Database Connection Pooling Service

3. service config：
![](http://i.imgur.com/zP17DfH.png)

4. getFtp:获取的文件是下图，注意语句的末尾没有分号。
![](http://i.imgur.com/ccc6A7I.png)

#5. 使用系统参数自定义 processor 的属性
在第四节已经详细说明 FTP->HDFS->alter table add partition。不过其中的属性值都是固定的。现在可以利用系统参数来配置 processor 的属性。

1. 在nifi 的目录下./conf/nifi.properties 中设置 
	1. `nifi.variable.registry.properties=./conf/custom.properties,./conf/system.properties`
2. custom.properties 和system.properties 都是新建的文件，在该文件中可以写参数，以及value，
3. custom.properties 文件
	1. nifi.custom.variable.1=yes
	2. nifi.custom.attributes.dir=/Users/ydavis/dev/tools/attributes
	3. hostnameftp=172.10.1.102

4. 在processor 中的config 中，如果某个属性 Supports Expression Language: true，那么可以将custom.properties 的参数写入其中，例如：getFTP 的 config 的 hostname 可以写成：${hostnameftp}
5. 注意：custom 自定义的属性要保证包含不同的属性值，以确保它们不会重写其他属性变量或系统，环境和流程文件属性。











--------------------------------------
--------------------------------------
--------------------------------------


开发者参考文档：
===
#1. 获取group data 的详细历史信息，从这些中有可能获取stop的时间
描述：为了判断任务是否done，进而是否可以stop group（以GetFTP -> PutHDFS为例），下列方法都具有是否可以判断任务完成的可能性。

1. Gets status history for a remote process group, Determines whether the Processor Group tasks to complete:
	1. url： http://nifi.apache.org/docs/nifi-docs/rest-api/index.html，
 	[Flow]:GET /flow/process-groups/{id}/status/history
	2. 从中可以获得该group的最新一天的传输数据，获取可以根据这个判断该任务是否跑完。
	3. http://localhost:18082/nifi-api/flow/process-groups/b88fe328-0157-1000-ae07-31ae4ce7c065/status/history

2. 获取数据传输列表：http://localhost:18082/nifi/provenance，监测的数据流：（从Nifi界面操作）
	1. 进入的方法：在nifi界面的右上角  更多->Data Provenance
	1. 从ftp的获取的文件列表
	2. 流经每个 processor 的数据流
	3. 每个flowfile 都有一个共同的uuid。
	4. 每个data provenance event 都有type,这个值表示：Each point in a dataflow where a FlowFile is processed in some way is considered a "processing event". Various types of processing events occur, depending on the dataflow design. For example, when data is brought into the flow, a RECEIVE event occurs, and when data is sent out of the flow, a SEND event occurs. Other types of processing events may occur, such as if the data is cloned (CLONE event), routed (ROUTE event), modified (CONTENT_MODIFIED or ATTRIBUTES_MODIFIED event), split (FORK event), combined with other data objects (JOIN event), and ultimately removed from the flow (DROP event).
	5. ![](http://i.imgur.com/QGml1s7.png)


3. Reporting Task Monitoring tasks，详细介绍reportingtask 见下。
4. 综上以及网上
	1. http://localhost:18082/nifi-api/flow/process-groups/b88fe328-0157-1000-ae07-31ae4ce7c065
	2. POST https://{host}:{port}/nifi-api/controller/provenance
	3. GET https://{host}:{port}/nifi-api/controller/provenance/{provenance-request-id}
	4. ... (as many GETs as necessary to complete) ...
	5. DELETE https://{host}:{port}/nifi-api/controller/provenance/{provenance-request-id}


	
#2. Reporting Task
描述： Reporting Tasks run in the background to provide statistical reports about what is happening in the NiFi instance. 

1. The ReportingTask interface is a mechanism that NiFi exposes to allow metrics, monitoring information, and internal NiFi state to be published to external endpoints, such as log files, e-mail, and remote web services.
2. 有很多已有的reporting task ，为了达到想要的目的，可以自定义reporting task，
3. When developing a ReportingTask, it can call reportingContext.getEventAccess().getProvenanceRepository().
4. 这种custom reporting task（control service） 较繁琐，和 custom rocessor 类似，[http://www.nifi.rocks/developing-a-custom-apache-nifi-controller-service/](http://www.nifi.rocks/developing-a-custom-apache-nifi-controller-service/ "Developing a Custom Apache Nifi Controller Service")，[http://www.nifi.rocks/developing-a-custom-apache-nifi-processor-json/](http://www.nifi.rocks/developing-a-custom-apache-nifi-processor-json/ "Developing a Custom Apache Nifi Processor (JSON)")。
5. 访问reporting task 的url: http://localhost:18082/nifi-api/reporting-tasks/044d31f5-0158-1000-1357-c5164832728c


#3.获取ftp的文件列表
描述：http://localhost:18082/nifi/provenance（post），这是监测的数据流

1. 目前只能从界面中获得。
2. 从中筛选processor type 是GetFTP且 event type值是 receive 的 event ，表示有数据进入 flowfile 流，也就是从ftp获取的数据。（当然不排除是其他processor 处理的数据）
3. 从中筛选processor type 是 PutHDFS 且 event type值是 delete 的 event，表示从 flowfile 流中删除，也就是上传到hdfs中的数据。

#4.关于NiFi
描述：有nifi的版本，server，timezone等消息

1. http://localhost:18082/nifi-api/flow/about
2. {"about":{"title":"NiFi","version":"1.0.0","uri":"http://localhost:18082/nifi-api/","contentViewerUrl":"/nifi-content-viewer/","timezone":"CST"}}

#5.Stop processorGroup
描述：监控group中每个processor的状态，如果没有数据的流动，则认为group 完成任务。

1. 根据 groupId 获取 processors
2. 根据每个processorId 判断指定时间内（设两分钟）有无event。url:POST http://localhost:18082/nifi/provenance, [post content]`{"provenance":{"request":{"maxResults":1000,"startDate":"11/01/2016 00:00:00 CST","endDate":"11/01/2016 23:59:59 CST","searchTerms":{"ProcessorID":"b7ac3ccc-0157-1000-d0d6-c9291d9b5b29"}}}}`
3. 根据processorid 获取的event列表，取出最新执行的event，比较执行的时间与现在的时间，间隔五分钟内没有相应的event ，表示该processor五分钟内没有数据的流动，即该processor 完成数据的传输
4. 每个 processor 都已完成，则该 Group 完成任务。

注意： 需要设置flow file first processor 的 Run Schedule 时间，例如 5 min，默认是 0 sec means the Processor should run as often as possible as long as it has data to process.