## JedisCluster源码分析

本文基于jedis 3.2.0版本

#### redis cluster测试环境搭建

- 参照[docker-redis-cluster](https://github.com/Grokzen/docker-redis-cluster)，基于docker-compose构建redis cluster环境

- 服务启动后，通过redis-cli连接到集群服务

  ```
  ./redis-cli -h 192.168.0.107 -p 7001 -c
  ```

- 执行cluster nodes命令，查看集群状态如下，三主三从，slot分配均匀，集群状态正常。

  ![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf3t0w0cajj32c80bck33.jpg)

#### 源码分析

##### UML类图

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf3tey361xj33920o4gn9.jpg)

JedisCluster源码比较简单，入口类为`JedisCluster`，主要流程分为两个：

- `JedisCluster`初始化，即如何构建`slot`与`jedisPool`的对应关系，并缓存到客户端本地，实现**smart client**。
- `redis`命令的执行过程，即如何从对应的`jedisPool`中拿到一个`jedis`连接，并执行`redis`命令。

##### `JedisCluster`初始化

首先来看`JedisCluster`，这个类提供了各种不同参数的构造方法供调用，核心参数如下：

- `Set<HostAndPort> nodes`：一组cluster节点的集合
- `timeout`：超时时间，可以细分为`connectionTimeout`和`soTimeout`，默认时间为200ms
- `maxAttempts`：最大尝试次数，默认值5，redis命令执行时的最大尝试次数
- `GenericObjectPoolConfig poolConfig`：jedis连接池配置

继续跟踪代码，发现`JedisCluster`最终调用的是父类`BinaryJedisCluster`构造方法，代码如下：

```
public BinaryJedisCluster(Set<HostAndPort> jedisClusterNode, int timeout, int maxAttempts,
      final GenericObjectPoolConfig poolConfig) {
    this.connectionHandler = new JedisSlotBasedConnectionHandler(jedisClusterNode, poolConfig,
        timeout);
    this.maxAttempts = maxAttempts;
}
```

可以看到`BinaryJedisCluster`的构造方法主要干了两件事情：

- 初始化`JedisSlotBasedConnectionHandler`
- 初始化`maxAttempts`

我们继续看`JedisSlotBasedConnectionHandler`，这个handler初始化时会调用父类`JedisClusterConnectionHandler`构造方法，代码如下：

```
public JedisClusterConnectionHandler(Set<HostAndPort> nodes,
      final GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout, String password, String clientName, boolean ssl, SSLSocketFactory sslSocketFactory, SSLParameters sslParameters,
      HostnameVerifier hostnameVerifier, JedisClusterHostAndPortMap portMap) {
    this.cache = new JedisClusterInfoCache(poolConfig, connectionTimeout, soTimeout, password, clientName,
        ssl, sslSocketFactory, sslParameters, hostnameVerifier, portMap);
    initializeSlotsCache(nodes, connectionTimeout, soTimeout, password, clientName, ssl, sslSocketFactory, sslParameters, hostnameVerifier);
}
```

看到这里，我们终于找到`JedisCluster`初始化，构建slot与jedisPool的对应关系的核心逻辑了，这个逻辑是由`JedisClusterInfoCache`类来完成的，我们首先来看下`initializeSlotsCache`方法：

```
private void initializeSlotsCache(Set<HostAndPort> startNodes,
      int connectionTimeout, int soTimeout, String password, String clientName,
      boolean ssl, SSLSocketFactory sslSocketFactory, SSLParameters sslParameters, HostnameVerifier hostnameVerifier) {
    // 遍历cluster nodes的集合
    for (HostAndPort hostAndPort : startNodes) {
      Jedis jedis = null;
      try {
        // 构建这个node的jedis连接
        jedis = new Jedis(hostAndPort.getHost(), hostAndPort.getPort(), connectionTimeout, soTimeout, ssl, sslSocketFactory, sslParameters, hostnameVerifier);
        // 认证相关代码
        if (password != null) {
          jedis.auth(password);
        }
        if (clientName != null) {
          jedis.clientSetname(clientName);
        }
        // 调用JedisClusterInfoCache的discoverClusterNodesAndSlots方法，构建slot与jedisPool的对应关系，并缓存
        cache.discoverClusterNodesAndSlots(jedis);
        // 如果成功，则对应关系已经缓存成功，没必要继续执行接下来的循环，跳出循环
        break;
      } catch (JedisConnectionException e) {
        // try next nodes
      } finally {
        if (jedis != null) {
          jedis.close();
        }
      }
    }
  }
```

