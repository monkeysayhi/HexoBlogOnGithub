---
title: 源码|HDFS之DataNode：启动过程
tags:
  - Hadoop
  - HDFS
  - 面试
  - 原创
reward: true
date: 2018-01-18 12:46:40
---

掌握[Mac编译Hadoop源码](/2018/01/15/Mac编译Hadoop源码/)与[Hadoop单步debug追源码](/2018/01/15/Hadoop单步debug追源码/)后，就能告别人肉调用栈，利用IDE轻松愉快的追各种开源框架的源码啦~

今天是HDFS中DataNode的第一篇——DataNode启动过程。

<!--more-->

<!--[TOC]-->

>源码版本：Apache Hadoop 2.6.0
>
>可参考猴子追源码时的[速记](http://note.youdao.com/noteshare?id=868f0221312ea53bbf430baa172310ad)打断点，亲自debug一遍。

# 开始之前

## 总览

HDFS-2.x与1.x的核心区别：

* 为支持Federation，会为每个namespace（或称nameservice）创建BPOfferService（提供BlockPool服务）
* 为支持HA，BPOfferService还会为一个namespace下的每个namenode创建BPServiceActor（作为具体实例与各active、standby的namenode通信；一个BPOfferService下只有一个active的BPServiceActor）

datanode的启动过程主要完成以下工作：

* 启动多种工作线程，主要包括：
    * 通信：BPServiceActor、IpcServer、DataXceiverServer、LocalDataXceiverServer
    * 监控：DataBlockScanner、DirectoryScanner、JVMPauseMonitor
    * 其他：InfoServer
* 向namenode注册
* 初始化存储结构，包括各数据目录`${dfs.datanode.data.dir}`，及数据目录下各块池的存储结构
* 【可能】数据块恢复等（暂不讨论）

>LazyWriter等特性暂不讨论。

## 文章的组织结构

1. 如果只涉及单个分支的分析，则放在同一节。
2. 如果涉及多个分支的分析，则在下一级分多个节，每节讨论一个分支。
3. 多线程的分析同多分支。
4. 每一个分支和线程的组织结构遵循规则1-3。

# 主流程

datanode的Main Class是DataNode，先找DataNode.main()：

```java
  public static void main(String args[]) {
    if (DFSUtil.parseHelpArgument(args, DataNode.USAGE, System.out, true)) {
      System.exit(0);
    }

    secureMain(args, null);
  }
  
  ...
  
  public static void secureMain(String args[], SecureResources resources) {
    int errorCode = 0;
    try {
      // 打印启动信息
      StringUtils.startupShutdownMessage(DataNode.class, args, LOG);
      // 完成创建datanode的主要工作
      DataNode datanode = createDataNode(args, null, resources);
      if (datanode != null) {
        datanode.join();
      } else {
        errorCode = 1;
      }
    } catch (Throwable e) {
      LOG.fatal("Exception in secureMain", e);
      terminate(1, e);
    } finally {
      LOG.warn("Exiting Datanode");
      terminate(errorCode);
    }
  }
```

**datanode封装了非常多工作线程，但绝大多数是守护线程**，DataNode#join()只需要等待BPServiceActor线程结束，就可以正常退出（略）。

DataNode.createDataNode()：

```java
  public static DataNode createDataNode(String args[], Configuration conf,
      SecureResources resources) throws IOException {
    // 完成大部分初始化的工作，并启动部分工作线程
    DataNode dn = instantiateDataNode(args, conf, resources);
    if (dn != null) {
      // 启动剩余工作线程
      dn.runDatanodeDaemon();
    }
    return dn;
  }
```

* 在DataNode.instantiateDataNode()执行的过程中会启动部分工作线程（见后）
* DataNode#runDatanodeDaemon()启动剩余的DataXceiverServer、localDataXceiverServer、IpcServer等：

```java
  /** Start a single datanode daemon and wait for it to finish.
   *  If this thread is specifically interrupted, it will stop waiting.
   */
  public void runDatanodeDaemon() throws IOException {
    // 在DataNode.instantiateDataNode()执行过程中会调用该方法（见后）
    blockPoolManager.startAll();

    dataXceiverServer.start();
    if (localDataXceiverServer != null) {
      localDataXceiverServer.start();
    }
    ipcServer.start();
    startPlugins(conf);
  }
```

回到DataNode.instantiateDataNode()：

```java
  public static DataNode instantiateDataNode(String args [], Configuration conf,
      SecureResources resources) throws IOException {
    if (conf == null)
      conf = new HdfsConfiguration();
    
    ... // 参数检查等
    
    Collection<StorageLocation> dataLocations = getStorageLocations(conf);
    UserGroupInformation.setConfiguration(conf);
    SecurityUtil.login(conf, DFS_DATANODE_KEYTAB_FILE_KEY,
        DFS_DATANODE_KERBEROS_PRINCIPAL_KEY);
    return makeInstance(dataLocations, conf, resources);
  }
```

dataLocations维护的是全部`${dfs.datanode.data.dir}`，猴子只配置了一个目录，实际使用中会在将每块磁盘都挂载为一块目录。

从DataNode.makeInstance()开始创建DataNode：

```java
  static DataNode makeInstance(Collection<StorageLocation> dataDirs,
      Configuration conf, SecureResources resources) throws IOException {
    ...// 检查数据目录的权限

    assert locations.size() > 0 : "number of data directories should be > 0";
    return new DataNode(conf, locations, resources);
  }
  
  ...
  
  DataNode(final Configuration conf,
           final List<StorageLocation> dataDirs,
           final SecureResources resources) throws IOException {
    super(conf);
    ...// 参数设置
    
    try {
      hostName = getHostName(conf);
      LOG.info("Configured hostname is " + hostName);
      startDataNode(conf, dataDirs, resources);
    } catch (IOException ie) {
      shutdown();
      throw ie;
    }
  }
  
  ...
  
  void startDataNode(Configuration conf, 
                     List<StorageLocation> dataDirs,
                     SecureResources resources
                     ) throws IOException {
    ...// 参数设置
    
    // 初始化DataStorage
    storage = new DataStorage();
    
    // global DN settings
    // 注册JMX
    registerMXBean();
    // 初始化DataXceiver（流式通信），DataNode#runDatanodeDaemon()中启动
    initDataXceiver(conf);
    // 启动InfoServer（Web UI）
    startInfoServer(conf);
    // 启动JVMPauseMonitor（反向监控JVM情况，可通过JMX查询）
    pauseMonitor = new JvmPauseMonitor(conf);
    pauseMonitor.start();
  
    ...// 略
    
    // 初始化IpcServer（RPC通信），DataNode#runDatanodeDaemon()中启动
    initIpcServer(conf);

    metrics = DataNodeMetrics.create(conf, getDisplayName());
    metrics.getJvmMetrics().setPauseMonitor(pauseMonitor);
    
    // 按照namespace（nameservice）、namenode的二级结构进行初始化
    blockPoolManager = new BlockPoolManager(this);
    blockPoolManager.refreshNamenodes(conf);
    
    ...// 略
  }
```

BlockPoolManager抽象了datanode提供的数据块存储服务。**BlockPoolManager按照namespace（nameservice）、namenode二级结构组织**，此处按照该二级结构进行初始化。

重点是BlockPoolManager#refreshNamenodes()：

```java
  void refreshNamenodes(Configuration conf)
      throws IOException {
    LOG.info("Refresh request received for nameservices: " + conf.get
            (DFSConfigKeys.DFS_NAMESERVICES));

    Map<String, Map<String, InetSocketAddress>> newAddressMap = DFSUtil
            .getNNServiceRpcAddressesForCluster(conf);

    synchronized (refreshNamenodesLock) {
      doRefreshNamenodes(newAddressMap);
    }
  }
```

>命名为刷新，是因为除了初始化过程主动调用，还可以由namespace通过datanode心跳过程下达刷新命令。

newAddressMap是这样一个映射：`Map<namespace, Map<namenode, InetSocketAddress>>`。

BlockPoolManager#doRefreshNamenodes()：

```java
  private void doRefreshNamenodes(
      Map<String, Map<String, InetSocketAddress>> addrMap) throws IOException {
    assert Thread.holdsLock(refreshNamenodesLock);

    Set<String> toRefresh = Sets.newLinkedHashSet();
    Set<String> toAdd = Sets.newLinkedHashSet();
    Set<String> toRemove;
    
    synchronized (this) {
      // Step 1. For each of the new nameservices, figure out whether
      // it's an update of the set of NNs for an existing NS,
      // or an entirely new nameservice.
      for (String nameserviceId : addrMap.keySet()) {
        if (bpByNameserviceId.containsKey(nameserviceId)) {
          toRefresh.add(nameserviceId);
        } else {
          toAdd.add(nameserviceId);
        }
      }
      
      ...// 略
      
      // Step 3. Start new nameservices
      if (!toAdd.isEmpty()) {
        LOG.info("Starting BPOfferServices for nameservices: " +
            Joiner.on(",").useForNull("<default>").join(toAdd));
      
        for (String nsToAdd : toAdd) {
          ArrayList<InetSocketAddress> addrs =
            Lists.newArrayList(addrMap.get(nsToAdd).values());
          // 为每个namespace创建对应的BPOfferService
          BPOfferService bpos = createBPOS(addrs);
          bpByNameserviceId.put(nsToAdd, bpos);
          offerServices.add(bpos);
        }
      }
      // 然后通过startAll启动所有BPOfferService
      startAll();
    }
    
    ...// 略
  }
```

addrMap即传入的newAddressMap。Step 3为每个namespace创建对应的BPOfferService（包括每个namenode对应的BPServiceActor），然后通过BlockPoolManager#startAll()启动所有BPOfferService（实际是启动所有
BPServiceActor）。

## BlockPoolManager#createBPOS()

BlockPoolManager#createBPOS()：

```java
  protected BPOfferService createBPOS(List<InetSocketAddress> nnAddrs) {
    return new BPOfferService(nnAddrs, dn);
  }
```

`BPOfferService.<init>`：

```java
  BPOfferService(List<InetSocketAddress> nnAddrs, DataNode dn) {
    Preconditions.checkArgument(!nnAddrs.isEmpty(),
        "Must pass at least one NN.");
    this.dn = dn;

    for (InetSocketAddress addr : nnAddrs) {
      this.bpServices.add(new BPServiceActor(addr, this));
    }
  }
```

BPOfferService通过bpServices维护同一个namespace下各namenode对应的BPServiceActor。

## BlockPoolManager#startAll()

BlockPoolManager#startAll()：

```java
  synchronized void startAll() throws IOException {
    try {
      UserGroupInformation.getLoginUser().doAs(
          new PrivilegedExceptionAction<Object>() {
            @Override
            public Object run() throws Exception {
              for (BPOfferService bpos : offerServices) {
                bpos.start();
              }
              return null;
            }
          });
    } catch (InterruptedException ex) {
      IOException ioe = new IOException();
      ioe.initCause(ex.getCause());
      throw ioe;
    }
  }
```

逐个调用BPOfferService#start()，启动BPOfferService：

```java
  //This must be called only by blockPoolManager
  void start() {
    for (BPServiceActor actor : bpServices) {
      actor.start();
    }
  }
```

逐个调用BPServiceActor#start()，启动BPServiceActor：

```java
  //This must be called only by BPOfferService
  void start() {
    // 保证BPServiceActor线程只启动一次
    if ((bpThread != null) && (bpThread.isAlive())) {
      return;
    }
    bpThread = new Thread(this, formatThreadName());
    bpThread.setDaemon(true); // needed for JUnit testing
    bpThread.start();
  }
```

BPServiceActor#start()的线程安全性由最外层的BlockPoolManager#startAll()（synchronized方法）保证。

>在完成datanode的初始化后，DataNode#runDatanodeDaemon()中又调用了一次BlockPoolManager#startAll()。猴子没明白这次调用的作用，但BlockPoolManager#startAll()的内部逻辑保证其只会被执行一次，没造成什么坏影响。

## 主流程小结

在datanode启动的主流程中，启动多种重要的工作线程，包括：

* 通信：BPServiceActor、IpcServer、DataXceiverServer、LocalDataXceiverServer
* 监控：JVMPauseMonitor
* 其他：InfoServer

接下来讨论BPServiceActor线程，它的主要工作是：

* 向namonode注册
* 启动DataBlockScanner、DirectoryScanner等工作线程
* 存储结构初始化

# BPServiceActor线程

在datanode启动的主流程中，启动了多种工作线程，包括InfoServer、JVMPauseMonitor、BPServiceActor等。其中，最重要的是BPServiceActor线程，真正代表datanode与namenode通信的正是BPServiceActor线程。

BPServiceActor#run()：

```java
  @Override
  public void run() {
    LOG.info(this + " starting to offer service");

    try {
      while (true) {
        // init stuff
        try {
          // 与namonode握手，注册
          connectToNNAndHandshake();
          break;
        } catch (IOException ioe) {
          ...// 大部分握手失败的情况都需要重试，除非抛出了非IOException异常或datanode关闭
        }
      }

      runningState = RunningState.RUNNING;

      while (shouldRun()) {
        try {
          // BPServiceActor提供的服务
          offerService();
        } catch (Exception ex) {
          ...// 不管抛出任何异常，都持续提供服务（包括心跳、数据块汇报等），直到datanode关闭
        }
      }
      runningState = RunningState.EXITED;
    } catch (Throwable ex) {
      LOG.warn("Unexpected exception in block pool " + this, ex);
      runningState = RunningState.FAILED;
    } finally {
      LOG.warn("Ending block pool service for: " + this);
      cleanUp();
    }
  }
```

此处说的“通信”包括与握手、注册（BPServiceActor#connectToNNAndHandshake）和后期循环提供服务（BPServiceActor#offerService()，本文暂不讨论）。

启动过程中主要关注BPServiceActor#connectToNNAndHandshake()：

```java
  private void connectToNNAndHandshake() throws IOException {
    // get NN proxy
    bpNamenode = dn.connectToNN(nnAddr);

    // 先通过第一次握手获得namespace的信息
    NamespaceInfo nsInfo = retrieveNamespaceInfo();
    
    // 然后验证并初始化该datanode上的BlockPool
    bpos.verifyAndSetNamespaceInfo(nsInfo);
    
    // 最后，通过第二次握手向各namespace注册自己
    register();
  }
```

**通过两次握手完成了datanode的注册**，比较简单，不讨论。

重点是BPOfferService#verifyAndSetNamespaceInfo()：

```java
  /**
   * Called by the BPServiceActors when they handshake to a NN.
   * If this is the first NN connection, this sets the namespace info
   * for this BPOfferService. If it's a connection to a new NN, it
   * verifies that this namespace matches (eg to prevent a misconfiguration
   * where a StandbyNode from a different cluster is specified)
   */
  void verifyAndSetNamespaceInfo(NamespaceInfo nsInfo) throws IOException {
    writeLock();
    try {
      if (this.bpNSInfo == null) {
        // 如果是第一次连接namenode（也就必然是第一次连接namespace），则初始化blockpool（块池）
        this.bpNSInfo = nsInfo;
        boolean success = false;

        try {
          // 以BPOfferService为单位初始化blockpool
          dn.initBlockPool(this);
          success = true;
        } finally {
          if (!success) {
            // 如果一个BPServiceActor线程失败了，还可以由同BPOfferService的其他BPServiceActor线程重新尝试
            this.bpNSInfo = null;
          }
        }
      } else {
        ...// 如果不是第一次连接（刷新），则检查下信息是否正确即可
      }
    } finally {
      writeUnlock();
    }
  }
```

尽管是在BPServiceActor线程中，却试图以BPOfferService为单位初始化blockpool（包括内存与磁盘上的存储结构）。如果初始化成功，万事大吉，以后同BPOfferService的其他BPServiceActor线程发现`BPOfferService#bpNSInfo != null`就不再初始化；而如果一个BPServiceActor线程初始化blockpool失败了，还可以由同BPOfferService的其他BPServiceActor线程重新尝试初始化。

DataNode#initBlockPool()：

```java
  /**
   * One of the Block Pools has successfully connected to its NN.
   * This initializes the local storage for that block pool,
   * checks consistency of the NN's cluster ID, etc.
   * 
   * If this is the first block pool to register, this also initializes
   * the datanode-scoped storage.
   * 
   * @param bpos Block pool offer service
   * @throws IOException if the NN is inconsistent with the local storage.
   */
  void initBlockPool(BPOfferService bpos) throws IOException {
    ...// 略

    // 将blockpool注册到BlockManager
    blockPoolManager.addBlockPool(bpos);
    
    // 初步初始化存储结构
    initStorage(nsInfo);

    ...// 检查磁盘损坏

    // 启动扫描器
    initPeriodicScanners(conf);
    
    // 将blockpool添加到FsDatasetIpml，并继续初始化存储结构
    data.addBlockPool(nsInfo.getBlockPoolID(), conf);
  }
```

此时可知，**blockpool是按照namespace逐个初始化的**。这很必要，因为要支持Federation的话，就必须让多个namespace既能共用BlockManager提供的数据块存储服务，又能独立启动、关闭、升级、回滚等。

## DataNode#initStorage()

在逐个初始化blockpool之前，先以datanode整体进行初始化。这一阶段操作的主要对象是DataStorage、StorageDirectory、FsDatasetImpl、FsVolumeList、FsVolumeImpl等；后面的FsDatasetImpl#addBlockPool操作的主要对象才会具体到各blockpool。

DataNode#initStorage()：

```java
  private void initStorage(final NamespaceInfo nsInfo) throws IOException {
    final FsDatasetSpi.Factory<? extends FsDatasetSpi<?>> factory
        = FsDatasetSpi.Factory.getFactory(conf);
    
    if (!factory.isSimulated()) {
      ...// 构造参数
      // 初始化DataStorage（每个datanode分别只持有一个）。可能会触发DataStorage级别的状态装换，因此，要在DataNode上加锁
      synchronized (this) {
        storage.recoverTransitionRead(this, bpid, nsInfo, dataDirs, startOpt);
      }
      final StorageInfo bpStorage = storage.getBPStorage(bpid);
      LOG.info("Setting up storage: nsid=" + bpStorage.getNamespaceID()
          + ";bpid=" + bpid + ";lv=" + storage.getLayoutVersion()
          + ";nsInfo=" + nsInfo + ";dnuuid=" + storage.getDatanodeUuid());
    }

    ...// 检查

    // 初始化FsDatasetImpl（同上，每个datanode分别只持有一个）
    synchronized(this)  {
      if (data == null) {
        data = factory.newInstance(this, storage, conf);
      }
    }
  }
```

### 初始化DataStorage：DataStorage#recoverTransitionRead()

DataStorage#recoverTransitionRead()：

```java
  void recoverTransitionRead(DataNode datanode, String bpID, NamespaceInfo nsInfo,
      Collection<StorageLocation> dataDirs, StartupOption startOpt) throws IOException {
    // First ensure datanode level format/snapshot/rollback is completed
    recoverTransitionRead(datanode, nsInfo, dataDirs, startOpt);

    // Create list of storage directories for the block pool
    Collection<File> bpDataDirs = new ArrayList<File>();
    for(StorageLocation dir : dataDirs) {
      File dnRoot = dir.getFile();
      File bpRoot = BlockPoolSliceStorage.getBpRoot(bpID, new File(dnRoot,
          STORAGE_DIR_CURRENT));
      bpDataDirs.add(bpRoot);
    }

    // 在各${dfs.datanode.data.dir}/current下检查并创建blockpool目录
    makeBlockPoolDataDir(bpDataDirs, null);
    
    // 创建BlockPoolSliceStorage，并放入映射DataStorage#bpStorageMap：`Map<bpid, BlockPoolSliceStorage>`
    BlockPoolSliceStorage bpStorage = new BlockPoolSliceStorage(
        nsInfo.getNamespaceID(), bpID, nsInfo.getCTime(), nsInfo.getClusterID());
    bpStorage.recoverTransitionRead(datanode, nsInfo, bpDataDirs, startOpt);
    addBlockPoolStorage(bpID, bpStorage);
  }
```

根据Javadoc，BlockPoolSliceStorage管理着该datanode上相同bpid的所有BlockPoolSlice。然而，猴子暂时没有发现这个类与升级外的操作有关（当然，启动也可能是由于升级重启），暂不深入。

>* BlockPoolSlice详见后文FsVolumeImpl#addBlockPool。
>* DataStorage#recoverTransitionRead()、BlockPoolSliceStorage#recoverTransitionRead()与数据节点恢复的关系非常大，猴子暂时还没看懂，以后回来补充。

### 初始化FsDatasetImpl：FsDatasetFactory#newInstance()

FsDatasetFactory#newInstance()：

```java
  public FsDatasetImpl newInstance(DataNode datanode,
      DataStorage storage, Configuration conf) throws IOException {
    return new FsDatasetImpl(datanode, storage, conf);
  }
```

`FsDatasetImpl.<init>()`：

```java
  FsDatasetImpl(DataNode datanode, DataStorage storage, Configuration conf
      ) throws IOException {
    ...// 检查，设置参数等

    @SuppressWarnings("unchecked")
    final VolumeChoosingPolicy<FsVolumeImpl> blockChooserImpl =
        ReflectionUtils.newInstance(conf.getClass(
            DFSConfigKeys.DFS_DATANODE_FSDATASET_VOLUME_CHOOSING_POLICY_KEY,
            RoundRobinVolumeChoosingPolicy.class,
            VolumeChoosingPolicy.class), conf);
    volumes = new FsVolumeList(volsFailed, blockChooserImpl);
    
    ...// 略

    // 每一个Storagedirectory都对应一个卷FsVolumeImpl，需要将这些卷添加到FsVolumeList中
    for (int idx = 0; idx < storage.getNumStorageDirs(); idx++) {
      addVolume(dataLocations, storage.getStorageDir(idx));
    }
    
    ...// 设置lazyWriter、cacheManager等
  }
  
  ...
  
  private void addVolume(Collection<StorageLocation> dataLocations,
      Storage.StorageDirectory sd) throws IOException {
    // 使用`${dfs.datanode.data.dir}/current`目录
    final File dir = sd.getCurrentDir();
    final StorageType storageType =
        getStorageTypeFromLocations(dataLocations, sd.getRoot());

    FsVolumeImpl fsVolume = new FsVolumeImpl(
        this, sd.getStorageUuid(), dir, this.conf, storageType);
    
    ...// 略
    
    volumes.addVolume(fsVolume);
    
    ...// 略
    
    LOG.info("Added volume - " + dir + ", StorageType: " + storageType);
  }
```

初始化DataStorage的过程中，将各`${dfs.datanode.data.dir}`放入了storage（即DataNode#storage）。对于datanode来说，`${dfs.datanode.data.dir}/current`目录就是要添加的卷FsVolumeImpl。

## FsDatasetImpl#initPeriodicScanners()

FsDatasetImpl#initPeriodicScanners()（名为初始化，实为启动）：

```java
  private void initPeriodicScanners(Configuration conf) {
    initDataBlockScanner(conf);
    initDirectoryScanner(conf);
  }
```

初始化并**启动DataBlockScanner、DirectoryScanners**。

>命名为init或许是考虑到有可能禁用了数据块和目录的扫描器，导致经过FsDatasetImpl#initPeriodicScanners方法后，扫描器并没有启动。但仍然给人造成了误解。

## FsDatasetImpl#addBlockPool()

FsDatasetImpl#addBlockPool()操作的主要对象具体到了各blockpool，完成blockpool、current、rbw、tmp等目录的检查、恢复或初始化：

```java
  public void addBlockPool(String bpid, Configuration conf)
      throws IOException {
    LOG.info("Adding block pool " + bpid);
    synchronized(this) {
      // 向所有卷添加blockpool（所有namespace共享所有卷）
      volumes.addBlockPool(bpid, conf);
      // 初始化ReplicaMap中blockpool的映射
      volumeMap.initBlockPool(bpid);
    }
    // 将所有副本加载到FsDatasetImpl#volumeMap中
    volumes.getAllVolumesMap(bpid, volumeMap, ramDiskReplicaTracker);
  }
```

### FsVolumeList#addBlockPool()

FsVolumeList#addBlockPool()，并发向FsVolumeList中的所有卷添加blockpool（所有namespace共享所有卷）：

```java
  void addBlockPool(final String bpid, final Configuration conf) throws IOException {
    long totalStartTime = Time.monotonicNow();
    
    final List<IOException> exceptions = Collections.synchronizedList(
        new ArrayList<IOException>());
    List<Thread> blockPoolAddingThreads = new ArrayList<Thread>();
    // 并发向FsVolumeList中的所有卷添加blockpool（所有namespace共享所有卷）
    for (final FsVolumeImpl v : volumes) {
      Thread t = new Thread() {
        public void run() {
          try {
            ...// 时间统计
            // 向卷FsVolumeImpl添加blockpool
            v.addBlockPool(bpid, conf);
            ...// 时间统计
          } catch (IOException ioe) {
            ...// 异常处理，循环外统一处理
          }
        }
      };
      blockPoolAddingThreads.add(t);
      t.start();
    }
    for (Thread t : blockPoolAddingThreads) {
      try {
        t.join();
      } catch (InterruptedException ie) {
        throw new IOException(ie);
      }
    }
    ...// 异常处理。如果存在异常，仅抛出扫描卷过程中的第一个异常
    
    ...// 时间统计
  }
```

>正如FsVolumeList#addBlockPool()，FsVolumeList封装了很多面向所有卷的操作。

FsVolumeImpl#addBlockPool()：

```java
  void addBlockPool(String bpid, Configuration conf) throws IOException {
    File bpdir = new File(currentDir, bpid);
    // 创建BlockPoolSlice
    BlockPoolSlice bp = new BlockPoolSlice(bpid, this, bpdir, conf);
    // 维护FsVolumeImpl中bpid到BlockPoolSlice的映射
    bpSlices.put(bpid, bp);
  }
```

**BlockPoolSlice是blockpool在每个卷上的实际存在形式**。所有卷上相同bpid的BlockPoolSlice组合成小blockpool（概念上即为BlockPoolSliceStorage），再将相关datanode（向同一个namespace汇报的datanode）上相同bpid的小blockpool组合起来，就构成了该namespace的blockpool。

而FsVolumeImpl#bpSlices维护了bpid到BlockPoolSlice的映射。_FsVolumeImpl通过该映射获取bpid对应的BlockPoolSlice，而BlockPoolSlice再反向借助FsDatasetImpl中的静态方法完成实际的文件操作_（见后续文章中的写数据块过程）。

回到`BlockPoolSlice.<init>`：

```java
  BlockPoolSlice(String bpid, FsVolumeImpl volume, File bpDir,
      Configuration conf) throws IOException {
    this.bpid = bpid;
    this.volume = volume;
    this.currentDir = new File(bpDir, DataStorage.STORAGE_DIR_CURRENT); 
    this.finalizedDir = new File(
        currentDir, DataStorage.STORAGE_DIR_FINALIZED);
    this.lazypersistDir = new File(currentDir, DataStorage.STORAGE_DIR_LAZY_PERSIST);
    // 检查并创建finalized目录
    if (!this.finalizedDir.exists()) {
      if (!this.finalizedDir.mkdirs()) {
        throw new IOException("Failed to mkdirs " + this.finalizedDir);
      }
    }

    this.deleteDuplicateReplicas = conf.getBoolean(
        DFSConfigKeys.DFS_DATANODE_DUPLICATE_REPLICA_DELETION,
        DFSConfigKeys.DFS_DATANODE_DUPLICATE_REPLICA_DELETION_DEFAULT);

    // 删除tmp目录。每次启动datanode都会删除tmp目录（并重建），重新协调数据块的一致性。
    this.tmpDir = new File(bpDir, DataStorage.STORAGE_DIR_TMP);
    if (tmpDir.exists()) {
      FileUtil.fullyDelete(tmpDir);
    }
    
    // 检查并创建rbw目录
    this.rbwDir = new File(currentDir, DataStorage.STORAGE_DIR_RBW);
    final boolean supportAppends = conf.getBoolean(
        DFSConfigKeys.DFS_SUPPORT_APPEND_KEY,
        DFSConfigKeys.DFS_SUPPORT_APPEND_DEFAULT);
    // 如果不支持append，那么同tmp一样，rbw里保存的必然是新写入的数据，可以在每次启动datanode时删除rbw目录，重新协调
    if (rbwDir.exists() && !supportAppends) {
      FileUtil.fullyDelete(rbwDir);
    } // 如果支持append，待datanode启动后，有可能继续append数据，因此不能删除，等待进一步确定或恢复
    
    if (!rbwDir.mkdirs()) {
      if (!rbwDir.isDirectory()) {
        throw new IOException("Mkdirs failed to create " + rbwDir.toString());
      }
    }
    
    if (!tmpDir.mkdirs()) {
      if (!tmpDir.isDirectory()) {
        throw new IOException("Mkdirs failed to create " + tmpDir.toString());
      }
    }
    
    // 启动dfsUsage的监控线程（详见对hadoop fs shell中df、du区别的总结）
    this.dfsUsage = new DU(bpDir, conf, loadDfsUsed());
    this.dfsUsage.start();
    ShutdownHookManager.get().addShutdownHook(
      new Runnable() {
        @Override
        public void run() {
          if (!dfsUsedSaved) {
            saveDfsUsed();
          }
        }
      }, SHUTDOWN_HOOK_PRIORITY);
  }
```

可知，**每个blockpool目录下的存储结构是在构造BlockPoolSlice时初始化的**。

>关于du的作用及优化：
>
>在linux系统上，该线程将定期通过`du -sk`命令统计各blockpool目录的占用情况，随着心跳汇报给namenode。
>
>执行linux命令需要从JVM继承fork出子进程，成本较高（尽管linux使用COW策略避免了对内存空间的完全copy）。为了加快datanode启动速度，此处允许使用之前缓存的dfsUsage值，该值保存在current目录下的dfsUsed文件中；缓存的dfsUsage会定期持久化到磁盘中；在虚拟机关闭时，也会将当前的dfsUsage值持久化。

### ReplicaMap#initBlockPool()

ReplicaMap#initBlockPool()，初始化ReplicaMap中blockpool的映射：

```java
  void initBlockPool(String bpid) {
    checkBlockPool(bpid);
    synchronized(mutex) {
      Map<Long, ReplicaInfo> m = map.get(bpid);
      if (m == null) {
        // Add an entry for block pool if it does not exist already
        m = new HashMap<Long, ReplicaInfo>();
        map.put(bpid, m);
      }
    }
  }
```

FsDatasetImpl#volumeMap（ReplicaMap实例）中维护了bpid到各blockpool在该datanode上的所有副本：`Map<bpid, Map<blockid, replicaInfo>>`。

# 例行挖坑

在以后的文章中，猴子会陆续整理DataNode章的写数据块过程、读数据块过程，NameNode章、Client章等。

由于猴子也是一步步学习，难免有错漏之处，烦请读者批评指正；随着猴子进一步的学习与自检，也会随时更新文章，重要修改会注明勘误。
