#zookeeper启动时加载snap,log文件#

    ##加载snap文件##
        * snap文件 zookeeper的快照根据config中的snapCount数字生成定期的快照
            + snap 文件配置在config中的dataDir路径下
            + 获取dataDir 路径下最大并且有效snapshot.xx的文件 xx表示最后的事务号zxid
            + 根据Adler32 检验snapshot文件.
            + 反序列化 snapshot文件
              1. fileHeader 查看snapshot二进制文件
                 反序列化 FileHeader 文件"第一行"
              		magic,version,bdid -> 默认为ZKSN,2,-1
              		magic 魔术用来判断文件是否是zookeeper snap文件 如果不是"ZKSN" 抛出异常
              2. sessionWithTimeout session,timeout -> id,timeout  
                 文件第二行count, [id, timeout](count个)
                 count 为Map中key,value个数。 id,timeout 有count个.
              3. dt : DataTree 反序列化 
                 1. Map<Long, List<ACL>> longKeyMap -> aclsMap 访问权限
                     map,long,[acls,[perms,scheme,id]]
                        map acls的个数
                        long aclsMap的key
                        acls key的应Acl集合的个数
                        [perms,[scheme,id]] 为 value 既 Acl perms, Id->scheme,id
                 2. ConcurrentHashMap<String, DataNode> nodes   路径和节点数据包装类 -> path, DataNode 
                    读取snapshot二进制文件获取path对应的4个字节以及它的真实路径；
                    根据path后面对应的数据继续反序列化DataNode node
                    完成一个path以及相对应的数据后继续"遍历"path直到所有的path以及节点数据被load为止
                    node = new DataNode();
                        data->读取4个字节 表示 data数据的长度, 后面长度为 实际的data
                        acl -> 8 字节
                        StatPersisted 统计
                            czxid -> createNode时事务号
                            mzxid -> createNode时与Czxid同,setData时的事务号
                            ctime -> 创建节点时间
                            mtime -> createNode时与ctime相同，setData时间
                            version -> createNode版本为0，setData的数据版本号
                            cversion -> createNode版本为0，增加/删除子节点时父节点+1
                            aversion -> createNode版本为0，setACL时节点的版本号
                            ephemeralOwner -> 临时节点表示sessionid,非临时节点这个值为0
                            pzxid -> createNode时与Czxid同,增加/删除子节点时为子节点事务号
                    //nodes里添加 path，node
                    nodes.put(path, node);
                    path 路径都是现有短路径后又长路径-> 现有父路径后又子路径
                    获取路径的父路径，并把node添加到父节点node中(parentNode.addChild(node))
                        int index = path.lastIndexOf("/");
                        if(index == -1) {
                            root = node;
                        } else {
                            String parentPath = path.subString(0, index); 获取路径的父路径
                            DataNode parentNode = nodes.get(parentPath);
                            parentNode.addChild();
                            long eowner = node.stat.getEphemeralOwner();
                            if (eowner == CONTAINER_EPHEMERAL_OWNER) {
                                containers.add(path);
                            } else if (eowner != 0) {
                                HashSet<String> list = ephemerals.get(eowner);
                              if (list == null) {
                                    list = new HashSet<String>();
                                    ephemerals.put(eowner, list);
                              }
                              list.add(path);
                          }
                        }
                 3. HashSet containers 容器临时节点集合  根据node.stat.getEphemeralOwner() == CONTAINER_EPHEMERAL_OWNER
                 4. Map<Long, HashSet<String>> ephemerals 临时节点集合 根据node.stat.getEphemeralOwner() == -1
                 5. pTrie
                 6. lastProcessedZxid 为snapshot.xx 中的xx
    
    ##加载txnLog文件(事务文件)##
    
        *
                                            
                           