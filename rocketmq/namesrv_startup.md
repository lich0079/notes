# NamesrvStartup step walk through


## Create NamesrvController

 * parse commandLine parameter
 * init NamesrvConfig object, it hold the kvConfigPath, configStorePath and rocketmqHome
 * init NettyServerConfig object, it includes network related config for name server
 * if user provide 'c' path in commandLine, then override configStorePath with it, also override properties in NamesrvConfig and NettyServerConfig which new value c provides
 * if user provide 'p' in commandLine, print all namesrvConfig and nettyServerConfig, then System.exit(0)
 * try to get property from commandLine to override default namesrvConfig
 * new NamesrvController with namesrvConfig and nettyServerConfig







## Start NamesrvController


### initialize
 
 * load namesrv/namesrv.properties file into KVConfigSerializeWrapper
 * new NettyRemotingServer as remotingServer
 * new DefaultRequestProcessor which handler for all incoming request
 * schedule a task scan not active broker every 10 sec
 * schedule a task print all kv config every 10 sec


### start

 * remotingServer start


## NamesrvController Structure

```
NamesrvController
├──NamesrvConfig
├──────rocketmqHome
├──────kvConfigPath
├──────configStorePath
├──NettyServerConfig
├──────listenPort
├──────serverWorkerThreads
├──────serverCallbackExecutorThreads
├──────serverSelectorThreads
├──────serverOnewaySemaphoreValue
├──────serverAsyncSemaphoreValue
├──────serverChannelMaxIdleTimeSeconds
├──────useEpollNativeSelector
├──ScheduledExecutorService             # to execute time fixed task
├──KVConfigManager
├──────configTable                      # <String/* Namespace */, HashMap<String, String>> 
├──RouteInfoManager
├──────topicQueueTable                  # <String/* topic */, List<QueueData>>
├──────brokerAddrTable                  # <String/* brokerName */, BrokerData>
├──────clusterAddrTable                 # <String/* clusterName */, Set<String/* brokerName */>>
├──────brokerLiveTable                  # <String/* brokerAddr */, BrokerLiveInfo>
├──────filterServerTable                # <String/* brokerAddr */, List<String>/* Filter Server */>
├──NettyRemotingServer                  # serve incoming request
├──BrokerHousekeepingService            # channelEventListener listen connection state and update routeInfo
├──ExecutorService                      # remotingExecutor, worker thread pool
├──Configuration
├──────allConfigs                       # runtime properties which can be get and update
├──FileWatchService                     # listener for reload SslContext

```