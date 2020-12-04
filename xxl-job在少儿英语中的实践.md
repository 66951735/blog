## XXL-JOB在少儿英语中的实践

### 背景

少儿英语项目组任务调度平台初期采用了Spring的@Schedule注解结合Reids分布式锁的方式实现，通过模板方法（Template）模式抽象出一个任务的执行流程，编写任务时只需要实现具体的方法即可。这种实现方式简单直接，解决了当时服务多节点部署时重复执行任务的问题。随着业务的不断壮大，任务调度平台的任务越来越多，原有的任务调度暴露了一些问题。

- @Schedule默认是单线程模式，存在大任务执行一直占用资源导致其他任务饿死
- 任务调度平台缺少监控和报警机制，无法查看一个任务的具体执行情况
- 测试不方便，自测或者是QA测试需要修改代码来验证任务执行的正确性

基于以上问题，决定对原有的任务调度平台进行改造和升级，引入XXL-JOB分布式任务调度平台

### 任务调度的一些解决方案

#### 单机版

##### Timer（JDK1.3）

```
Timer timer = new Timer();
timer.scheduleAtFixedRate(new TimerTask() {
    @Override
    public void run() {
        System.out.println("this is a timer task example");
    }
}, 1000, 1000);
```

##### ScheduledExecutorService（JDK1.5）

```
ScheduledExecutorService executorService = Executors.newScheduledThreadPool(5);
executorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        System.out.println("this is scheduledExecutorService example");
    }
}, 1, 30, TimeUnit.SECONDS);
```

一般在实际应用过程中，推荐用Executor框架，而不是JDK1.3就已经存在的Timer，主要原因有两点：

- Timer是单线程的框架，只有一个线程在执行任务，而Executor是一个多线程的框架，初始化的时候可以指定线程的个数，并发执行提交的任务。
- 对于任务中未捕获的异常，Timer中线程直接终止，不能继续执行任务，而Executor只是取消当前的任务，还可以执行其他已提交的任务

##### @Scheduled（Spring）

在集成了SpringBoot的项目中，实现定时任务很简单，通过@EnableScheduling和@Scheduled两个注解就能够非常方便的编写一个单进程的定时任务。通过查看源码我们不难发现其底层实现还是依赖于JUC调度线程池ScheduledThreadPoolExecutor。需要注意的一点是，默认所有的@Scheduled方法由单线程调度，没有同时执行的任务，如果需要需要多线程调度，需要手动配置下。

```
public class ScheduleConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
    scheduledTaskRegistrar.setScheduler(Executors.newScheduledThreadPool(5));
    }
}
```

#### 集群部署

以下对比了Quartz、elastic-job以及xxl-job这三个支持集群部署的分布式任务调度框架

| 对比项         | Quartz                                           | ElasticJob                                                  | XXL-JOB（许雪里）                                       |
| :------------- | ------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------- |
| 项目背景       | OpenSymphony                                     | 当当网开源项目（Apache ShardingSphere）                     | 大众点评开源项目                                        |
| Github         | https://github.com/quartz-scheduler/quartz（4k） | https://github.com/apache/shardingsphere-elasticjob（6.5k） | https://github.com/xuxueli/xxl-job（15.9k）             |
| 可视化         | 无                                               | 支持通过Web页面对任务进行CRUD操作、基本用户权限管理等       | 支持通过Web页面对任务进行CRUD操作，支持报表、日志查看等 |
| 触发规则       | 时间触发                                         | 时间触发                                                    | 时间触发、事件触发（手动触发）                          |
| 依赖           | JDBC支持的关系型数据库（Mysql、Oracle等）        | Zookeeper                                                   | Mysql                                                   |
| 调度器集群部署 | 支持                                             | 支持                                                        | 支持                                                    |
| 执行器集群部署 | 支持                                             | 支持                                                        | 支持                                                    |
| 日志追溯       | 不支持                                           | 通过事件订阅的方式处理调度过程中重要的事件                  | 支持                                                    |
| 报警           | 不支持                                           | 可定制开发                                                  | 提供邮件报警支持，可扩展短信、钉钉等方式                |
| 弹性扩容       | 支持                                             | 支持                                                        | 支持                                                    |
| 依赖任务       | 不支持                                           | 不支持                                                      | 支持                                                    |
| 任务分片       | 不支持                                           | 支持                                                        | 支持                                                    |
| 多种作业类型   | 内置Java                                         | 支持Simple、Script等任务                                    | 通过GLUE支持Java以及Shell、Python的脚本任务             |

##### Quartz

Quartz作为开源作业调度中的佼佼者，是作业调度的首选，但是同样存在以下问题：