接下来我们来看`JedisClusterInfoCache`这个核心类

- 两个核心字段

  - `Map<String, JedisPool> nodes`，node与JedisPool的缓存map
  - `Map<Integer, JedisPool> slots`，slot与JedisPool的缓存map

- 核心方法`discoverClusterNodesAndSlots`

  ```
  public void discoverClusterNodesAndSlots(Jedis jedis) {
      // 加写锁
      w.lock();
  
      try {
        // Clear discovered nodes collections and gently release allocated resources
        reset();
        // 执行cluster slots命令，得到slot与主从节点的对应关系
        // 关于cluster slots命令可以参考https://redis.io/commands/cluster-slots
        // 在本文开始前的构建的测试集群上执行cluster slots命令，结果如下图，可以看下返回值的格式
        List<Object> slots = jedis.clusterSlots();
  
        // 遍历返回的slot信息
        for (Object slotInfoObj : slots) {
          List<Object> slotInfo = (List<Object>) slotInfoObj;
  	      
  	      // 正常情况下master节点的index为MASTER_NODE_INDEX
  	      // 如果slotInfo的size小于index，则返回格式由问题，跳过当前循环
          if (slotInfo.size() <= MASTER_NODE_INDEX) {
            continue;
          }
  
          // 构建当前slot从start到end的整型list
          List<Integer> slotNums = getAssignedSlotArray(slotInfo);
  
          // hostInfos
          int size = slotInfo.size();
          // 遍历master以及接下来的slave节点，构建nodes和slots缓存
          for (int i = MASTER_NODE_INDEX; i < size; i++) {
            List<Object> hostInfos = (List<Object>) slotInfo.get(i);
            if (hostInfos.size() <= 0) {
              continue;
            }
  					
            HostAndPort targetNode = generateHostAndPort(hostInfos);
            // 构建nodes缓存，即当前node与jedisPool的对应关系
            setupNodeIfNotExist(targetNode);
            // 如果是master节点，还需要构建slots缓存，即slotNums与当前节点jedisPool的对应关系
            if (i == MASTER_NODE_INDEX) {
              assignSlotsToNode(slotNums, targetNode);
            }
            // 至此，JedisCluster核心逻辑初始化完成
          }
        }
      } finally {
        w.unlock();
      }
    }
  ```

- `cluster slots`执行结果

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gf3vecv49gj30wg0u0gua.jpg" style="zoom:50%;" />



##### 如何执行redis命令

首先还是从`JedisCluster`类开始，以`set`命令为例，分析命令的具体执行过程，代码如下：

```
@Override
public String set(final String key, final String value, final SetParams params) {
    return new JedisClusterCommand<String>(connectionHandler, maxAttempts) {
      @Override
      public String execute(Jedis connection) {
        return connection.set(key, value, params);
      }
    }.run(key);
}
```

可以看到，`JedisCluster`类中每一个命令的调用过程都是这样一个逻辑

- new出 `JedisClusterCommand`这个抽象类的匿名类
- 实现`JedisClusterCommand`这个抽象类的`execute`方法，很明显这是一个模版方法，每个命令执行的时候都需要调用自己的`jedis`命令
- `run`方法的主要逻辑就是根据`key`找到对应`slot`的`jedisPool`，然后从这个`pool`中拿到一个`jedis`连接，执行`execute`方法，完成命令的执行并返回结果

下面我们具体看下`JedisClusterCommand`这个类的`run`方法：

```
public T run(String key) {
    // 这里可以看到根据key计算slot的过程，是基于CRC16来做的
    return runWithRetries(JedisClusterCRC16.getSlot(key), this.maxAttempts, false, null);
}
```

下边具体来看下`runWithRetries` 方法

