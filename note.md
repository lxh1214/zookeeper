#zookeeper-源码笔记#

1. 下载源码，在源码的路径下一般(zookeeper) 执行 ant eclipse 
2. ant 首先得安装, 对应当前 zookeeper 版本里的 build.xml 指定的 jdk 版本为1.7 保证当前环境下的 jdk 版本与build.xml一致
3. ant eclipse 成功之后 将项目引入 intellij 或者 eclipse 中.

##zookeeper入口-QuorumPeerMain##
    
   
   * QuorumPeerConfig 解析zookeeper 相关的配置文件 
    
    
    注： 1. 根据配置文件 可以解析到一下类 QuorumPeerConfig, 一个或者多个 QuorumServer (每个server.x 对应的 法定服务), QuorumVerifier 法定验证者.
         2. 对于配置文件的解析 处理通过固定路径path, 也可以通过 path 里的 dynamicConfigFileStr 属性给出的路径增加配置 但解析方式 没什么变化.
           
    + clientPort : 服务器提供客户端链接的访问的端口（默认2181）
    + tickTime : (滴答时间 单位 毫秒) zookeeper 配置的时间单位 以后所有设计到的时间配置都是以 tickTime的整数倍为配置　（官方给出的默认的配置为2000)
                    心跳包时间?
    + initLimit : 是指follower链接并同步到leader的初始化连接时间， initLimit为tickTime的倍数 (官方给出默认的配置为 10）
    + syncLimit : leader 通过心跳包检测机制在syncLimit的时间范围内收不到follower的响应则认为follower已经不在线了
                    syncLimit 为tickTime的倍数（官方给出的默认的配置为5) 
    + dataDir : 快照存储的路径.(快照是内存中的数据) 同时 myid 也在这个目录下 (异步写到磁盘中, 对内存中的数据没有影响）所以不需要单独的磁盘.
                默认情况下，事务日志也会存储在这个目录里.建议配置 dataLogDir (因为事务的写性能，直接影响到zk的性能)
                dataDir 必填项
    + dataLogDir : 事务日志输出路径， 尽量给事务日志给一个单独的磁盘或者挂载点, 对zk的性能会有所提高.
                    事务日志输出时是顺序且同步写到磁盘中， 只有从磁盘写完日志后才会触发follower 和 leader发的回调事务确认消息.
    + maxClientCnxns : 对单个zk服务可链接的 客户端数量限制（指的是ip的限制） 如果为 0 则不限制   默认为 60
    + autopurge.purgeIntreval : 3.4.0 后zk提供自动清理事务日志和快照的功能。 这里是以小时为单位。如果配置为0 则不做清理.
                                可通过bin/zkCleanup.sh 手动清理.
    + autopurge.snapRetainCount : 快照文件保留数 (默认为3）(如果参数没有配置或者小于3的话则取默认值 3)
    
    + minSessionTimeout,maxSessionTimeout : Session 超时时间限制. 如果"客户端"设置的超时时间不在 minSessionTimeout-maxSessionTimeout 范围内，
                                            则会被强制 超时范围在 minSessionTimeout-maxSessionTimeout 内, 如果"服务端"没有 minSessionTimeout,maxSessionTimeout
                                            默认情况下 2*tickTime - 20*tickTime.
    + electionAlg: 配置选举算法.(1. LeaderElection, 2. AuthFastLeaderElection, 3. FastLeaderElection) 默认的3.
    + peerType: follower 分两种 一种是 observer 一种 是 participant (观察者 和 参与者) (集群环境中应用)
                        observer 观察者不参与投票
    + clientPortAddress : 多网卡情况下 可为每个ip 配置成不同的 端口 默认的情况下和(clientPort 保持一致)
    
    + server.x=[hostname]:n:n:peerType : 集群配置 x表示myid, hostname(ip) 第一个n 表示follower 和 leader 数据间的同步和其他通信的端口, 第二个n表示选举过程中投票的通信端口.
                        默认为 server.myid=ip:2888:3888
                        peerType 为选填
                        
    + group.x=nnnnn[:nnnnn] 分组
         group.1=1:2:3   x 为组名,1:2:3 为sid 也就是myid
         group.2=4:5:6
         group.3=7:8:9
         也就是说上面 是有9个myid组成的
    + weight.x=nnnnn 权重
        weight.1=1
        weight.2=1
        weight.3=1
        weight.4=1
        weight.5=1
        weight.6=1
        weight.7=1
        weight.8=1
        weight.9=1
        x 为 sid 也就myid
        =1 为权重值 value
        
            配置文件扩展方式: zookeeper.key = value 放入System.property 
    + setupMyid
       根据配置文件dataDir目录下找到myid文件
       + 如果是standalone 单机模式的话 不用配置myid文件
       + 如果是集群模式的话 这配置myid -> 第一行输入一个数字 -> 这个数字就是 serverId
         如果不是数字将会抛出异常
         
    +  配置里的 clientPortAddress 客户端可访问的ip加端口
        "clientPortAddress" 是配置服务端的hostname
        如果没有配置
        QuorumPeerConfig 里真实的 clientPortAddress 根据配置文件里的"clientPortAddress" 如果没有配置在 hostname 为localhost 端口为 clientPort
        前提是clientPortAddress.getAddress().isAnyLocalAddress() 为 true
            clientPortAddress.getAddress().isAnyLocalAddress() 为本地的address 且 它的port 和 serverId 对应的QuorumServer 里的 clientAddr的port 需要一样.
            也就每个myid文件对应的myid 配置提供的clientPort 需要与 server.x解析出的 x对应的myid的端口保持一致.
        把 serverId对应的quorumServer.clientAddr 赋给 clientPortAddress
        
    + peerType
        QuorumVerifier.getObservingMembers().get(serverId) 存在则当前的 serverId 为 observer 模式也就是 观察者模式
        peerType 以  QuorumVerifier.getObservingMembers().get(serverId)推到推来的 为准.
        
    + checkValidity
        验证几个参数：
        首先 判断 是不是分布式的 也就是集群  QuorumVerifier != null && QuorumVerifier.getVotingMembers() > 1;
        分布式 是需要 1个以上的可投票者的
        如果是分布式 ：initLimit，syncLimit 必须有值 serverId 不能为 -1
        
    + backup-conf 文件
        如果是分布式 也就是集群的配置  需要备份当地zoo.cfg文件 且名字为 zoo.cfg.bak
                        
   * QuorumServer的生成策略
    
    
      +   根据server.x=[hostname]:port1:port2:peerType,可以确定
            addr -> hostname:port1
            electionAddr -> hostname:port2
            clientAddr -> 
            peerType -> type 
            myAddrs -> {addr, electionAddr, clientAddr}
        
   * QuorumVerifier 生成策略 (法定人数验证者)
      下面是 QuorumVerifer 的具体实现： 主要根据 配置文件 读取server.x=, group.x, weight.x, version
            几个参数来判断配置项里 所对应的
            allMembers 所有的myid
            votingMembers 参与者(也就可投票) myid
            observingMembers  观察者 不参与投票 myid
            1. 如果没有group 配置 则实现为 QuorumMaj
            2. 如果有group 配置 则实现则为 QuorumHierarchical 
            3. 对传入sid 集合 是否符合半数以上判断，半数的判断都是基于 votingMembers (可投票的）
                如果 QuorumHierarchical 为对权重的计算
            4. 对每个votingMembers对应的 QuorumServer electionAddr 投票端口 判断 （electionAddr 必须有)
            具体的说明见下面实现
            
      + QuorumMaj 实现 QuorumVerifier
        根据配置文件
        - allMembers -> myid, qs(上面QuorumServer) map <myid, QuorumServer>
        - votingMembers -> qs.getType == PARTICIPANT 可投票的群体
        - observingMembers -> qs.getType != PARTICIPANT 观察者的群体
        - half -> 半数  votingMembers.size() / 2  投票者半数
        - version -> 配置文件中的version 以 16进制 显示
        - containsQuorum(Set<Long> ackSet) 投票确认数是否超过半数
       
      + QuorumHierarchical 实现 QuorumVerifier 带层级的（也就是有group和weight权重的)
        根据配置文件 如果有 group 情况下
        - allMembers -> myid, qs(上面QuorumServer) map <myid, QuorumServer>
        - participatingMembers -> qs.getType == PARTICIPANT 可投票的群体
        - observingMembers -> qs.getType != PARTICIPANT 观察者的群体
        - numGroups 取决与 group.x 有多少个( 但最终以groupWeight sid 对应的 weight 值 大于 0 的个数为准).
        - serverGroup  key,value -> sid(myid), 组(group.x) 不能有重复的 sid 的定义 key
            allMembers qs.getType == PARTICIPANT  参与者的sid，
                    必须在serverGroup里(既myid都需要有group)
        - serverWeight key,value -> sid(weight.x) , value 
                allMembers qs.getType == PARTICIPANT  参与者的sid，
                   如果没有配置权重 value 则追加默认 权重值为 1
        - version -> 配置文件中的version 以 16进制 显示
        - groupWeight -> 权重算法: 有serverGroup 和 serverWeight 计算出来
                计算规则:  
                    serverGroup key 循环遍历 ->
                    既sid遍历 ->
                        if(groupWeight.get(serverGroup.get(sid) == null) {   
                                groupWeight.put(serverGroup.get(sid), serverWeight.get(sid));
                            } else {
                             groupWeight.put(serverGroup.get(sid), serverGroup.get(sid) +  serverWeight.get(sid))
                        }
					->
					因为组对应的sid 比较多 上面的 serverWeight.get(sid) 是叠加的总和.		   
        - containsQuorum(Set<Long> sids) 给定的sid集合是否 符合法定人数
            获取 sids 对应组的 权重（expansion Map<gid,value>)  算法和groupWeight 一致
            expansion 遍历gid 和 groupWeight.get(gid) 的value对比(既给定集合的组的权重值和配置项里的权重值)
            如果 expansion.get(gid) > groupWeight.get(gid)/2 则 majGroup 加 1
            majGroup 大于 numGroups / 2 则返回 真
            也就是 参数集合给定的权重一定要大于配置项里的权重的半数
        
   * DatadirCleanupManager 根据dataDir，dataLogDir 快照，事务日志文件路径 按照 snapRetainCount,PurgeInterval (快照保留数，清理间隔（单位小时))
      
      + 清理以上两个路径下的文件
        dataDir, dataLogDir 策略为 按 每隔PurgeInterval 小时 snapRetainCount 保留 最新的 数目.
        算法描述： 按 snapshot.21158f文件 从大到小排序， 获取 snapRetainCount 保留个数 既 前面几个 为一个集合 retainSnapFiles
                    然后再取 retainSnapFiles 最后一个 file 既 retainSnapFiles最小的一个 数字这一段 表示zxid,
                    删除所有 比zxid 小的 snapshot.xxx的文件, zxid 也是事务日志文件log.30a3 的数字对应 
                    如上 删除log.zxid 小于 zxid 所有log 文件
 