- 调用API的的方式操作任务，不人性化
- 需要持久化业务QuartzJobBean到底层数据表中，系统侵入性相当严重
- 调度逻辑和QuartzJobBean耦合在同一个项目中，这将导致一个问题，在调度任务数量逐渐增多，同时调度任务逻辑逐渐加重的情况下，此时调度系统的性能将大大受限于业务
- Quartz底层以“抢占式”获取DB锁并由抢占成功节点负责运行任务，会导致节点负载悬殊非常大

基于以上的问题，ElasticJob和XXL-JOB从两个不同的方向对Quartz进行了优化，下面具体说明下。

##### ElasticJob

ElasticJob是当当网的开源项目，2017年开始停止更新了一段时间，2020年成为了Apache ShardingSphere下的子项目，目前正在开发3.X的大版本。

ElasticJob基于Quartz演化而来，由于基于DB的方式缺少分布式协调功能，所以ElasticJob将DB换成成Zookeeper，基于Quarts调度组件和Curator实现了全局的作业控制中心，用于注册、控制和协调分布式作业执行。由于引入了Zookeeper，ElasticJob支持弹性扩容和数据分片的功能。

##### XXL-JOB

XXL-JOB是大众点评的开源项目，社区比较活跃，目前在持续更新中。

同样XXL-JOB早期也是基于Quartz的调度组建进行开发的，由于基于DB调度的一些问题，XXL-JOB最终选择自研调度组件，一方面是为了精简系统降低冗余依赖，另一方面是为了提供系统的可控度与稳定性。

XXL-JOB中“调度模块”和“任务模块”完全解耦，调度模块进行任务调度时，将会解析不同的任务参数发起远程调用，调用各自的远程执行器服务。这种调用模型类似RPC调用，调度中心提供调用代理的功能，而执行器提供远程服务的功能。

XXL-JOB虽然也是基于DB的方式实现任务调度，但是在XXL-JOB中调度和执行分离，XXL-JOB通过执行器实现“协同分配式”运行任务，充分发挥集群优势，负载各节点均衡，这里建议调度器和执行器的分别是独立的数据源，这样调度和执行完全隔离。

基于以上分析，少儿英语项目决定采用XXL-JOB对原有的任务调度平台进行升级和改造，这里选择XXL-JOB还有一个很重要的原因是，原来的任务调度平台直接可以作为执行器模块，只需要基于xxl-job-admin搭建一套调度中心即可，迁移成本较小，如果基于ElasticJob做改造，成本更高。

### XXL-JOB在少儿英语的实践

#### 架构图

![image-20201102174604569](https://tva1.sinaimg.cn/large/0081Kckwly1gkaz08jfraj31si0u01df.jpg)

#### 一次完成的任务调度流程

- “调度中心”向“执行器”发送http调度请求：“执行器”中接受请求的服务，实际上是一个内嵌的Server，默认端口号9999
- “执行器”执行任务逻辑
- “执行器”http回调“调度中心调度结果”

#### 基于Rancher平台的实际部署方案

<img src="https://tva1.sinaimg.cn/large/0081Kckwly1gkaz4nni5wj32ez0u0n35.jpg" alt="image-20201102175021401" style="zoom:50%;" />

#### 使用方式

现在开发一个定时任务主要分为两个步骤

- 在执行器（kids-job）中编写定时任务具体的业务代码
- 在调度中心（kids-job-ms）中配置定时任务的调度策略

##### 步骤一：编写定时任务

###### 说明

- 在执行器（kids-job）编写定时任务具体逻辑
- 一个定时任务以http接口的方式提供给调度中心（kids-job-ms）管理，简单来说一个定时任务就是controller中的一个方法
- 支持有参和无参两种调用方式

###### 示例

定时任务1（无参）

```
@RestController
@RequestMapping("/job/example")
public class JobExampleController {

  @PostMapping("/noArgsJob")
  public ResultData<String> noArgsJob() {
    boolean success = false;
    // TODO 具体定时任务逻辑
    // success = exampleJobService.execute();
    if (success) {
      return new ResultData<>(ResultCodeEnum.SUCCESS);
    }
    return new ResultData<>(ResultCodeEnum.PROCESS_ERROR, "错误描述");
  }
}
```

定时任务2（有参）

```
@RestController
@RequestMapping("/job/example")
public class JobExampleController {

  @PostMapping("/argsJob")
  public ResultData<String> noArgsJob(@RequestBody JobArgs jobArgs) {
    boolean success = false;
    // TODO 具体定时任务逻辑
    // success = exampleJobService.execute(jobArgs);
    if (success) {
      return new ResultData<>(ResultCodeEnum.SUCCESS);
    }
    return new ResultData<>(ResultCodeEnum.PROCESS_ERROR, "错误描述");
  }

  @Data
  public static class JobArgs {
    private String arg1;
    private Integer arg2;
  }
}
```

###### 一些约定

- requestType：POST
- path：/job/{模块名}/{任务名}
- 返回值：请使用ResultData作为返回值，retcode为10000，代表任务执行成功（这里对xxl-job的httpJobHandler做了下改造，只有业务处理成功才代表任务执行成功，而不是简单用http status 200做判断）
- 有参任务的参数格式为application/json，POST方式

##### 步骤二：调度中心设置调度策略

###### 新建无参定时任务

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkb03aqt6uj31cg0u0wib.jpg)

