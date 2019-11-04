# Spring源码阅读

标签（空格分隔）： spring 源码

---

本文主要针对springBoot、springCloud的相关组件的源码阅读及个人理解。

参考链接
----

 1. https://blog.csdn.net/woshilijiuyi/article/details/82219585
 2. https://blog.csdn.net/qq_35119422/article/details/81559410

版本
--

 - springboot  2.1.4.RELEASE
 - springcloud Greenwich.SR1

 
目录
--

 1. springboot启动过程
 2. springboot的starter
 3. 自定义starter
 4. eureka源码
 5. feign源码
 6. hystrix源码

eureka源码分析
----------

 - 各个服务组件是如何将自己注册到eureka上的呢？

简单来说就是想eureka发送REST请求将自己注册到Eureka Server上的。

 - 那么，这些注册信息又是如何在Eureka上保存的呢？

Eureka内部通过，使用双层的Map结构来保存服务的注册信息。第一层key是服务名，通过spring.application.name配置，第二层key是具体的实例名称（比如，一个服务可以启动多个实例），通过spring.instance.hostname设置。

 - 服务注册到eureka上以后，如何告诉eureka自己还”活着“呢？避免eureka将自己剔除呢？

服务于eureka会通过心跳机制，来告诉eureka，自己还”活着“。这也叫”服务续约“，关于续约，有下面两个重要的参数：
```java
spring.instance.lease-renewal-interval-in-seconds=30
spring.instance.lease-expiration-duration-in-seconds=90
```
前者定义了服务多久eureka通信一次，后者告诉eureka多久将无心跳的服务踢下线。上面的30和90秒都是默认时间，也就是说，不手动配置，这两个参数也是生效的，只不过取默认值。

 - 服务消费者如何获取eureka上注册的服务提供者呢？

 消费者会发送REST请求给eureka，以获取eureka上注册的服务清单（可能不仅仅只包含服务提供者）。处于性能考虑，eureka会维护（缓存）一份只读的服务清单来返回给客户端。该缓存清单没隔30s会更新一次。如果希望修改这个时间，可以使用如下的配置：
 ```java
 eureka.client.registry-fetch-interval-seconds=15
 ```
 消费者获取到服务清单以后（起止包含服务提供者的元数据信息），可以根据自己的实际需要进行调用，Ribbon默认采用轮训的方式进行调用，从而实现负载均衡。
 
 

 - 服务下线

正常的服务下线时，会给eureka发送下线的Rest请求，告诉eureka将自己从服务列表剔除。

但是，如果服务异常关闭了，是不会往eureka发送下线请求的，eureka会每60秒检测一次是否有超时时间超过90秒的无心跳服务，如果有就将其从服务列表清除。

 - 源码分析

一般配置一个eureka服务，我们会做两件事情：

 1. 在Application主类中配置@EnableDiscoveryClient/@EnableEurekaClient注解
 2. 在application.properties中配置defaultZone

注意：@EnableDiscoveryClient/@EnableEurekaClient功能类似，区别在于@EnableDiscoveryClient适用于包含eureka在内的多个注册中心，但是@EnableEurekaClient只适用于eureka。（@EnableEurekaServer注解也可以达到上面两个注解的效果，但是区别嘛。。。嗯  暂时不清楚，官方也没说。）

@EnableDiscoveryClient主要用来开启DiscoveryClient实例，通过梳理，可以得到如下所示的依赖关系图：

![此处输入图片的描述][1]

真正实现发现服务的是com.netflix.discovery.DiscoveryClient类，该类声明及注释如下：
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
哈哈哈哈

**服务注册源码分析**

在com.netflix.discovery.DiscoveryClient类中可以找到一个方法initScheduledTasks()，这个方法在DiscoveryClient的都早函数中，会被调用，他主要是负责启动服务列表获取、服务注册的定时任务，源码如下：
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
上面的if (clientConfig.shouldFetchRegistry())就是判断是否需要启动服务列表获取的定时任务，if (clientConfig.shouldRegisterWithEureka())就是判断是否需要将服务注册到eureka上去。定时任务的执行周期都可以配置文件进行配置，否则取默认值。

 
 在if (clientConfig.shouldRegisterWithEureka())分支内创建了一个InstanceInfoReplicator对象，这个对象实现了Runnable接口，通过查看其run方法：
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
 
 上面if判断的结果会决定remoteRegionsModified标志位的值，真正的服务获取是在fetchRegistry方法内完成的。
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


Ribbon源码分析
----------

ribbon主要是作为一个服务负载均衡调度，以及服务调用的工具。


  [1]: https://github.com/Audi-A7/learn/blob/master/image/spring/eurekaClient.png?raw=true