##ServerCnxnFactory zookeeper 核心##
           
###NIOServerCnxnFactory ServerCnxnFactory NIO 实现###
  * 典型的情况 在32 核机器上， 1 接收accept thread线程， 1 检查过期的线程， 4个selector ， 64个worker 线程
    1. 1个接受线程(accept thread)  接收新的链接请求 并分发到 selector 线程 (selector 主要是 应用了 NIO non-blocking 模式）
        但non blocking 模式和 linux network 下的 non-blocking 不一样 (linux 下的 non-blocking 是用户不断的发送 recvfrom 检测 是否准备好数据,
        不论数据是否准备好都会立刻返回)
        而上面 selector 对映的其实事 io mutil 模式 ，是用户调用selector/epoll 函数 selector 同时可以监视多个 socket 只要有一个socket 数据准备好后
        立即返回，并处理 kernel到用户内存的复制 然后返回给 用户。
    2. 1-N selector 线程, 每个 selector 线程处理较大 数量的 链接(connection) ， 现在的瓶颈在 平台的select函数 上
    3. 0-M 个 socket io worker线程，执行基本的 read，write。如果配置设置成0个 worker 线程，那么 selector 线程 直接做 io的 read，write。
    4. 1 链接(connection) 过期线程， 关闭闲置的链接。 其中没有建立session 的链接，是必须被终止的。
     
  * connect过期时间间隔
    sessionlessCnxnTimeOut "zookeeper.nio.sessionlessCnxnTimeout" 默认为 10 秒 
  
  * selector 线程数量(NIO Selector 数量)
    numSelectorThreads "zookeeper.nio.numSelectorThreads" 不能小于1 . selector thread 数量 默认情况下 机器core 处理器数量
                                coreNum/2 开平方后与1的最大值 -> 32/2 -> 16 sqr -> 4        max(4,1)  
  * worker 线程数量
    numWorkerThreads "zookeeper.nio.numWorkerThreads" 默认为 cores *　2 ->cores 32  * 2 -> 64
     
  * worker 关闭超时时间 (毫秒)
    workerShutdownTimeoutMS "ZOOKEEPER_NIO_SHUTDOWN_TIMEOUT" 默认为 5000

    
                
                
    
   
   
    
    
 