###### 新建有参定时任务

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkb044xwt7j31ck0u0ae0.jpg)

如图所示，新建任务时只需要填写标红框的部分，其中

- 任务描述：必填，请填写有意义的任务描述，方便搜索
- cron：必填
- 负责人：必填
- 报警邮件：选填，群组邮件，若失败需要报警邮件，请设置
- 任务参数：必填
  - json格式，{"path":"/job/example/testJob","param":{}}
  - 其中path为job的http路径，param为job的参数，param选填
  - 目前job对应的http接口仅支持POST请求，json入参方式

新建任务完成后，在操作里点击"启动"按钮，任务即可按照cron规则来执行了

关于调度中心系统的其他操作，可以参考https://www.xuxueli.com/xxl-job/

#### 后续的工作

- 支持分片，目前的部署方式是手动注册的执行器，一个执行器地址对应的是多pod实例的clusterIP地址，通过k8s的内部域名服务以及kube-proxy来进行pod的选择，所以对于调度中心来说，执行器只有一个，无法支持分片。后续调整部署方式，每个pod自动向调度中心注册成为一个执行器来支持分片。
- 接入有道LDAP服务，原生支持rd登录
- 接入泡泡报警，目前只支持邮箱报警

### XXL-JOB核心源码分析

注：以下源码分析基于v2.2.0版本

#### 调度中心

调度中心负责调度的核心类为JobScheduleHelper，里边的scheduleThread线程，通过while循环不断地从数据库中查找符合执行条件的任务（单线程），然后通过JobTriggerPoolHelper中的线程池并发地将任务发送给执行器去执行（多线程）。

##### 分布式调度的一致性

调度中心通过DB锁保证集群分布式调度的一致性，一次任务调度只会触发一次执行

```
while (!scheduleThreadToStop) {
    Connection conn = null;
    Boolean connAutoCommit = null;
    PreparedStatement preparedStatement = null;

    boolean preReadSuc = true;
    try {

        conn = XxlJobAdminConfig.getAdminConfig().getDataSource().getConnection();
        connAutoCommit = conn.getAutoCommit();
        // 自动提交设置为false
        conn.setAutoCommit(false);
        // 利用for update加锁
        preparedStatement = conn.prepareStatement("select * from xxl_job_lock where lock_name = 'schedule_lock' for update");
        preparedStatement.execute();
        
        // 查询可执行的任务，交给执行器去执行
        .....

    } catch (Exception e) {
        if (!scheduleThreadToStop) {
            logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#scheduleThread error:{}", e);
        }
    } finally {

        // commit
        if (conn != null) {
            try {
                // finally提交事物
                conn.commit();
            } catch (SQLException e) {
                if (!scheduleThreadToStop) {
                    logger.error(e.getMessage(), e);
                }
            }
            try {
                // 撤销之前的操作
                conn.setAutoCommit(connAutoCommit);
            } catch (SQLException e) {
                if (!scheduleThreadToStop) {
                    logger.error(e.getMessage(), e);
                }
            }
            try {
                conn.close();
            } catch (SQLException e) {
                if (!scheduleThreadToStop) {
                    logger.error(e.getMessage(), e);
                }
            }
        }

        // close PreparedStatement
        ......
    }
```

##### 线程池并发调度

调度采用线程池方式实现，避免单线程因阻塞而引起任务调度延迟。同时调度线程池进行隔离拆分，慢任务自动降级进入”Slow”线程池，避免耗尽调度线程，提高系统稳定性。