```
/**
   * @param slot 根据key计算出来的slot值
   * @param attempts 最大尝试次数
   * @param tryRandomNode 是否随机选择节点，初次调用为false
   * @param redirect 有无JedisRedirectionException异常，初次调用为null
   * @return
   */
private T runWithRetries(final int slot, int attempts, boolean tryRandomNode, JedisRedirectionException redirect) {
    // 达到最大尝试次数，抛出异常
    if (attempts <= 0) {
      throw new JedisClusterMaxAttemptsException("No more cluster attempts left.");
    }

    Jedis connection = null;
    try {
      
      // 首次获取jedis连接失败时，会递归调用runWithRetries进行重试，重试的时候会携带上JedisRedirectionException异常
      if (redirect != null) {
        // 如果存在move或者ask异常，则从nodes缓存中获取对应的jedisPool，然后拿到jedis连接
        connection = this.connectionHandler.getConnectionFromNode(redirect.getTargetNode());
        if (redirect instanceof JedisAskDataException) {
          // TODO: Pipeline asking with the original command to make it faster....
          connection.asking();
        }
      } else {
        if (tryRandomNode) {
          // 从nodes缓存随机获取一个节点的jedisPool，然后拿到jedis连接
          connection = connectionHandler.getConnection();
        } else {
          // 通过slot从slots缓存中获取对应的jedisPool，然后拿到jedis连接
          connection = connectionHandler.getConnectionFromSlot(slot);
        }
      }
      // 调用对应命令的模版方法execute，返回命令的执行结果
      return execute(connection);

    } catch (JedisNoReachableClusterNodeException jnrcne) {
      throw jnrcne;
    } catch (JedisConnectionException jce) {
      // release current connection before recursion
      releaseConnection(connection);
      connection = null;

      if (attempts <= 1) {
        //We need this because if node is not reachable anymore - we need to finally initiate slots
        //renewing, or we can stuck with cluster state without one node in opposite case.
        //But now if maxAttempts = [1 or 2] we will do it too often.
        //TODO make tracking of successful/unsuccessful operations for node - do renewing only
        //if there were no successful responses from this node last few seconds
        this.connectionHandler.renewSlotCache();
      }

      return runWithRetries(slot, attempts - 1, tryRandomNode, redirect);
    } catch (JedisRedirectionException jre) {
      // if MOVED redirection occurred,
      if (jre instanceof JedisMovedDataException) {
        // it rebuilds cluster's slot cache recommended by Redis cluster specification
        this.connectionHandler.renewSlotCache(connection);
      }

      // release current connection before recursion
      releaseConnection(connection);
      connection = null;

      return runWithRetries(slot, attempts - 1, false, jre);
    } finally {
      releaseConnection(connection);
}
```

下面重点看下`connectionHandler`以下两个方法

- `getConnectionFromSlot`

```
@Override
public Jedis getConnectionFromSlot(int slot) {
    // 从slots缓存中获取对一个的jedisPool
    JedisPool connectionPool = cache.getSlotPool(slot);
    if (connectionPool != null) {
      // It can't guaranteed to get valid connection because of node
      // assignment
      return connectionPool.getResource();
    } else {
      // 重新构建slots缓存
      renewSlotCache(); //It's abnormal situation for cluster mode, that we have just nothing for slot, try to rediscover state
      connectionPool = cache.getSlotPool(slot);
      if (connectionPool != null) {
        return connectionPool.getResource();
      } else {
        //no choice, fallback to new connection to random node
        return getConnection();
      }
    }
}
```

- `renewClusterSlots`

```
// 异常情况下（譬如slot重新分配，或者集群网络故障），如果重新构建slot缓存
public void renewClusterSlots(Jedis jedis) {
    //If rediscovering is already in process - no need to start one more same rediscovering, just return
    // 这里rediscovering是一个volatile的标志位，如果正在重新构建slot缓存的话，就直接返回，避免重复操作
    if (!rediscovering) {
      try {
        w.lock();
        if (!rediscovering) {
          rediscovering = true;

          try {
            // 如果传入的jedis连接不为null的话，根据这个jedis连接来重新构建slot缓存
            if (jedis != null) {
              try {
                discoverClusterSlots(jedis);
                return;
              } catch (JedisException e) {
                //try nodes from all pools
              }
            }
            
            // 如果传入的jedis连接为null的话，从nodes缓存里，随机选择一个jedisPool，拿到一个jedis连接
            for (JedisPool jp : getShuffledNodesPool()) {
              Jedis j = null;
              try {
                j = jp.getResource();
                discoverClusterSlots(j);
                return;
              } catch (JedisConnectionException e) {
                // try next nodes
              } finally {
                if (j != null) {
                  j.close();
                }
              }
            }
          } finally {
            rediscovering = false;      
          }
        }
      } finally {
        w.unlock();
      }
    }
}
```

至此，**初始化JedisCluster**以及**执行redis的命令**两个核心流程的源码分析完毕。