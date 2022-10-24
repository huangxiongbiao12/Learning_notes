#### 1、集群节点的感知和健康探测



ProtocolManager 一致性协议管理

​	getCpProtocol -》initCPProtocol-》动态获取CPProtocol.class -》init protocol

​	





ServerMemberManager  集群节点管理

初始化节点：

  serverList 集群列表：启动的时候 将本节点加入，

​		启动initAndStartLookup    FileConfigMemberLookup 从配置文件cluster.conf中读取集群的ip:port信息

定期更新节点：

​		GlobalExecutor 定期执行 MemberInfoReportTask  轮询向serverList中的节点发送当前节点信息更新/cluster/report



ServerListManager	集群节点管理 已废弃

-  This object will be deleted sometime after version 1.3.0

​													ServerInfoUpdater轮询 /cluster/state 更新serverList中的一个节点信息

​									延迟2s执行	ServerStatusReporter  向所有的节点发送状态信息/operator/server/status





#### 2、配置写操作，集群一致性同步



ConfigController -》publishConfig -》persistService.insertOrUpdate-》发布事件ConfigDataChangeEvent -》事件监听AsyncNotifyService处理-》 长链接AsyncRpcTask

​												不支持长链接AsyncTask



persistService -》持久化分为内嵌数据库，和外部数据库仅支持mysql

​	EmbeddedStoragePersistServiceImpl内嵌数据库，集群模式  需要通过 CPProtocol同步数据

​	ExternalStoragePersistServiceImpl外部数据库，没有节点分布  不需要同步各节点数据



内嵌数据库时  DatabaseOperate  databaseOperate.blockUpdate

​					集群模式DistributedDatabaseOperateImpl  通过 CPProtocol同步数据

​					单机模式StandaloneDatabaseOperateImpl 执行sql



持久化之后对集群节点广播发送变化事件

事件监听ConfigDataChangeEvent 异步通知变化 dataId, group, tenant, tag

​	ConfigExecutor.executeAsyncNotify    http AsyncTask  长链接AsyncRpcTask

不支持长链接AsyncTask 调用接口 广播

​			http接口	/v1/cs/communication/dataChange   

  		 接口处理						dumpService.dump

​					TaskManager-》DumpProcessor -》persistService 数据库查出配置信息-》DumpConfigHandler-》ConfigCacheService 更新缓存MD5信息-》LocalDataChangeEvent 通知改变-》LongPollingService-》DataChangeTask-》allSubs执行改变通知



支持长链接AsyncRpcTask  rpc调用ConfigChangeClusterSyncRequest

​	ConfigChangeClusterSyncRequestHandler-》dumpService.dump -》TaskManager-》DumpProcessor -》DumpConfigHandler.configDump-》ConfigCacheService.dump-》updateMd5-》LocalDataChangeEvent-》RpcConfigChangeNotifier监听configDataChanged-》长链接客户端监听listeners-》RpcPushTask->返回长链接监听端信息

返回失败

ConfigChangeNotifyRequest-》ClientWorker监听-》notifyListenConfig-》executeConfigListen-> rpc请求更新数据







#### 3、服务注册，集群数据一致性同步



#### 4、客户端服务注册发现



#### 5、客户端配置动态感知

不支持长链接客户端

ConfigController /listener接口 

LongPollingService addLongPollingClient-》ClientLongPolling-》收到LocalDataChangeEvent 变化allSubs 提前返回-》否则等长轮询时间结束执行

请求头 包含Long-Pulling-Timeout  则长轮询 此参数时间减0.5s

http 请求  最长9.5s长轮询 收到变化 提前返回，



支持长连接









ClientWorker  缓存cacheMap  没有的话  去服务器查询

监听变换 ClientWorker-》 start -》startInternal 5s一次有事件通知立即执行-》executeConfigListen

5s 查询一次服务器是否有变化

服务器有变化  notifyListenConfig 也会主动推送过来 主动推送ConfigChangeNotifyRequest





#### 6、事件驱动设计



#### 7、集群和单节点启动设计

PersistService  		

​	EmbeddedStoragePersistServiceImpl 单节点持久化

​	ExternalStoragePersistServiceImpl 集群持久化



利用ConditionOnExternalStorage、ConditionOnEmbeddedStorage动态加载 

默认启动参数nacos.standalone   为true单节点默认内嵌数据库  false 集群默认外部数据库

配置参数  spring.datasource.platform=mysql 使用外部数据库

启动参数 embeddedStorage



#### 8、rpc设计