```
public class JobTriggerPoolHelper {

    // ---------------------- trigger pool ----------------------

    // fast/slow thread pool，做了基本的线程池隔离
    private ThreadPoolExecutor fastTriggerPool = null;
    private ThreadPoolExecutor slowTriggerPool = null;

    public void start(){
        // fast线程池，maxSize可配置
        fastTriggerPool = new ThreadPoolExecutor(
                10,
                XxlJobAdminConfig.getAdminConfig().getTriggerPoolFastMax(),
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(1000),
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r, "xxl-job, admin JobTriggerPoolHelper-fastTriggerPool-" + r.hashCode());
                    }
                });

				// fast线程池，maxSize可配置
        slowTriggerPool = new ThreadPoolExecutor(
                10,
                XxlJobAdminConfig.getAdminConfig().getTriggerPoolSlowMax(),
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(2000),
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r, "xxl-job, admin JobTriggerPoolHelper-slowTriggerPool-" + r.hashCode());
                    }
                });
    }

		......

    // job timeout count
    private volatile long minTim = System.currentTimeMillis()/60000;     // ms > min
    private volatile ConcurrentMap<Integer, AtomicInteger> jobTimeoutCountMap = new ConcurrentHashMap<>();

    /**
     * add trigger，添加一个调度的触发器
     */
    public void addTrigger(final int jobId,
                           final TriggerTypeEnum triggerType,
                           final int failRetryCount,
                           final String executorShardingParam,
                           final String executorParam,
                           final String addressList) {

        // choose thread pool
        // 默认选择fast pool
        ThreadPoolExecutor triggerPool_ = fastTriggerPool;
        // 从全局的jobTimeoutCountMap拿到1分钟超时的数量
        AtomicInteger jobTimeoutCount = jobTimeoutCountMap.get(jobId);
        // 如果大于10，切换为slow pool
        if (jobTimeoutCount!=null && jobTimeoutCount.get() > 10) {      // job-timeout 10 times in 1 min
            triggerPool_ = slowTriggerPool;
        }

        // 具体的触发调度逻辑，通过xxl-rpc将任务信息传递个执行器集群
        triggerPool_.execute(new Runnable() {
            @Override
            public void run() {

                long start = System.currentTimeMillis();

                try {
                    // do trigger
                    XxlJobTrigger.trigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam, addressList);
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                } finally {

                    // check timeout-count-map
                    long minTim_now = System.currentTimeMillis()/60000;
                    if (minTim != minTim_now) {
                        minTim = minTim_now;
                        jobTimeoutCountMap.clear();
                    }

                    // incr timeout-count-map
                    long cost = System.currentTimeMillis()-start;
                    if (cost > 500) {       // ob-timeout threshold 500ms
                        AtomicInteger timeoutCount = jobTimeoutCountMap.putIfAbsent(jobId, new AtomicInteger(1));
                        if (timeoutCount != null) {
                            timeoutCount.incrementAndGet();
                        }
                    }

                }

            }
        });
    }

    ......

}
```

#### 执行器

执行器负责接收调度请求并执行任务逻辑。

##### 自动注册

```
public class ExecutorRegistryThread {

    // 单例
    private static ExecutorRegistryThread instance = new ExecutorRegistryThread();
    public static ExecutorRegistryThread getInstance(){
        return instance;
    }

    private Thread registryThread;
    private volatile boolean toStop = false;
    public void start(final String appname, final String address){

        // 一些参数有效性校验
        ......

        registryThread = new Thread(new Runnable() {
            @Override
            public void run() {

                // registry
                while (!toStop) {
                    try {
                        RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
                        // 向配置的每一个调度中心地址发送注册请求
                        for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
                            try {
                                ReturnT<String> registryResult = adminBiz.registry(registryParam);
                                // 注册结果的处理
                                ......
                            } catch (Exception e) {
                                logger.info(">>>>>>>>>>> xxl-job registry error, registryParam:{}", registryParam, e);
                            }

                        }
                    } catch (Exception e) {
                        if (!toStop) {
                            logger.error(e.getMessage(), e);
                        }

                    }

                    try {
                        if (!toStop) {
                            // 周期性注册的心跳间隔，默认30s
                            TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
                        }
                    } catch (InterruptedException e) {
                        if (!toStop) {
                            logger.warn(">>>>>>>>>>> xxl-job, executor registry thread interrupted, error msg:{}", e.getMessage());
                        }
                    }
                }

                // registry remove
                ......
            }
        });
        registryThread.setDaemon(true);
        registryThread.setName("xxl-job, executor ExecutorRegistryThread");
        registryThread.start();
    }

    ......

}
```

##### GLUE模式任务

XXL-JOB支持GLUE模式的任务，可以在调度中心在线编写任务逻辑代码，动态发布，实时编译生效。

主要分为两类：Script语言任务和JAVA语言任务

###### Script

```
// 通过Runtime执行脚本语言命令
final Process process = Runtime.getRuntime().exec(cmdarrayFinal);
```

###### Java

```
private Class<?> getCodeSourceClass(String codeSource){
   try {
      // md5
      byte[] md5 = MessageDigest.getInstance("MD5").digest(codeSource.getBytes());
      String md5Str = new BigInteger(1, md5).toString(16);

      Class<?> clazz = CLASS_CACHE.get(md5Str);
      if(clazz == null){
         // 通过GroovyClassLoader动态加载Java类
         clazz = groovyClassLoader.parseClass(codeSource);
         CLASS_CACHE.putIfAbsent(md5Str, clazz);
      }
      return clazz;
   } catch (Exception e) {
      return groovyClassLoader.parseClass(codeSource);
   }
}
```

### 参考

- https://www.xuxueli.com/xxl-job/