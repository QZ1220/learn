﻿# Spring源码阅读

标签（空格分隔）： spring 源码

* [Spring源码阅读](#spring源码阅读)
   * [相关版本](#相关版本)
   * [eureka源码分析](#eureka源码分析)
      * [注册](#注册)
      * [对外提供服务](#对外提供服务)
      * [下线](#下线)
      * [源码分析](#源码分析)
         * [客户端发起注册与服务获取](#客户端发起注册与服务获取)
         * [服务端接收注册](#服务端接收注册)
         * [多个eureka server信息同步](#多个eureka-server信息同步)
   * [Ribbon源码分析](#ribbon源码分析)
      * [@LoadBalanced注解](#loadbalanced注解)
      * [Ribbon的负载均衡策略](#ribbon的负载均衡策略)
         * [轮询](#轮询)
         * [重试](#重试)
      * [重试机制](#重试机制)
      * [基于Ribbon实现服务灰度发布](#基于ribbon实现服务灰度发布)
   * [Hystrix源码分析](#hystrix源码分析)
      * [命令模式](#命令模式)
      * [源码解析](#源码解析)
         * [command执行过程](#command执行过程)
         * [断路器HystrixCircuitBreaker](#断路器hystrixcircuitbreaker)
      * [Hystrix线程池隔离](#hystrix线程池隔离)
      * [请求合并](#请求合并)
      * [Hystrix状态转换](#hystrix状态转换)
      * [Hystrix健康度计算规则](#hystrix健康度计算规则)
   * [feign](#feign)
      * [重试机制](#重试机制-1)
      * [服务降级](#服务降级)
      * [请求压缩](#请求压缩)
      * [因为重试导致的一个bug](#因为重试导致的一个bug)
      * [如何关闭重试及源码调试](#如何关闭重试及源码调试)
   * [spring cloud config](#spring-cloud-config)

本文主要针对springBoot、springCloud的相关组件的源码阅读及个人理解，参考了很多《Spring Cloud微服务实战》一书的源码分析。

## 相关版本

- springboot  2.3.3.RELEASE
- springcloud Hoxton.SR10


## eureka源码分析

### 注册
 - 各个服务组件是如何将自己注册到eureka上的呢？

简单来说就是向eureka发送REST请求将自己注册到Eureka Server上的。

 - 那么，这些注册信息又是如何在Eureka上保存的呢？

Eureka内部通过，使用双层的Map结构来保存服务的注册信息。第一层key是服务名，通过spring.application.name配置，第二层key是具体的实例名称（比如，一个服务可以启动多个实例），通过spring.instance.hostname设置。

 - 服务注册到eureka上以后，如何告诉eureka自己还”活着“呢？避免eureka将自己剔除呢？

服务与eureka会通过心跳机制，来告诉eureka，自己还”活着“。这也叫”服务续约“，关于续约，有下面两个重要的参数：

```java
spring.instance.lease-renewal-interval-in-seconds=30
spring.instance.lease-expiration-duration-in-seconds=90
```

前者定义了服务多久和eureka通信一次，后者告诉eureka多久将无心跳的服务踢下线。上面的30和90秒都是默认时间，也就是说，不手动配置，这两个参数也是生效的，只不过取默认值。

### 对外提供服务
 - 服务消费者如何获取eureka上注册的服务提供者呢？

消费者会发送REST请求给eureka，以获取eureka上注册的服务清单（可能不仅仅只包含服务提供者）。出于性能考虑，eureka会维护（缓存）一份只读的服务清单来返回给客户端。该缓存清单每隔30s会更新一次。如果希望修改这个时间，可以使用如下的配置：
 
```java
 eureka.client.registry-fetch-interval-seconds=15
```
 
消费者获取到服务清单以后（其中包含服务提供者的元数据信息），可以根据自己的实际需要进行调用，Ribbon默认采用轮训的方式进行调用，从而实现负载均衡。
 
 
### 下线

 - 服务下线

正常的服务下线时，会给eureka发送下线的Rest请求，告诉eureka将自己从服务列表剔除。

但是，如果服务异常关闭了，是不会往eureka发送下线请求的，eureka会每60秒检测一次是否有超时时间超过90秒的无心跳服务，如果有就将其从服务列表清除。


### 源码分析

一般配置一个eureka服务端，我们会做两件事情：

 1. 在Application主类中配置EnableEurekaServer，（@EnableDiscoveryClient/@EnableEurekaClient是客户端是使用的注解）注解
 2. 在application.properties中配置defaultZone

注意：@EnableDiscoveryClient/@EnableEurekaClient功能类似，区别在于@EnableDiscoveryClient适用于包含eureka在内的多个注册中心，但是@EnableEurekaClient只适用于eureka。

#### 客户端发起注册与服务获取

@EnableDiscoveryClient主要用来开启DiscoveryClient实例，通过梳理，可以得到如下所示的依赖关系图：

![eurekaClient](./image/spring/eurekaClient.png)

真正实现发现服务的是`com.netflix.discovery.DiscoveryClient`类，该类声明及注释如下：
```java
/**
 * The class that is instrumental for interactions with <tt>Eureka Server</tt>.
 *
 * <p>
 * <tt>Eureka Client</tt> is responsible for a) <em>Registering</em> the
 * instance with <tt>Eureka Server</tt> b) <em>Renewal</em>of the lease with
 * <tt>Eureka Server</tt> c) <em>Cancellation</em> of the lease from
 * <tt>Eureka Server</tt> during shutdown
 * <p>
 * d) <em>Querying</em> the list of services/instances registered with
 * <tt>Eureka Server</tt>
 * <p>
 *
 * <p>
 * <tt>Eureka Client</tt> needs a configured list of <tt>Eureka Server</tt>
 * {@link java.net.URL}s to talk to.These {@link java.net.URL}s are typically amazon elastic eips
 * which do not change. All of the functions defined above fail-over to other
 * {@link java.net.URL}s specified in the list in the case of failure.
 * </p>
 *
 * @author Karthik Ranganathan, Greg Kim
 * @author Spencer Gibb
 *
 */
@Singleton
public class DiscoveryClient implements EurekaClient {
```

**服务注册源码分析**

在com.netflix.discovery.DiscoveryClient类中可以找到一个方法initScheduledTasks()，这个方法在DiscoveryClient的构造函数中，会被调用，他主要是负责启动服务列表获取、服务注册的定时任务，源码如下：
```java
/**
     * Initializes all scheduled tasks.
     */
    private void initScheduledTasks() {
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "heartbeat",
                            scheduler,
                            heartbeatExecutor,
                            renewalIntervalInSecs,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new HeartbeatThread()
                    ),
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // InstanceInfo replicator
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                            InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                        // log at warn level if DOWN was involved
                        logger.warn("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }

            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
```
上面的`if (clientConfig.shouldFetchRegistry())`就是判断是否需要启动服务列表获取的定时任务，`if (clientConfig.shouldRegisterWithEureka())`就是判断是否需要将服务注册到eureka上去。定时任务的执行周期都可以配置文件进行配置，否则取默认值。

 
 在`if (clientConfig.shouldRegisterWithEureka())`分支内创建了一个InstanceInfoReplicator对象，这个对象实现了Runnable接口，通过查看其run方法：
```java
 public void run() {
        try {
            discoveryClient.refreshInstanceInfo();

            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
```
 它内部的register方法就是做注册使用的，同时com.netflix.appinfo.InstanceInfo保存了注册服务的元信息。
```java
 /**
     * Register with the eureka service by making the appropriate REST call.
     */
    boolean register() throws Throwable {
        logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
```
 
 **服务获取源码分析**
 
 在com.netflix.discovery.DiscoveryClient类内部的initScheduledTasks方法内部还有一个if判断主要是启动服务获取的定时任务的，源码如下：
```java
 if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }
```
查看`TimedSupervisorTask`类的`run`方法可以知道，它内部会去定时发起请求，源码如下：
```java
    @Override
    public void run() {
        Future<?> future = null;
        try {
        // 将任务提交到线程池
            future = executor.submit(task);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
            // 阻塞获取执行结果，直至超时
            future.get(timeoutMillis, TimeUnit.MILLISECONDS);  // block until done or timeout
            delay.set(timeoutMillis);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
            successCounter.increment();
        } catch (TimeoutException e) {
            logger.warn("task supervisor timed out", e);
            timeoutCounter.increment();

            long currentDelay = delay.get();
            // 如果访问超时，那么下次executor的执行时间间隔就翻倍
            long newDelay = Math.min(maxDelay, currentDelay * 2);
            delay.compareAndSet(currentDelay, newDelay);

        } catch (RejectedExecutionException e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, reject the task", e);
            } else {
                logger.warn("task supervisor rejected the task", e);
            }

            rejectedCounter.increment();
        } catch (Throwable e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, can't accept the task");
            } else {
                logger.warn("task supervisor threw an exception", e);
            }

            throwableCounter.increment();
        } finally {
            if (future != null) {
                future.cancel(true);
            }

            if (!scheduler.isShutdown()) {
            // 设置下一次调度的执行信息
                scheduler.schedule(this, delay.get(), TimeUnit.MILLISECONDS);
            }
        }
    }
```
从上面的代码来看，可以回答一个很经典的问题，eureka注册中心挂了，服务之间还可以相互调用吗？源码告诉我们，可以的！！！
 
 通过CacheRefreshThread类的run方法可以知道他其实内部又调用了refreshRegistry方法：
```java
     /**
     * The task that fetches the registry information at specified intervals.
     *
     */
    class CacheRefreshThread implements Runnable {
        public void run() {
            refreshRegistry();
        }
    }
```
refreshRegistry方法内部会做一个判断，以决定是否全量拉取服务注册信息还是只拉取更新的服务注册信息。
 
下面`if`判断的结果会决定`remoteRegionsModified`标志位的值，真正的服务获取是在`fetchRegistry`方法内完成的。
 
```java
 @VisibleForTesting
    void refreshRegistry() {
        try {
            boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

            boolean remoteRegionsModified = false;
            // This makes sure that a dynamic change to remote regions to fetch is honored.
            String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
            if (null != latestRemoteRegions) {
                String currentRemoteRegions = remoteRegionsToFetch.get();
                if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                    // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                    synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                        if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                            String[] remoteRegions = latestRemoteRegions.split(",");
                            remoteRegionsRef.set(remoteRegions);
                            instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                            remoteRegionsModified = true;
                        } else {
                            logger.info("Remote regions to fetch modified concurrently," +
                                    " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                        }
                    }
                } else {
                    // Just refresh mapping to reflect any DNS/Property change
                    instanceRegionChecker.getAzToRegionMapper().refreshMapping();
                }
            }

            boolean success = fetchRegistry(remoteRegionsModified);
            if (success) {
                registrySize = localRegionApps.get().size();
                lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
            }

            if (logger.isDebugEnabled()) {
                StringBuilder allAppsHashCodes = new StringBuilder();
                allAppsHashCodes.append("Local region apps hashcode: ");
                allAppsHashCodes.append(localRegionApps.get().getAppsHashCode());
                allAppsHashCodes.append(", is fetching remote regions? ");
                allAppsHashCodes.append(isFetchingRemoteRegionRegistries);
                for (Map.Entry<String, Applications> entry : remoteRegionVsApps.entrySet()) {
                    allAppsHashCodes.append(", Remote region: ");
                    allAppsHashCodes.append(entry.getKey());
                    allAppsHashCodes.append(" , apps hashcode: ");
                    allAppsHashCodes.append(entry.getValue().getAppsHashCode());
                }
                logger.debug("Completed cache refresh task for discovery. All Apps hash code is {} ",
                        allAppsHashCodes);
            }
        } catch (Throwable e) {
            logger.error("Cannot fetch registry from server", e);
        }
    }
```
 
 #### 服务端接收注册
 
 上面讲的都是服务如何向eureka发起注册，那么**eureka是如何处理这些注册请求的**呢？下面我们具体看一下：
 
 
以「**服务注册**」为例：

```java
    /**
     * Registers information about a particular instance for an
     * {@link com.netflix.discovery.shared.Application}.
     *
     * @param info
     *            {@link InstanceInfo} information of the instance.
     * @param isReplication
     *            a header parameter containing information whether this is
     *            replicated from other nodes.
     */
    @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
        // validate that the instanceinfo contains all the necessary required fields
        if (isBlank(info.getId())) {
            return Response.status(400).entity("Missing instanceId").build();
        } else if (isBlank(info.getHostName())) {
            return Response.status(400).entity("Missing hostname").build();
        } else if (isBlank(info.getIPAddr())) {
            return Response.status(400).entity("Missing ip address").build();
        } else if (isBlank(info.getAppName())) {
            return Response.status(400).entity("Missing appName").build();
        } else if (!appName.equals(info.getAppName())) {
            return Response.status(400).entity("Mismatched appName, expecting " + appName + " but was " + info.getAppName()).build();
        } else if (info.getDataCenterInfo() == null) {
            return Response.status(400).entity("Missing dataCenterInfo").build();
        } else if (info.getDataCenterInfo().getName() == null) {
            return Response.status(400).entity("Missing dataCenterInfo Name").build();
        }

        // handle cases where clients may be registering with bad DataCenterInfo with missing data
        DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
        if (dataCenterInfo instanceof UniqueIdentifier) {
            String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
            if (isBlank(dataCenterInfoId)) {
                boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
                if (experimental) {
                    String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                    return Response.status(400).entity(entity).build();
                } else if (dataCenterInfo instanceof AmazonInfo) {
                    AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                    String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                    if (effectiveId == null) {
                        amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                    }
                } else {
                    logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
                }
            }
        }

        registry.register(info, "true".equals(isReplication));
        return Response.status(204).build();  // 204 to be backwards compatible
    }
```

从代码可以看出，他首先会进行一系列的非空以及合法性校验，然后会调用register函数进行真正的注册。

```java
	@Override
	public void register(final InstanceInfo info, final boolean isReplication) {
		handleRegistration(info, resolveInstanceLeaseDuration(info), isReplication);
		super.register(info, isReplication);
	}
```
handleRegistration的源码如下：
```java
	private void handleRegistration(InstanceInfo info, int leaseDuration,
			boolean isReplication) {
		log("register " + info.getAppName() + ", vip " + info.getVIPAddress()
				+ ", leaseDuration " + leaseDuration + ", isReplication "
				+ isReplication);
		publishEvent(new EurekaInstanceRegisteredEvent(this, info, leaseDuration,
				isReplication));
	}
```

注册的大致流程为：先调用publishEvent函数，将该新服务注册的时间传播出去。然后调用com.netflix.eureka.registry.AbstractInstanceRegistry抽象类的register方法进行注册：
```java
    /**
     * Registers a new instance with a given duration.
     *
     * @see com.netflix.eureka.lease.LeaseManager#register(java.lang.Object, int, boolean)
     */
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
            } else {
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        // Since the client wants to register it, increase the number of clients sending renews
                        this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                        updateRenewsPerMinThreshold();
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            synchronized (recentRegisteredQueue) {
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            read.unlock();
        }
    }
```

注册的过程中使用了读锁，可以不影响服务列表的获取，但是会阻塞其他的并发写请求。

同时，也可以看出真正的数据存储在`registry`中，而`registry`的数据结构定义如下：
```java
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry = new ConcurrentHashMap();
```
这也印证了之前说的，eureka使用了双层Map的结构来存储注册在其上面的服务实例信息。

#### 多个eureka server信息同步

- https://cloud.tencent.com/developer/article/1083131

eureka server的信息同步大致分为两部分，一部分是线程池周期性的定时同步eureka server节点间的相互发现，一部分是eureka server上注册信息发生变化触发的同步。

- 定时器同步peers信息

`DefaultEurekaServerContext`中有一个`initialize`方法，源码如下：

```java
    @PostConstruct
    @Override
    public void initialize() {
        logger.info("Initializing ...");
        peerEurekaNodes.start();
        try {
            registry.init(peerEurekaNodes);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        logger.info("Initialized");
    }
```
重点关注其中的`peerEurekaNodes.start();`语句：
```java
    public void start() {
        taskExecutor = Executors.newSingleThreadScheduledExecutor(
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                        thread.setDaemon(true);
                        return thread;
                    }
                }
        );
        try {
        // 默认直接触发一次peer是相互发现，后期依靠线程池定时执行实现peers信息更新
            updatePeerEurekaNodes(resolvePeerUrls());
            Runnable peersUpdateTask = new Runnable() {
                @Override
                public void run() {
                    try {
                        updatePeerEurekaNodes(resolvePeerUrls());
                    } catch (Throwable e) {
                        logger.error("Cannot update the replica Nodes", e);
                    }

                }
            };
            taskExecutor.scheduleWithFixedDelay(
                    peersUpdateTask,
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    TimeUnit.MILLISECONDS
            );
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
        for (PeerEurekaNode node : peerEurekaNodes) {
            logger.info("Replica node URL:  {}", node.getServiceUrl());
        }
    }
```

在`start`方法内部，定义了一个单线程的线程池`taskExecutor`，然后定义了一个`Runnable`类型的对象`peersUpdateTask`对象，从其名字可以看出其就是更新peers节点信息的task，然后这个task以定期延时任务的形式提交到线程池中，默认的初始延时10min，定期执行时间间隔10min。重点看下`updatePeerEurekaNodes`方法：
```java
    /**
     * Given new set of replica URLs, destroy {@link PeerEurekaNode}s no longer available, and
     * create new ones.
     *
     * @param newPeerUrls peer node URLs; this collection should have local node's URL filtered out
     */
    protected void updatePeerEurekaNodes(List<String> newPeerUrls) {
        if (newPeerUrls.isEmpty()) {
            logger.warn("The replica size seems to be empty. Check the route 53 DNS Registry");
            return;
        }

        Set<String> toShutdown = new HashSet<>(peerEurekaNodeUrls);
        toShutdown.removeAll(newPeerUrls);
        Set<String> toAdd = new HashSet<>(newPeerUrls);
        toAdd.removeAll(peerEurekaNodeUrls);

        if (toShutdown.isEmpty() && toAdd.isEmpty()) { // No change
            return;
        }

        // Remove peers no long available
        List<PeerEurekaNode> newNodeList = new ArrayList<>(peerEurekaNodes);

        if (!toShutdown.isEmpty()) {
            logger.info("Removing no longer available peer nodes {}", toShutdown);
            int i = 0;
            while (i < newNodeList.size()) {
                PeerEurekaNode eurekaNode = newNodeList.get(i);
                if (toShutdown.contains(eurekaNode.getServiceUrl())) {
                    newNodeList.remove(i);
                    eurekaNode.shutDown();
                } else {
                    i++;
                }
            }
        }

        // Add new peers  添加新的节点
        if (!toAdd.isEmpty()) {
            logger.info("Adding new peer nodes {}", toAdd);
            for (String peerUrl : toAdd) {
                newNodeList.add(createPeerEurekaNode(peerUrl));
            }
        }

        this.peerEurekaNodes = newNodeList;
        this.peerEurekaNodeUrls = new HashSet<>(newPeerUrls);
    }
```

代码已经展示的很直白了，就是剔除一些不可用的peers，添加新的peers节点。

- 注册信息的相互同步

在`PeerAwareInstanceRegistryImpl`类中，有一个`replicateToPeers`方法，源码如下：
```java
    /**
     * Replicates all eureka actions to peer eureka nodes except for replication
     * traffic to this node.
     *
     */
    private void replicateToPeers(Action action, String appName, String id,
                                  InstanceInfo info /* optional */,
                                  InstanceStatus newStatus /* optional */, boolean isReplication) {
        Stopwatch tracer = action.getTimer().start();
        try {
            if (isReplication) {
                numberOfReplicationsLastMin.increment();
            }
            // If it is a replication already, do not replicate again as this will create a poison replication
            if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
                return;
            }

            for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
                // If the url represents this host, do not replicate to yourself.
                if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                    continue;
                }
                replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
            }
        } finally {
            tracer.stop();
        }
    }
```
源码相对也比较简单，就是首先会做对从其他peer是同步来的instance信息进行计数，然后会判断是否有peers节点存在，且当前instance的信息不是从其他peer同步来的（因为从其他peer同步的信息，你在给别人同步回去，容易形成死循环）。换言之，eureka server只会将原生注册在自己之上的instance信息同步到其他peer。然后会调用`replicateInstanceActionsToPeers`方法进行真正的instance信息同步。








## Ribbon源码分析

ribbon主要是作为一个服务负载均衡调度，以及服务调用的工具。

ribbon调用工具实现了常用的GET、POST、PUT、DELETE等常用的REST风格的api接口。这个相对简单，下面我们探究一下他是如何实现负载均衡的？？

### @LoadBalanced注解

一般我们会使用该注解使得应用带有负载均衡的能力，那他底层的原理呢？

通过搜索LoadBalancerClient，可以发现他是一个org.springframework.cloud.client.loadbalancer下的接口。他有三个方法，如下所示：
```java
	/**
	 * execute request using a ServiceInstance from the LoadBalancer for the specified
	 * service
	 * @param serviceId the service id to look up the LoadBalancer
	 * @param request allows implementations to execute pre and post actions such as
	 * incrementing metrics
	 * @return the result of the LoadBalancerRequest callback on the selected
	 * ServiceInstance
	 */
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

	/**
	 * execute request using a ServiceInstance from the LoadBalancer for the specified
	 * service
	 * @param serviceId the service id to look up the LoadBalancer
	 * @param serviceInstance the service to execute the request to
	 * @param request allows implementations to execute pre and post actions such as
	 * incrementing metrics
	 * @return the result of the LoadBalancerRequest callback on the selected
	 * ServiceInstance
	 */
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

	/**
	 * Create a proper URI with a real host and port for systems to utilize.
	 * Some systems use a URI with the logical serivce name as the host,
	 * such as http://myservice/path/to/service.  This will replace the
	 * service name with the host:port from the ServiceInstance.
	 * @param instance
	 * @param original a URI with the host as a logical service name
	 * @return a reconstructed URI
	 */
	URI reconstructURI(ServiceInstance instance, URI original);
```

他还从ServiceInstanceChooser那里继承过来了一个方法：
```java
    /**
     * Choose a ServiceInstance from the LoadBalancer for the specified service
     * @param serviceId the service id to look up the LoadBalancer
     * @return a ServiceInstance that matches the serviceId
     */
    ServiceInstance choose(String serviceId);
```

综合来看：

 - choose：为服务的调用方选择一个合适的服务提供方
 - execute：执行具体的服务调用
 - reconstructURI：构建真实的服务调用地址，例如`http://myservice/path/to/service`，而不是`ip:port`的形式

通过整理LoadBalancerClient接口的依赖关系，得到如下图所示依赖关系：
![ribbon](./image/spring/ribbon.png)

从上图可以看出，`LoadBalancerAutoConfiguration`较为关键，从名字上我们也可以看出他是一个实现负载均衡的自动化配置的类，源码如下：
```java
package org.springframework.cloud.client.loadbalancer;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.SmartInitializingSingleton;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingClass;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.retry.backoff.BackOffPolicy;
import org.springframework.retry.support.RetryTemplate;
import org.springframework.web.client.RestTemplate;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Auto configuration for Ribbon (client side load balancing).
 *
 * @author Spencer Gibb
 * @author Dave Syer
 * @author Will Tran
 * @author Gang Li
 */
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
	}

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
	}

	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}

	@Configuration
	@ConditionalOnClass(RetryTemplate.class)
	public static class RetryAutoConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public LoadBalancedRetryFactory loadBalancedRetryFactory() {
			return new LoadBalancedRetryFactory() {};
		}
	}

	@Configuration
	@ConditionalOnClass(RetryTemplate.class)
	public static class RetryInterceptorAutoConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public RetryLoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient, LoadBalancerRetryProperties properties,
				LoadBalancerRequestFactory requestFactory,
				LoadBalancedRetryFactory loadBalancedRetryFactory) {
			return new RetryLoadBalancerInterceptor(loadBalancerClient, properties,
					requestFactory, loadBalancedRetryFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final RetryLoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}
}
```

通过类开头的注解，我们就可以得知，要实现自动装配，需要满足一定的条件：

 - @ConditionalOnClass(RestTemplate.class)   RestTemplate类必须被加载
 - @ConditionalOnBean(LoadBalancerClient.class)  在spring的bean实例工厂中要有LoadBalancerClient这个bean

该自动化配置类，主要完成了三件事情：

 1. 创建一个LoadBalancerInterceptor的bean，用于实现拦截客户端发起的请求
 2. 创建一个RestTemplateCustomizer用于给RestTemplate加上LoadBalancerInterceptor拦截器
 3. 维护了一个被@LoadBalanced注解修饰的RestTemplate对象，通过调用RestTemplateCustomizer来给需要做负载均衡的客户端加LoadBalancerInterceptor的拦截器。

```java
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();
```
 
 接下来，我们看看LoadBalancerInterceptor是如何将普通的RestTemplate变成具有负载均衡功能的。
 
 下面是LoadBalancerInterceptor的源码：
```java
import java.io.IOException;
import java.net.URI;

import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.util.Assert;

/**
 * @author Spencer Gibb
 * @author Dave Syer
 * @author Ryan Baxter
 * @author William Tran
 */
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}
```
 
 通过源码可以看到，拦截器中注入了LoadBalancerClient的实现，当客户端发起请求时，会被拦截器的intercept方法拦截。
 
 ribbon会定时ping（默认10s）服务列表里的服务。
 
 
### Ribbon的负载均衡策略
 
 ribbon本身有比较多的策略，基本都集成自IRule接口，如下图所示：
 
  ![IRule](./image/spring/IRule.png)
 
 IRule作为顶层的接口，其源码如下：
```java
 /**
 * Interface that defines a "Rule" for a LoadBalancer. A Rule can be thought of
 * as a Strategy for loadbalacing. Well known loadbalancing strategies include
 * Round Robin, Response Time based etc.
 * 
 * @author stonse
 * 
 */
public interface IRule{
    /*
     * choose one alive server from lb.allServers or
     * lb.upServers according to key
     * 
     * @return choosen Server object. NULL is returned if none
     *  server is available 
     */

    public Server choose(Object key);
    
    public void setLoadBalancer(ILoadBalancer lb);
    
    public ILoadBalancer getLoadBalancer();    
}
```
 
 从图上也可以看出，大概有轮询，随机、权重、重试、基于Zone、预判等负载策略。
 
从`BaseLoadBalancer`类可以知道Ribbon默认的rule就是`RoundRobinRule`
```java
private final static IRule DEFAULT_RULE = new RoundRobinRule();
```

可以通过配置更改使用的路由规则：
```java
ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule
```

#### 轮询

查看RoundRobinRule的源码中的`choose`方法：
```java
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
            List<Server> reachableServers = lb.getReachableServers();
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();

            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }

            int nextServerIndex = incrementAndGetModulo(serverCount);
            server = allServers.get(nextServerIndex);

            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }

            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            // Next.
            server = null;
        }

        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }

    /**
     * Inspired by the implementation of {@link AtomicInteger#incrementAndGet()}.
     *
     * @param modulo The modulo to bound the value of the counter.
     * @return The next value.
     */
    private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextServerCyclicCounter.get();
            int next = (current + 1) % modulo;
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }
```
可以看出，首先会获取所有可调用的服务方列表，然后在`incrementAndGetModulo`方法内部使用CAS的策略进行轮询操作，如果轮询过程中，某个服务不可用，会进行重试，最多重试10次。

#### 重试

我们再看看重试【retry】的`choose`源代码：
```java
	/*
	 * Loop if necessary. Note that the time CAN be exceeded depending on the
	 * subRule, because we're not spawning additional threads and returning
	 * early.
	 */
	public Server choose(ILoadBalancer lb, Object key) {
		long requestTime = System.currentTimeMillis();
		// maxRetryMillis = 500ms
		long deadline = requestTime + maxRetryMillis;

		Server answer = null;

		answer = subRule.choose(key);

		if (((answer == null) || (!answer.isAlive()))
				&& (System.currentTimeMillis() < deadline)) {

			InterruptTask task = new InterruptTask(deadline
					- System.currentTimeMillis());

			while (!Thread.interrupted()) {
				answer = subRule.choose(key);

				if (((answer == null) || (!answer.isAlive()))
						&& (System.currentTimeMillis() < deadline)) {
					/* pause and retry hoping it's transient */
					// 表示当前线程愿意让出CPU资源，但是并不一定会真正让出，需要看调度器的调度，这里也是为了防止while循环占用过多的CPU资源
					Thread.yield();
				} else {
					break;
				}
			}

			task.cancel();
		}

		if ((answer == null) || (!answer.isAlive())) {
			return null;
		} else {
			return answer;
		}
	}
```

总结下来就是，根据subRule（这里的subRule默认就是RoundRobinRule，当然也可以是其他配置的rule）的规则去匹配服务提供方answer，如果(answer为空或者不可用) 且 未到超时时间（500ms），那么就会不断尝试调用subRule的choose已获得answer。同时，会创建一个InterruptTask对象，以便在超时时间以后中断当前while循环。

### 重试机制

首选需要注意，这里的`重试机制`与上面的`重试`规则是不一样的，这里强调的是当注册中心与服务之间不能正常维持心跳机制时，依然会尝试调用服务提供方，调用失败时会进行重试（重试相关行为需要结合配置参数）。上面说的`重试`rule主要说的是在发起服务调用时，如果服务不可用，那么就会不断轮询可用的服务，直到超时时间到。

由于eureka实现的服务治理机制强调了CAP中的AP，它与zookeeper这类强调CP的服务治理不同。Eureka为了保证服务的可用性，牺牲了一定的一致性，在极端情况下，宁愿接受故障实例，也不要丢掉「健康」的实例。比如，当服务注册中心的网络发送故障时，eureka会因为超过85%的实例丢失心跳而触发保护机制，它会保留此时的所有节点，以保证大多数节点的调用是可以正常完成的（即便存在小部分故障节点）。


### 基于Ribbon实现服务灰度发布

基本思路就是，首先开启`endpoints.shutdown.enabled:  true`启用shutdown端点，以便在灰度以后可以支持优雅停机.

然后，重写自己的路由负载Rule，配置Ribbon使用该Rule，并且对于灰度时希望灰度的请求，在请求头增加标识信息。并且，对于灰度服务实例，通过`spring.instance.hostname`设置实例的名称，以和正常服务进行区分。

后续，通过请求头中的标识信息，判断是否应该走灰度服务实例。

具体，整理了一个简易流程图，如下所示：

![ribbon-gray-deploy](./image/spring/ribbon-gray-deploy.png)

灰度测试通过以后，对于原旧服务实例发送shutdown请求实现优雅关机。然后往eureka注册新的服务，最好再优雅关机灰度服务实例。
 

 

## Hystrix源码分析

### 命令模式

由于Hystrix内使用了命令模式，在开始讲解之前我们简单认识一下[命令模式](https://zhuanlan.zhihu.com/p/56949325)：

命令模式主要是为了实现解耦，简化系统编码设计，但是它的概念却并不那么简单，需要仔细思考（区别于策略模式）。命令模式一般都有命令的发出者、命令的传递者、命令的执行者三个角色。

```java
package com.audi.demo.cmd;

/**
 * 抽象命令接口
 *
 * @author WangQuanzhou
 * @date 2019-11-11
 */
public interface Command {
    void execute();
}
```

```java
package com.audi.demo.cmd;

/**
 * 具体的命令实现(传递者)
 *
 * @author WangQuanzhou
 * @date 2019-11-11
 */
public class ConcreteCommand implements Command {

    private Receiver receiver;

    public ConcreteCommand(Receiver receiver) {
        this.receiver = receiver;
    }

    @Override
    public void execute() {
        System.out.println("ConcreteCommand execute method...");
        this.receiver.action();
    }
}
```

```java
package com.audi.demo.cmd;

/**
 * 命令的接收者
 *
 * @author WangQuanzhou
 * @date 2019-11-11
 */
public class Receiver {

    public void action() {
        System.out.println("Receiver action method...");
    }
}
```

```java
package com.audi.demo.cmd;

/**
 * 客户端调用者（命令的发起者）
 *
 * @author WangQuanzhou
 * @date 2019-11-11
 */
public class Invoker {

    private Command command;

    public void setCommand(Command command) {
        this.command = command;
    }

    public void action() {
        this.command.execute();
    }

}
```

```java
package com.audi.demo.cmd;

/**
 * 测试调用
 *
 * @author WangQuanzhou
 * @date 2019-11-11
 */
public class Client {
    public static void main(String[] args) {
        Receiver receiver = new Receiver();
        Command command = new ConcreteCommand(receiver);
        Invoker invoker = new Invoker();

        invoker.setCommand(command);
        invoker.action();
    }
}
```

执行结果：
```java
ConcreteCommand execute method...
Receiver action method...

Process finished with exit code 0
```

命令模式的使用场景如下：

![Command](./image/spring/Command.png)

### 源码解析

#### command执行过程

从网上找了一张图，这张图对于hystrix的断路过程解释的挺清晰的，如下所示（原图地址：https://pbs.twimg.com/media/DRAsk2fW0AAngaX.jpg）：

![Hystrix](./image/spring/Hystrix.jpg)

按图中的数字编号顺序，一一解释如下：

 1. **【构造Command】**  构造一个HystrixCommand（阻塞，通过observe()方法可变为非阻塞）或者HystrixObservableCommand（非阻塞）对象，用来表示对依赖服务的操作请求，同时传递所需参数。
 2. **【命令执行】**  HystrixCommand实现了execute()方法和queue()方法

同步方法`execute`：
```java
    /**
     * Used for synchronous execution of command.
     * 
     * @return R
     *         Result of {@link #run()} execution or a fallback from {@link #getFallback()} if the command fails for any reason.
     * @throws HystrixRuntimeException
     *             if a failure occurs and a fallback cannot be retrieved
     * @throws HystrixBadRequestException
     *             if invalid arguments or state were used representing a user failure, not a system failure
     * @throws IllegalStateException
     *             if invoked more than once
     */
    public R execute() {
        try {
            return queue().get();
        } catch (Exception e) {
            throw Exceptions.sneakyThrow(decomposeException(e));
        }
    }
```

异步方法`queue`源码如下，如果没有配置单独的线程来执行，这个方法也会是同步执行，和`execute`的效果一样：

```java
    /**
     * Used for asynchronous execution of command.
     * <p>
     * This will queue up the command on the thread pool and return an {@link Future} to get the result once it completes.
     * <p>
     * NOTE: If configured to not run in a separate thread, this will have the same effect as {@link #execute()} and will block.
     * <p>
     * We don't throw an exception but just flip to synchronous execution so code doesn't need to change in order to switch a command from running on a separate thread to the calling thread.
     * 
     * @return {@code Future<R>} Result of {@link #run()} execution or a fallback from {@link #getFallback()} if the command fails for any reason.
     * @throws HystrixRuntimeException
     *             if a fallback does not exist
     *             <p>
     *             <ul>
     *             <li>via {@code Future.get()} in {@link ExecutionException#getCause()} if a failure occurs</li>
     *             <li>or immediately if the command can not be queued (such as short-circuited, thread-pool/semaphore rejected)</li>
     *             </ul>
     * @throws HystrixBadRequestException
     *             via {@code Future.get()} in {@link ExecutionException#getCause()} if invalid arguments or state were used representing a user failure, not a system failure
     * @throws IllegalStateException
     *             if invoked more than once
     */
    public Future<R> queue() {
        /*
         * The Future returned by Observable.toBlocking().toFuture() does not implement the
         * interruption of the execution thread when the "mayInterrupt" flag of Future.cancel(boolean) is set to true;
         * thus, to comply with the contract of Future, we must wrap around it.
         */
        final Future<R> delegate = toObservable().toBlocking().toFuture();
    	
        final Future<R> f = new Future<R>() {

            @Override
            public boolean cancel(boolean mayInterruptIfRunning) {
                if (delegate.isCancelled()) {
                    return false;
                }

                if (HystrixCommand.this.getProperties().executionIsolationThreadInterruptOnFutureCancel().get()) {
                    /*
                     * The only valid transition here is false -> true. If there are two futures, say f1 and f2, created by this command
                     * (which is super-weird, but has never been prohibited), and calls to f1.cancel(true) and to f2.cancel(false) are
                     * issued by different threads, it's unclear about what value would be used by the time mayInterruptOnCancel is checked.
                     * The most consistent way to deal with this scenario is to say that if *any* cancellation is invoked with interruption,
                     * than that interruption request cannot be taken back.
                     */
                    interruptOnFutureCancel.compareAndSet(false, mayInterruptIfRunning);
        		}

                final boolean res = delegate.cancel(interruptOnFutureCancel.get());

                if (!isExecutionComplete() && interruptOnFutureCancel.get()) {
                    final Thread t = executionThread.get();
                    if (t != null && !t.equals(Thread.currentThread())) {
                        t.interrupt();
                    }
                }

                return res;
			}

            @Override
            public boolean isCancelled() {
                return delegate.isCancelled();
			}

            @Override
            public boolean isDone() {
                return delegate.isDone();
			}

            @Override
            public R get() throws InterruptedException, ExecutionException {
                return delegate.get();
            }

            @Override
            public R get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
                return delegate.get(timeout, unit);
            }
        	
        };

        /* special handling of error states that throw immediately */
        if (f.isDone()) {
            try {
                f.get();
                return f;
            } catch (Exception e) {
                Throwable t = decomposeException(e);
                if (t instanceof HystrixBadRequestException) {
                    return f;
                } else if (t instanceof HystrixRuntimeException) {
                    HystrixRuntimeException hre = (HystrixRuntimeException) t;
                    switch (hre.getFailureType()) {
					case COMMAND_EXCEPTION:
					case TIMEOUT:
						// we don't throw these types from queue() only from queue().get() as they are execution errors
						return f;
					default:
						// these are errors we throw from queue() as they as rejection type errors
						throw hre;
					}
                } else {
                    throw Exceptions.sneakyThrow(t);
                }
            }
        }

        return f;
    }
```


 HystrixObservableCommand实现了observe()和toObservable()方法，关于此的官方文档地址在[这里](https://github.com/Netflix/Hystrix/wiki/How-To-Use#Reactive-Execution)。源码如下：
```java
     /**
     * Used for asynchronous execution of command with a callback by subscribing to the {@link Observable}.
     * <p>
     * This eagerly starts execution of the command the same as {@link HystrixCommand#queue()} and {@link HystrixCommand#execute()}.
     * <p>
     * A lazy {@link Observable} can be obtained from {@link #toObservable()}.
     * <p>
     * See https://github.com/Netflix/RxJava/wiki for more information.
     * 
     * @return {@code Observable<R>} that executes and calls back with the result of command execution or a fallback if the command fails for any reason.
     * @throws HystrixRuntimeException
     *             if a fallback does not exist
     *             <p>
     *             <ul>
     *             <li>via {@code Observer#onError} if a failure occurs</li>
     *             <li>or immediately if the command can not be queued (such as short-circuited, thread-pool/semaphore rejected)</li>
     *             </ul>
     * @throws HystrixBadRequestException
     *             via {@code Observer#onError} if invalid arguments or state were used representing a user failure, not a system failure
     * @throws IllegalStateException
     *             if invoked more than once
     */
    public Observable<R> observe() {
        // us a ReplaySubject to buffer the eagerly subscribed-to Observable
        ReplaySubject<R> subject = ReplaySubject.create();
        // eagerly kick off subscription
        final Subscription sourceSubscription = toObservable().subscribe(subject);
        // return the subject that can be subscribed to later while the execution has already started
        return subject.doOnUnsubscribe(new Action0() {
            @Override
            public void call() {
                sourceSubscription.unsubscribe();
            }
        });
    }
```
 
 

```java
     /**
     * Used for asynchronous execution of command with a callback by subscribing to the {@link Observable}.
     * <p>
     * This lazily starts execution of the command once the {@link Observable} is subscribed to.
     * <p>
     * An eager {@link Observable} can be obtained from {@link #observe()}.
     * <p>
     * See https://github.com/ReactiveX/RxJava/wiki for more information.
     * 
     * @return {@code Observable<R>} that executes and calls back with the result of command execution or a fallback if the command fails for any reason.
     * @throws HystrixRuntimeException
     *             if a fallback does not exist
     *             <p>
     *             <ul>
     *             <li>via {@code Observer#onError} if a failure occurs</li>
     *             <li>or immediately if the command can not be queued (such as short-circuited, thread-pool/semaphore rejected)</li>
     *             </ul>
     * @throws HystrixBadRequestException
     *             via {@code Observer#onError} if invalid arguments or state were used representing a user failure, not a system failure
     * @throws IllegalStateException
     *             if invoked more than once
     */
    public Observable<R> toObservable() {
        final AbstractCommand<R> _cmd = this;

        //doOnCompleted handler already did all of the SUCCESS work
        //doOnError handler already did all of the FAILURE/TIMEOUT/REJECTION/BAD_REQUEST work
        final Action0 terminateCommandCleanup = new Action0() {

            @Override
            public void call() {
                if (_cmd.commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.TERMINAL)) {
                    handleCommandEnd(false); //user code never ran
                } else if (_cmd.commandState.compareAndSet(CommandState.USER_CODE_EXECUTED, CommandState.TERMINAL)) {
                    handleCommandEnd(true); //user code did run
                }
            }
        };

        //mark the command as CANCELLED and store the latency (in addition to standard cleanup)
        final Action0 unsubscribeCommandCleanup = new Action0() {
            @Override
            public void call() {
                if (_cmd.commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.UNSUBSCRIBED)) {
                    if (!_cmd.executionResult.containsTerminalEvent()) {
                        _cmd.eventNotifier.markEvent(HystrixEventType.CANCELLED, _cmd.commandKey);
                        try {
                            executionHook.onUnsubscribe(_cmd);
                        } catch (Throwable hookEx) {
                            logger.warn("Error calling HystrixCommandExecutionHook.onUnsubscribe", hookEx);
                        }
                        _cmd.executionResultAtTimeOfCancellation = _cmd.executionResult
                                .addEvent((int) (System.currentTimeMillis() - _cmd.commandStartTimestamp), HystrixEventType.CANCELLED);
                    }
                    handleCommandEnd(false); //user code never ran
                } else if (_cmd.commandState.compareAndSet(CommandState.USER_CODE_EXECUTED, CommandState.UNSUBSCRIBED)) {
                    if (!_cmd.executionResult.containsTerminalEvent()) {
                        _cmd.eventNotifier.markEvent(HystrixEventType.CANCELLED, _cmd.commandKey);
                        try {
                            executionHook.onUnsubscribe(_cmd);
                        } catch (Throwable hookEx) {
                            logger.warn("Error calling HystrixCommandExecutionHook.onUnsubscribe", hookEx);
                        }
                        _cmd.executionResultAtTimeOfCancellation = _cmd.executionResult
                                .addEvent((int) (System.currentTimeMillis() - _cmd.commandStartTimestamp), HystrixEventType.CANCELLED);
                    }
                    handleCommandEnd(true); //user code did run
                }
            }
        };

        final Func0<Observable<R>> applyHystrixSemantics = new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                if (commandState.get().equals(CommandState.UNSUBSCRIBED)) {
                    return Observable.never();
                }
                return applyHystrixSemantics(_cmd);
            }
        };

        final Func1<R, R> wrapWithAllOnNextHooks = new Func1<R, R>() {
            @Override
            public R call(R r) {
                R afterFirstApplication = r;

                try {
                    afterFirstApplication = executionHook.onComplete(_cmd, r);
                } catch (Throwable hookEx) {
                    logger.warn("Error calling HystrixCommandExecutionHook.onComplete", hookEx);
                }

                try {
                    return executionHook.onEmit(_cmd, afterFirstApplication);
                } catch (Throwable hookEx) {
                    logger.warn("Error calling HystrixCommandExecutionHook.onEmit", hookEx);
                    return afterFirstApplication;
                }
            }
        };

        final Action0 fireOnCompletedHook = new Action0() {
            @Override
            public void call() {
                try {
                    executionHook.onSuccess(_cmd);
                } catch (Throwable hookEx) {
                    logger.warn("Error calling HystrixCommandExecutionHook.onSuccess", hookEx);
                }
            }
        };

        return Observable.defer(new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                 /* this is a stateful object so can only be used once */
                if (!commandState.compareAndSet(CommandState.NOT_STARTED, CommandState.OBSERVABLE_CHAIN_CREATED)) {
                    IllegalStateException ex = new IllegalStateException("This instance can only be executed once. Please instantiate a new instance.");
                    //TODO make a new error type for this
                    throw new HystrixRuntimeException(FailureType.BAD_REQUEST_EXCEPTION, _cmd.getClass(), getLogMessagePrefix() + " command executed multiple times - this is not permitted.", ex, null);
                }

                commandStartTimestamp = System.currentTimeMillis();

                if (properties.requestLogEnabled().get()) {
                    // log this command execution regardless of what happened
                    if (currentRequestLog != null) {
                        currentRequestLog.addExecutedCommand(_cmd);
                    }
                }

                final boolean requestCacheEnabled = isRequestCachingEnabled();
                final String cacheKey = getCacheKey();

                /* try from cache first */
                if (requestCacheEnabled) {
                    HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.get(cacheKey);
                    if (fromCache != null) {
                        isResponseFromCache = true;
                        return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                    }
                }

                Observable<R> hystrixObservable =
                        Observable.defer(applyHystrixSemantics)
                                .map(wrapWithAllOnNextHooks);

                Observable<R> afterCache;

                // put in cache
                if (requestCacheEnabled && cacheKey != null) {
                    // wrap it for caching
                    HystrixCachedObservable<R> toCache = HystrixCachedObservable.from(hystrixObservable, _cmd);
                    HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.putIfAbsent(cacheKey, toCache);
                    if (fromCache != null) {
                        // another thread beat us so we'll use the cached value instead
                        toCache.unsubscribe();
                        isResponseFromCache = true;
                        return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                    } else {
                        // we just created an ObservableCommand so we cast and return it
                        afterCache = toCache.toObservable();
                    }
                } else {
                    afterCache = hystrixObservable;
                }

                return afterCache
                        .doOnTerminate(terminateCommandCleanup)     // perform cleanup once (either on normal terminal state (this line), or unsubscribe (next line))
                        .doOnUnsubscribe(unsubscribeCommandCleanup) // perform cleanup once
                        .doOnCompleted(fireOnCompletedHook);
            }
        });
    }
```
 
 https://github.com/Netflix/Hystrix/wiki/How-To-Use#Reactive-Execution

 - observe() — returns a “hot” Observable that executes the command   
   immediately, though because the Observable is filtered through a   
   ReplaySubject you are not in danger of losing any items that it emits
   before you have a chance to subscribe
 - toObservable() — returns a “cold” Observable that won’t execute the
   command and begin emitting its results until you subscribe to the
   Observable

Hystrix底层大量使用了RxJava，即响应式编程的思想。其内部核心对象就是Observable（事件源or被观察者）和Subscriber（观察者or订阅者）。

3. **【结果是否被缓存】**  若当前命令的请求缓存功能是被启用的，并且该命令缓存命中，那么缓存的结果会立即以Observable对象的形式返回。

4. **【断路器是否开路】**  当缓存未命中时，Hystrix在执行命令前会检测断路器是否处于开路状态，如果是就执行fallback的逻辑，否则继续执行命令
5. **【线程池、请求队列、信号量是否占满】**  如果与命令相关的线程池和请求队列（阻塞队列），或者信号量（当不使用线程池的时候）已经占满，那么Hystrix也不会执行命令，而是转而执行fallback的处理逻辑。      需要**注意**的是，这里所说的线程池并非web容器的（接收）线程池，而是每个依赖服务的专有线程池。Hystrix为了保证不会因为某个依赖服务出现问题而影响到其他服务，使用了【Bulkhead Pattern】来隔离每个依赖的服务。
6. **【HystrixObservableCommand.construct()或者HystrixCommand.run()】**     HystrixCommand.run()返回单一结果，或者抛出异常；HystrixObservableCommand.construct()返回一个Observable对象用于发射多个返回值，或者提供onError发射错误通知。    如果run或者construct方法执行时间超出了命令设置的超时时间，**当前处理线程**就会抛出一个TimeOutException（如果该命令不在其自身的线程中执行，则会通过单独的计时线程来抛出）。如果命令执行失败或者执行超时，命令都会转而执行fallback。  
7. **【健康度计算】**   上一步命令执行的结果（无论正常执行或者抛出异常），都会产生一个反馈，从而影响【断路器是否开路】这一开关判断。如果将【断路器】开路，那么也不是一直开路，它有一个恢复期的概念（类似于倒计时），如果恢复期结果结束仍不能达到【断路器】闭合的条件，那么就会再次开路进入下一个恢复期。
8. **【fallback】**   也就是【服务降级】，当命令不能正常被执行的时候，都会转而执行fallback的逻辑。如果fallback不存在，那么就会抛出HystrixRuntimeException的异常。
9. **【返回成功的响应】**   当Hystrix命令执行成功以后，它会将处理结果直接返回或是以Observable对象的形式返回。

![hystrix-return-flow](./image/spring/hystrix-return-flow.png)

 - toObservable()：返回最原始的Observable，必须通过订阅它，才会真正触发命令的执行流程；
 - observe()：在toObservable()产生原始的Observable对象之后立即订阅它，让命令能够马上开始异步执行，并返回一个Observable对象。当调用它的subscribe时，将重新产生结果并通知到订阅者；
 - queue()：将toObservable()产生的原始Observable对象通过toBlocking()方法转换成BlockingObservable对象，并调用它的toFuture()方法返回异步的Future对象
 - execute()：在queue()产生异步结果Future对象之后，通过调用get()方法阻塞并等待结果的返回

#### 断路器HystrixCircuitBreaker

以上是Command的执行过程，下面看一下断路器的具体内容。

下图出自 https://github.com/alexandregama/hystrix-book

![circuit-breaker-1280](.//image/spring/circuit-breaker-1280.png)

HystrixCircuitBreaker源码位于com.netflix.hystrix.HystrixCircuitBreaker下，源码如下：
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.netflix.hystrix;

import com.netflix.hystrix.HystrixCommandMetrics.HealthCounts;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicLong;

public interface HystrixCircuitBreaker {
    boolean allowRequest();

    boolean isOpen();

    void markSuccess();

    public static class NoOpCircuitBreaker implements HystrixCircuitBreaker {
        public NoOpCircuitBreaker() {
        }

        public boolean allowRequest() {
            return true;
        }

        public boolean isOpen() {
            return false;
        }

        public void markSuccess() {
        }
    }

    public static class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker {
        private final HystrixCommandProperties properties;
        private final HystrixCommandMetrics metrics;
        private AtomicBoolean circuitOpen = new AtomicBoolean(false);
        private AtomicLong circuitOpenedOrLastTestedTime = new AtomicLong();

        protected HystrixCircuitBreakerImpl(HystrixCommandKey key, HystrixCommandGroupKey commandGroup, HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
            this.properties = properties;
            this.metrics = metrics;
        }

        public void markSuccess() {
            if (this.circuitOpen.get() && this.circuitOpen.compareAndSet(true, false)) {
                this.metrics.resetStream();
            }

        }

        public boolean allowRequest() {
            if ((Boolean)this.properties.circuitBreakerForceOpen().get()) {
                return false;
            } else if ((Boolean)this.properties.circuitBreakerForceClosed().get()) {
                this.isOpen();
                return true;
            } else {
            // 通过isOpen和allowSingleTest配合，实现断路器的打开与关闭功能
                return !this.isOpen() || this.allowSingleTest();
            }
        }

        public boolean allowSingleTest() {
            long timeCircuitOpenedOrWasLastTested = this.circuitOpenedOrLastTestedTime.get();
            return this.circuitOpen.get() && System.currentTimeMillis() > timeCircuitOpenedOrWasLastTested + (long)(Integer)this.properties.circuitBreakerSleepWindowInMilliseconds().get() && this.circuitOpenedOrLastTestedTime.compareAndSet(timeCircuitOpenedOrWasLastTested, System.currentTimeMillis());
        }

        public boolean isOpen() {
            if (this.circuitOpen.get()) {
                return true;
            } else {
                HealthCounts health = this.metrics.getHealthCounts();
                // 如果请求数量没有超过阈值（默认为20）Hystrix几乎所有的配置参数默认值都可以在 HystrixCommandProperties 找到
                if (health.getTotalRequests() < (long)(Integer)this.properties.circuitBreakerRequestVolumeThreshold().get()) {
                    return false;
                } else if (health.getErrorPercentage() < (Integer)this.properties.circuitBreakerErrorThresholdPercentage().get()) {
                // 错误的百分比没有超过阈值（默认为50）
                    return false;
                } else if (this.circuitOpen.compareAndSet(false, true)) {
                    this.circuitOpenedOrLastTestedTime.set(System.currentTimeMillis());
                    return true;
                } else {
                    return true;
                }
            }
        }
    }

    public static class Factory {
        private static ConcurrentHashMap<String, HystrixCircuitBreaker> circuitBreakersByCommand = new ConcurrentHashMap();

        public Factory() {
        }

        public static HystrixCircuitBreaker getInstance(HystrixCommandKey key, HystrixCommandGroupKey group, HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
            HystrixCircuitBreaker previouslyCached = (HystrixCircuitBreaker)circuitBreakersByCommand.get(key.name());
            if (previouslyCached != null) {
                return previouslyCached;
            } else {
                HystrixCircuitBreaker cbForCommand = (HystrixCircuitBreaker)circuitBreakersByCommand.putIfAbsent(key.name(), new HystrixCircuitBreaker.HystrixCircuitBreakerImpl(key, group, properties, metrics));
                return cbForCommand == null ? (HystrixCircuitBreaker)circuitBreakersByCommand.get(key.name()) : cbForCommand;
            }
        }

        public static HystrixCircuitBreaker getInstance(HystrixCommandKey key) {
            return (HystrixCircuitBreaker)circuitBreakersByCommand.get(key.name());
        }

        static void reset() {
            circuitBreakersByCommand.clear();
        }
    }
}
```

其中的方法都是一些判断、处理、反馈等，直接看源码注释就可以，注意一下该接口的三个内部类。

 - 内部类Factory：类中维护了Hystrix命令和HystrixCircuitBreaker的对应关系，使用ConcurrentHashMap存储该对应关系。
 - **静态**  内部类NoOpCircuitBreaker：从其注释和内部方法实现都可以看出，这个类默认允许所有的请求，断路器始终处于闭合状态
 - 内部类HystrixCircuitBreakerImpl：它是HystrixCircuitBreaker的实现类，在该类中定义了断路器的5个核心对象：


 1. HystrixCommandProperties properties：维护了HystrixCommand的配置参数。
 2. HystrixCommandMetrics metrics：记录HystrixCommand的度量指标。
 3. AtomicLong circuitOpened：断路器打开标志，默认为false
 4. AtomicReference<Status> status：断路器状态引用，默认为CLOSED闭合状态。
 5. AtomicReference<Subscription> activeSubscription：被激活的订阅，默认为空

### Hystrix线程池隔离

https://github.com/alexandregama/hystrix-book

hystrix使用线程池隔离的技术（Bulkheading-舱壁模式），来避免依赖服务之间相互影响。Hystrix uses separate, per-dependency thread pools as a way of constraining any given dependency so latency on the underlying executions will saturate the available threads only in that pool.

![Hystrix-Bulkheading](./image/spring/Hystrix-Bulkheading.png)

![request-example-with-latency-1280](./image/spring/request-example-with-latency-1280.png)

![isolation-options-1280](./image/spring/isolation-options-1280.png)

关于使用信号量替代线程池（主要是为了性能考虑），官方有如下解释：

 - You can use semaphores (or counters) to limit the number of
   concurrent calls to any given dependency, instead of using thread
   pool/queue sizes
 - This allows Hystrix to shed load without using thread pools but it
   does not allow for timing out and walking away

**Note**: if a dependency is isolated with a semaphore and then becomes latent, the parent threads will remain blocked until the underlying network calls timeout.（也就是说，信号量是没有超时机制的，如果出现调用耗时较长，那么调用它的上层线程会一直block，直到信号量调用的下层线程超时【本质的原因是因为信号量无法设置超时时间，这也是使用信号量的一个缺点】）。

通过设置execution.isolation.strategy=SEMAPHORE时，Hystrix会使用信号量替代线程池来控制依赖服务的并发。信号量默认值为10.

如果请求的线程池线程用完了怎么办？

答案很简单，直接使用拒绝新的请求，而不是将请求加入等待队列中。
Maintaning a small thread pool (or semaphore) for each depencency. if it becomes full, requests destined for that dependency will be immediately rejected instead of queued up.

### 请求合并

Hystrix会将一段时间以内的请求（默认10ms）打包一起发送，从而提高并发效率。

![collapser-1280](./image/spring/collapser-1280.png)


### Hystrix状态转换

 [参考链接](https://blog.csdn.net/qq_44209563/article/details/104697221?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

Hystrix一般存在open、closed、half-open三个状态，默认情况下hystrix处于closed状态（注意Hystrix断路器到语义）。

**closed->open**:正常情况下熔断器为closed状态，当访问同一个接口次数超过设定阈值或者错误比例超过设置错误阈值时候，就会打开熔断机制，这时候熔断器状态从closed->open。

**open->half-open**:当服务接口对应的熔断器状态为open状态时候，所有服务调用方调用该服务方法时候都是执行本地降级方法，那么什么时候才会恢复到远程调用那？Hystrix提供了一种测试策略，也就是设置了一个时间窗口，从熔断器状态变为open状态开始的一个时间窗口内，调用该服务接口时候都委托服务降级方法进行执行。如果时间超过了时间窗口，则把熔断状态从open->half-open,这时候服务调用方调用服务接口时候，就可以发起远程调用而不再使用本地降级接口，如果发起远程调用还是失败，则重新设置熔断器状态为open状态，从新记录时间窗口开始时间。

**half-open->closed**: 当熔断器状态为half-open,这时候服务调用方调用服务接口时候，就可以发起远程调用而不再使用本地降级接口，如果发起远程调用成功，则重新设置熔断器状态为closed状态。
 
### Hystrix健康度计算规则

在`HystrixCircuitBreakerImpl`中有一个`isOpen`方法，源码如下，注释我都写在里面了：
```java
       @Override
        public boolean isOpen() {
            if (circuitOpen.get()) {
                // if we're open we immediately return true and don't bother attempting to 'close' ourself as that is left to allowSingleTest and a subsequent successful test to close
                return true;
            }

            // we're closed, so let's see if errors have made us so we should trip the circuit open
            HealthCounts health = metrics.getHealthCounts();

            // check if we are past the statisticalWindowVolumeThreshold
            // 默认是10秒内要有20个请求，此时断路器才会工作。也就是说，假设10秒内总共才15个请求，即便每个都失败了，那么断路器也不会打开
            if (health.getTotalRequests() < properties.circuitBreakerRequestVolumeThreshold().get()) {
                // we are not past the minimum volume threshold for the statisticalWindow so we'll return false immediately and not calculate anything
                return false;
            }


            if (health.getErrorPercentage() < properties.circuitBreakerErrorThresholdPercentage().get()) {
                return false;
            } else {
            // 默认情况下，在10秒内，如果50%的请求都失败了，那么断路器就会处理打开的状态
                // our failure rate is too high, trip the circuit
                if (circuitOpen.compareAndSet(false, true)) {
                    // if the previousValue was false then we want to set the currentTime
                    // 设置断路器打开的时间，以便在一定的时间周期以后，断路器可以自动处于
                    circuitOpenedOrLastTestedTime.set(System.currentTimeMillis());
                    return true;
                } else {
                    // How could previousValue be true? If another thread was going through this code at the same time a race-condition could have
                    // caused another thread to set it to true already even though we were in the process of doing the same
                    // In this case, we know the circuit is open, so let the other thread set the currentTime and report back that the circuit is open
                    return true;
                }
            }
        }
```
 

## feign
feign调用的时候，如果参数使用@RequestParam或者@RequestHeader注解的话，那么**记得**要写上**value**属性，否则无法正确绑定参数抛出IllegalStateException。
```java
@ApiParam("操作者id") @RequestHeader(name = "user_id") String userId
```

feign的客户端负载均衡是通过ribbon来实现的，我们可以直接使用ribbon的配置参数实现客户端的超时时间设置。
```java
ribbon:
  ReadTimeout: 60000
  ConnectTimeout: 60000
```

上面的这种是全局性的配置，如果想针对性对某几个服务的超时时间进行配置的话（比如A服务同时调用了B、C、D服务，如果要特殊针对C、D的超时时间进行设置），那么该怎么配置呢？我们可以使用如下配置方式来进行设置。
```java
[service-name].ribbon.ReadTimeout=xxx
```

注意Hystrix的超时和Ribbon的超时，需要区分开，一般来说，Ribbon的超时都要小于Hystrix的超时时间才有意义，否则Hystrix一超时，直接熔断，那Ribbon的超时时间就没有任何意义了。

feign组件，默认会引入Ribbon和Hystrix组件。默认情况下，feign会将所有feign客户端的方法都封装到Hystrix的命令中进行服务保护。

### 重试机制

feign的重试机制，底层依赖的就是ribbon的重试机制，具体可以擦可靠之前的[笔记](https://github.com/AudiVehicle/learn/blob/master/source/Spring%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB.md#%E9%87%8D%E8%AF%95%E6%9C%BA%E5%88%B6)。在新版本的spring cloud中，重试机制的默认是`enable`的。

### 服务降级

feign的服务降级，相对来说，配置比较简单。直接使用`@FeignClient`的`fallback`属性来指定对应的服务降级的实现类。这个类应该和真正的业务处理类继承自同一个接口，只不过这个类内部的方法都是很简单的mock数据。

### 请求压缩

为了减少网络请求的开销，feign支持对请求与响应进行GZIP压缩，配置参数如下：
```java
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```
此外，我们还能限制，哪些请求or响应才需要压缩，以及数据量到多大才需要进行压缩：
```java
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

### 因为重试导致的一个bug

事情是这样的，我们有一个服务需要请求第三方服务进行电子签章，某一天一位产品找到我说，签字的结果没有回调成功，让我看看是为啥。由于这个服务之前一直运行的很好，近半年来几乎都没出过错。

当时是有点疑惑的，我首先去kibanna上看了发起签署的相关日志，一看吓一跳，相隔几秒钟，发起签署的日志打印了两次，并且traceId是一样的，只是线程id不一样。

这里简单介绍一下，服务的调用链，如下图所示：

![sign_bug](./image/spring/sign_bug.png)

当时还以为是哪位大侠手速超快（因为前端做了hover，且批量提交的数据需要手动选择），连着点了两次呢。。。但是仔细一想，不对啊

即便手速快，连着点两次，traceId不可能是一样的吧，再仔细一想就想到了feign的服务重试机制上来了。并且，两次请求的时间差刚好和配置里的ribbon readTimeOut一致。

那为什么会在短时间内发起两次调用呢？仔细查了查日志发现，是因为我们的服务调用第三方签署服务时，较长时间没有返回。触发了feign的超时重试，加上结合业务代码知道，A服务调用B服务走的Get接口，真是要命。。。这居然是个Get接口。

解决的办法有以下几种：

- B服务的接口，加锁
- B服务的接口改为Post接口
- 关闭feign的重试机制

最终，我们采取的方案是加锁+改为Post接口。

这里，再提一句，要关掉重试机制，怎么关闭？

```java
spring.cloud.loadbalancer.retry.enabled=false
```
实测，上面这样是不行的（Ribbon版本 2.3.0）。

经过仔细调试，发现需要使用下面这种方式关闭：
```java
ribbon.MaxAutoRetriesNextServer=0
```
下面把详细的过程阐述一下。

### 如何关闭重试及源码调试
- https://cloud.tencent.com/developer/article/1009212

如参考文献所说，feign本身不做任何关于服务调用的事情，底层都是依赖ribbon和hyxtrix。大概的流程是：在服务调用发起的时候，spring会拦截当前这次请求中@FeignClient注解修饰的client，然后使用IOC的思想，生成真正的RestTemplate对象，最好再把整个请求封装到Command中去。

以SynchronousMethodHandler类的invoke方法作为入口，整个调试的大概流程下图所示，部分方法省略：

![config_server](./image/spring/feign_retry_debug.png)

因此，结论就是设置如下参数：
```java
ribbon.MaxAutoRetriesNextServer=0
```


## spring cloud config

![config_server](./image/spring/config_server.png)

上面通过一张图，简单描绘了spring cloud config组件与git以及业务服务的交互方式。简单来说，作为业务服务，启动时的配置参数可以向config server要，如果config server本地有，就直接返回给业务服务。否则config server向远端的git仓库拉取相关配置，在自己本地缓存一份，同时返回给业务服务。

这里，cofig server保存一份配置在本地的好处就是，当远端git不可访问的时候，config server依然可以对外提供服务。



