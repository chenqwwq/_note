# XXL - JOB



## 概述

XXL - JOB 属于分布式调度平台。

整个系统可以分为两个部分，Executor（执行器） 和 Scheduler（调度程序）。

调度程序就是统一的定时器，任务注册之后由调度程序进行统一的定时调度，由 Scheduler 指定 Executor 执行，并拉取执行的日志。









## Executor 

> 以 Spring 的客户端，MethodJobHandler 为例。



### 执行器的扫描和注册

SpringBoot 中通过 @XxlJob 指定执行器，包含执行器的名称，初始化方法以及销毁方法。

执行器通过 XxlJobSpringExecutor 扫描（该类继承了 SmartInitializingSingleton#afterSingletonsInstantiated。

该类的初始化方法里获取容器中的所有 Bean 对象，并扫描 Bean 中标注了 @XxlJob 的方法，**针对单个执行器方法包装并注册 IJobHandler**（例如对于方法的执行器就是 MtehodJobHandler，**另外就是要求必须声明为 Bean 对象，才能被扫描到**。

> 扫描所有类的所有方法是否效率过低？

MethodJobHandler 的执行就是**反射调用方法**。

![MethodJobHandler#executr](assets/image-20220117163552053.png)

（因此，调度的时候也无法传入任何参数，类似分片信息都需要通过另外的方式获取。



### 启动本地服务

**Xxl-Job 是通过本地 Http 服务端接收命令的执行请求的，**因此在客户端启动时还会开启一个 Http 服务（EmbedServer，默认绑定最大200个线程的线程池执行调度任务。

![image-20220117170837089](assets/image-20220117170837089.png)



在 EmbedHttpServerHandler 中响应 Scheduler 的任务调度。

> 额外说一句，Executor 似乎并不需要和很多个 Scheduler 连接，为什么采用 NIO 呢？
>
> NIO 在极少量连接的情况下性能也超过 BIO 吗？



### 本地任务响应

XXL-JOB 使用 Netty 建立的 Http 的服务端，所以请求处理也在 ChannelHandler 里，**XXL-JOB 客户端只响应 POST 请求，并根据请求的 URI 做调度。**

```java
// EmbedServer$EmbedHttpServerHandler#process
private Object process(HttpMethod httpMethod, String uri, String requestData, String accessTokenReq) {

  // valid
  // 只支持 POST 请求 
  if (HttpMethod.POST != httpMethod) {
    return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, HttpMethod not support.");
  }
  // xxl 会根据 URI 做任务调度
  if (uri==null || uri.trim().length()==0) {
    return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping empty.");
  }
  // 检查 accessToken
  if (accessToken!=null
      && accessToken.trim().length()>0
      && !accessToken.equals(accessTokenReq)) {
    return new ReturnT<String>(ReturnT.FAIL_CODE, "The access token is wrong.");
  }

  // services mapping
  try {
    // 心跳发送
    if ("/beat".equals(uri)) {
      return executorBiz.beat();
    // 检查是否空闲
    } else if ("/idleBeat".equals(uri)) {
      IdleBeatParam idleBeatParam = GsonTool.fromJson(requestData, IdleBeatParam.class);
      return executorBiz.idleBeat(idleBeatParam);
    // 任务调度
    } else if ("/run".equals(uri)) {
      TriggerParam triggerParam = GsonTool.fromJson(requestData, TriggerParam.class);
      return executorBiz.run(triggerParam);
    // 任务终止
    } else if ("/kill".equals(uri)) {
      KillParam killParam = GsonTool.fromJson(requestData, KillParam.class);
      return executorBiz.kill(killParam);
    // 日志拉取
    } else if ("/log".equals(uri)) {
      LogParam logParam = GsonTool.fromJson(requestData, LogParam.class);
      return executorBiz.log(logParam);
    } else {
      return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping("+ uri +") not found.");
    }
  } catch (Exception e) {
    logger.error(e.getMessage(), e);
    return new ReturnT<String>(ReturnT.FAIL_CODE, "request error:" + ThrowableUtil.toString(e));
  }
}
```

类似 Netty 的 ChannelHandler，XXL-JOB 也有自己的业务处理接口 ExecutorBiz，客户端的具体实现就是 ExecutorBizImpl。



![image-20220118140419963](assets/image-20220118140419963.png)

接口中包含了所有的业务逻辑方法，先来看执行方法 run()：

```java
@Override
public ReturnT<String> run(TriggerParam triggerParam) {
    // load old：jobHandler + jobThread
    // 获取执行线程，根据 jobId
    JobThread jobThread = XxlJobExecutor.loadJobThread(triggerParam.getJobId());
    // 获取线程绑定的执行器 JobHandler
    IJobHandler jobHandler = jobThread!=null?jobThread.getHandler():null;
    String removeOldReason = null;

    // valid：jobHandler + jobThread
    GlueTypeEnum glueTypeEnum = GlueTypeEnum.match(triggerParam.getGlueType());
    if (GlueTypeEnum.BEAN == glueTypeEnum) {

        // new jobhandler
        IJobHandler newJobHandler = XxlJobExecutor.loadJobHandler(triggerParam.getExecutorHandler());

        // valid old jobThread
        if (jobThread!=null && jobHandler != newJobHandler) {
            // change handler, need kill old thread
            removeOldReason = "change jobhandler or glue type, and terminate the old job thread.";

            jobThread = null;
            jobHandler = null;
        }

        // valid handler
        if (jobHandler == null) {
            jobHandler = newJobHandler;
            if (jobHandler == null) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "job handler [" + triggerParam.getExecutorHandler() + "] not found.");
            }
        }

    } else if (GlueTypeEnum.GLUE_GROOVY == glueTypeEnum) {

        // valid old jobThread
        if (jobThread != null &&
                !(jobThread.getHandler() instanceof GlueJobHandler
                    && ((GlueJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
            // change handler or gluesource updated, need kill old thread
            removeOldReason = "change job source or glue type, and terminate the old job thread.";

            jobThread = null;
            jobHandler = null;
        }

        // valid handler
        if (jobHandler == null) {
            try {
                IJobHandler originJobHandler = GlueFactory.getInstance().loadNewInstance(triggerParam.getGlueSource());
                jobHandler = new GlueJobHandler(originJobHandler, triggerParam.getGlueUpdatetime());
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
                return new ReturnT<String>(ReturnT.FAIL_CODE, e.getMessage());
            }
        }
    } else if (glueTypeEnum!=null && glueTypeEnum.isScript()) {

        // valid old jobThread
        if (jobThread != null &&
                !(jobThread.getHandler() instanceof ScriptJobHandler
                        && ((ScriptJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
            // change script or gluesource updated, need kill old thread
            removeOldReason = "change job source or glue type, and terminate the old job thread.";

            jobThread = null;
            jobHandler = null;
        }

        // valid handler
        if (jobHandler == null) {
            jobHandler = new ScriptJobHandler(triggerParam.getJobId(), triggerParam.getGlueUpdatetime(), triggerParam.getGlueSource(), GlueTypeEnum.match(triggerParam.getGlueType()));
        }
    } else {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "glueType[" + triggerParam.getGlueType() + "] is not valid.");
    }

    // executor block strategy
    // 执行阻塞策略
    if (jobThread != null) {
        ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(triggerParam.getExecutorBlockStrategy(), null);
        if (ExecutorBlockStrategyEnum.DISCARD_LATER == blockStrategy) {
            // discard when running
            if (jobThread.isRunningOrHasQueue()) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "block strategy effect："+ExecutorBlockStrategyEnum.DISCARD_LATER.getTitle());
            }
        } else if (ExecutorBlockStrategyEnum.COVER_EARLY == blockStrategy) {
            // kill running jobThread
            if (jobThread.isRunningOrHasQueue()) {
                removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();

                jobThread = null;
            }
        } else {
            // just queue trigger
        }
    }

    // replace thread (new or exists invalid)
    // 注册 Job 的处理线程，如果存在旧线程则先中断旧线程
    if (jobThread == null) {
        jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason);
    }

    // push data to queue
    ReturnT<String> pushResult = jobThread.pushTriggerQueue(triggerParam);
    return pushResult;
}
```
