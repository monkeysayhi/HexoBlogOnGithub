---
title: 源码|HDFS之DataNode：写数据块（3）
tags:
  - Hadoop
  - HDFS
  - 面试
  - 原创
reward: true
date: 2018-01-23 12:41:02
---

[源码|HDFS之DataNode：写数据块（1）](/2018/01/19/源码|HDFS之DataNode：写数据块（1）/)、[源码|HDFS之DataNode：写数据块（2）](/2018/01/21/源码|HDFS之DataNode：写数据块（2）/)分别分析了无管道无异常、管道写无异常的情况下，datanode上的写数据块过程。本文分析管道写有异常的情况，假设**副本系数3**（即写数据块涉及1个客户端+3个datanode），讨论datanode对不同异常种类、不同异常时机的处理。

<!--more-->

<!--[TOC]-->

>源码版本：Apache Hadoop 2.6.0
>
>结论与实现都相对简单。可仅阅读总览。

# 开始之前

## 总览

datanode对写数据块过程中的异常处理比较简单，通常采用两种策略：

1. 当前节点抛异常，关闭上下游的IO流、socket等，以关闭管道。
2. 向上游节点发送携带故障信息的ack。

只有少部分情况采用方案2；大部分情况采用方案1，简单关闭管道了事；部分情况二者结合。

虽然异常处理策略简单，但涉及异常处理的代码却不少，整体思路参照[源码|HDFS之DataNode：写数据块（1）](/2018/01/19/源码|HDFS之DataNode：写数据块（1）/)主流程中的DataXceiver#writeBlock()方法，部分融合了[源码|HDFS之DataNode：写数据块（2）](/2018/01/21/源码|HDFS之DataNode：写数据块（2）/)中管道写的内容 。本文从主流程DataXceiver#writeBlock()入手，部分涉及DataXceiver#writeBlock()的外层方法。

>更值得关注的是写数据块的故障恢复流程，该工作由客户端主导，猴子将在对客户端的分析中讨论。

## 文章的组织结构

1. 如果只涉及单个分支的分析，则放在同一节。
2. 如果涉及多个分支的分析，则在下一级分多个节，每节讨论一个分支。
3. 多线程的分析同多分支。
4. 每一个分支和线程的组织结构遵循规则1-3。

# 主流程：DataXceiver#writeBlock()

DataXceiver#writeBlock()：

```java
  public void writeBlock(final ExtendedBlock block,
      final StorageType storageType, 
      final Token<BlockTokenIdentifier> blockToken,
      final String clientname,
      final DatanodeInfo[] targets,
      final StorageType[] targetStorageTypes, 
      final DatanodeInfo srcDataNode,
      final BlockConstructionStage stage,
      final int pipelineSize,
      final long minBytesRcvd,
      final long maxBytesRcvd,
      final long latestGenerationStamp,
      DataChecksum requestedChecksum,
      CachingStrategy cachingStrategy,
      final boolean allowLazyPersist) throws IOException {
    ...// 检查，设置参数等

    ...// 构建向上游节点或客户端回复的输出流（此处即为客户端）

    ...// 略
    
    try {
      if (isDatanode || 
          stage != BlockConstructionStage.PIPELINE_CLOSE_RECOVERY) {
        // 创建BlockReceiver，准备接收数据块
        blockReceiver = new BlockReceiver(block, storageType, in,
            peer.getRemoteAddressString(),
            peer.getLocalAddressString(),
            stage, latestGenerationStamp, minBytesRcvd, maxBytesRcvd,
            clientname, srcDataNode, datanode, requestedChecksum,
            cachingStrategy, allowLazyPersist);

        storageUuid = blockReceiver.getStorageUuid();
      } else {
        ...// 管道错误恢复相关
      }

      // 下游节点的处理：以当前节点为“客户端”，继续触发下游管道的建立
      if (targets.length > 0) {
        // 连接下游节点
        InetSocketAddress mirrorTarget = null;
        mirrorNode = targets[0].getXferAddr(connectToDnViaHostname);
        if (LOG.isDebugEnabled()) {
          LOG.debug("Connecting to datanode " + mirrorNode);
        }
        mirrorTarget = NetUtils.createSocketAddr(mirrorNode);
        mirrorSock = datanode.newSocket();
        // 尝试建立管道（下面展开）
        try {
          // 设置建立socket的超时时间、写packet的超时时间、写buf大小等
          int timeoutValue = dnConf.socketTimeout
              + (HdfsServerConstants.READ_TIMEOUT_EXTENSION * targets.length);
          int writeTimeout = dnConf.socketWriteTimeout + 
                      (HdfsServerConstants.WRITE_TIMEOUT_EXTENSION * targets.length);
          NetUtils.connect(mirrorSock, mirrorTarget, timeoutValue);
          mirrorSock.setSoTimeout(timeoutValue);
          mirrorSock.setSendBufferSize(HdfsConstants.DEFAULT_DATA_SOCKET_SIZE);
          
          // 设置当前节点到下游的输出流mirrorOut、下游到当前节点的输入流mirrorIn等
          OutputStream unbufMirrorOut = NetUtils.getOutputStream(mirrorSock,
              writeTimeout);
          InputStream unbufMirrorIn = NetUtils.getInputStream(mirrorSock);
          DataEncryptionKeyFactory keyFactory =
            datanode.getDataEncryptionKeyFactoryForBlock(block);
          IOStreamPair saslStreams = datanode.saslClient.socketSend(mirrorSock,
            unbufMirrorOut, unbufMirrorIn, keyFactory, blockToken, targets[0]);
          unbufMirrorOut = saslStreams.out;
          unbufMirrorIn = saslStreams.in;
          mirrorOut = new DataOutputStream(new BufferedOutputStream(unbufMirrorOut,
              HdfsConstants.SMALL_BUFFER_SIZE));
          mirrorIn = new DataInputStream(unbufMirrorIn);

          // 向下游节点发送建立管道的请求，未来将继续使用mirrorOut作为写packet的输出流
          new Sender(mirrorOut).writeBlock(originalBlock, targetStorageTypes[0],
              blockToken, clientname, targets, targetStorageTypes, srcDataNode,
              stage, pipelineSize, minBytesRcvd, maxBytesRcvd,
              latestGenerationStamp, requestedChecksum, cachingStrategy, false);
          mirrorOut.flush();

          // 如果是客户端发起的写数据块请求（满足），则存在管道，需要从下游节点读取建立管道的ack
          if (isClient) {
            BlockOpResponseProto connectAck =
              BlockOpResponseProto.parseFrom(PBHelper.vintPrefixed(mirrorIn));
            // 将下游节点的管道建立结果作为整个管道的建立结果（要么从尾节点到头结点都是成功的，要么都是失败的）
            mirrorInStatus = connectAck.getStatus();
            firstBadLink = connectAck.getFirstBadLink();
            if (LOG.isDebugEnabled() || mirrorInStatus != SUCCESS) {
              LOG.info("Datanode " + targets.length +
                       " got response for connect ack " +
                       " from downstream datanode with firstbadlink as " +
                       firstBadLink);
            }
          }

        } catch (IOException e) {
          // 如果是客户端发起的写数据块请求（满足），则立即向上游发送状态ERROR的ack（尽管无法保证上游已收到）
          if (isClient) {
            BlockOpResponseProto.newBuilder()
              .setStatus(ERROR)
               // NB: Unconditionally using the xfer addr w/o hostname
              .setFirstBadLink(targets[0].getXferAddr())
              .build()
              .writeDelimitedTo(replyOut);
            replyOut.flush();
          }
          // 关闭下游的IO流，socket
          IOUtils.closeStream(mirrorOut);
          mirrorOut = null;
          IOUtils.closeStream(mirrorIn);
          mirrorIn = null;
          IOUtils.closeSocket(mirrorSock);
          mirrorSock = null;
          // 如果是客户端发起的写数据块请求（满足），则重新抛出该异常
          // 然后，将跳到外层的catch块
          if (isClient) {
            LOG.error(datanode + ":Exception transfering block " +
                      block + " to mirror " + mirrorNode + ": " + e);
            throw e;
          } else {
            LOG.info(datanode + ":Exception transfering " +
                     block + " to mirror " + mirrorNode +
                     "- continuing without the mirror", e);
          }
        }
      }
      
      // 发送的第一个packet是空的，只用于建立管道。这里立即返回ack表示管道是否建立成功
      // 由于该datanode没有下游节点，则执行到此处，表示管道已经建立成功
      if (isClient && !isTransfer) {
        if (LOG.isDebugEnabled() || mirrorInStatus != SUCCESS) {
          LOG.info("Datanode " + targets.length +
                   " forwarding connect ack to upstream firstbadlink is " +
                   firstBadLink);
        }
        BlockOpResponseProto.newBuilder()
          .setStatus(mirrorInStatus)
          .setFirstBadLink(firstBadLink)
          .build()
          .writeDelimitedTo(replyOut);
        replyOut.flush();
      }

      // 接收数据块（也负责发送到下游，不过此处没有下游节点）
      if (blockReceiver != null) {
        String mirrorAddr = (mirrorSock == null) ? null : mirrorNode;
        blockReceiver.receiveBlock(mirrorOut, mirrorIn, replyOut,
            mirrorAddr, null, targets, false);

        ...// 数据块复制相关
      }

      ...// 数据块恢复相关
      
      ...// 数据块复制相关
      
    } catch (IOException ioe) {
      // 如果捕获到IOC，则直接抛出
      LOG.info("opWriteBlock " + block + " received exception " + ioe);
      throw ioe;
    } finally {
      // 不管正常还是异常，都直接关闭IO流、socket
      IOUtils.closeStream(mirrorOut);
      IOUtils.closeStream(mirrorIn);
      IOUtils.closeStream(replyOut);
      IOUtils.closeSocket(mirrorSock);
      IOUtils.closeStream(blockReceiver);
      blockReceiver = null;
    }

    ...// 更新metrics
  }
```

最后的finally块对异常处理至关重要：

正常情况不表。对于异常情况，关闭所有到下游的IO流（mirrorOut、mirrorIn）、socket（mirrorSock），关闭到上游的输出流（replyOut），关闭blockReceiver内部封装的大部分资源（通过BlockReceiver#close()完成），剩余资源如到上游的输入流（in）由外层的DataXceiver#run()中的finally块关闭。

>replyOut只是一个过滤器流，其包装的底层输出流也可以由DataXceiver#run()中的finally块关闭。限于篇幅，本文不展开。

记住此处finally块的作用，后面将多次重复该处代码，构成总览中的方案1。

下面以三个关键过程为例，分析这三个关键过程中的异常处理，及其与外层异常处理逻辑的交互。

## 本地准备：`BlockReceiver.<init>()`

根据前文的分析，`BlockReceiver.<init>()`的主要工作比较简单：在rbw目录下创建block文件和meta文件：

```java
  BlockReceiver(final ExtendedBlock block, final StorageType storageType,
      final DataInputStream in,
      final String inAddr, final String myAddr,
      final BlockConstructionStage stage, 
      final long newGs, final long minBytesRcvd, final long maxBytesRcvd, 
      final String clientname, final DatanodeInfo srcDataNode,
      final DataNode datanode, DataChecksum requestedChecksum,
      CachingStrategy cachingStrategy,
      final boolean allowLazyPersist) throws IOException {
    try{
      ...// 检查，设置参数等

      // 打开文件，准备接收数据块
      if (isDatanode) { // 数据块复制和数据块移动是由数据节点发起的，这是在tmp目录下创建block文件
        replicaInfo = datanode.data.createTemporary(storageType, block);
      } else {
        switch (stage) {
        // 对于客户端发起的写数据请求（只考虑create，不考虑append），在rbw目录下创建数据块（block文件、meta文件，数据块处于RBW状态）
        case PIPELINE_SETUP_CREATE:
          replicaInfo = datanode.data.createRbw(storageType, block, allowLazyPersist);
          datanode.notifyNamenodeReceivingBlock(
              block, replicaInfo.getStorageUuid());
          break;
        ...// 其他case
        default: throw new IOException("Unsupported stage " + stage + 
              " while receiving block " + block + " from " + inAddr);
        }
      }
      ...// 略
      
      // 对于数据块复制、数据块移动、客户端创建数据块，本质上都在创建新的block文件。对于这些情况，isCreate为true
      final boolean isCreate = isDatanode || isTransfer 
          || stage == BlockConstructionStage.PIPELINE_SETUP_CREATE;
      streams = replicaInfo.createStreams(isCreate, requestedChecksum);
      assert streams != null : "null streams!";

      ...// 计算meta文件的文件头
      // 如果需要创建新的block文件，也就需要同时创建新的meta文件，并写入文件头
      if (isCreate) {
        BlockMetadataHeader.writeHeader(checksumOut, diskChecksum);
      } 
    } catch (ReplicaAlreadyExistsException bae) {
      throw bae;
    } catch (ReplicaNotFoundException bne) {
      throw bne;
    } catch(IOException ioe) {
      // IOE通常涉及文件等资源，因此要额外清理资源
      IOUtils.closeStream(this);
      cleanupBlock();
      
      // check if there is a disk error
      IOException cause = DatanodeUtil.getCauseIfDiskError(ioe);
      DataNode.LOG.warn("IOException in BlockReceiver constructor. Cause is ",
          cause);
      
      if (cause != null) { // possible disk error
        ioe = cause;
        datanode.checkDiskErrorAsync();
      }
      
      // 重新抛出IOE
      throw ioe;
    }
  }
```

>特别提一下DataNode#checkDiskErrorAsync()，该方法异步检查是否有磁盘错误，如果错误磁盘超过阈值，就关闭datanode。但阈值的计算猴子还没有看懂，看起来是对DataStorage的理解有问题。

BlockReceiver#close()的工作已经介绍过了。需要关注的是_对ReplicaAlreadyExistsException与其他IOException的处理：重新抛出_。

>ReplicaAlreadyExistsException是IOException的子类，由FsDatasetImpl#createRbw()抛出。
>
>至于抛出IOException的情况就太多了，无权限、磁盘错误等非常原因。

重新抛出这些异常块会怎样呢？触发外层DataXceiver#writeBlock()中的catch块与finally块。

由于至今还没有建立下游管道，先让我们看看由于异常执行finally块，对上游节点产生的恶果：

* 在DataXceiver线程启动后，DataXceiver#peer中封装了当前节点到上游节点的输出流（out）与上游节点到当前节点的输入流（in）。
* 这些IO流的本质是socket，关闭当前节点端的socket后，上游节点端的socket也会在一段时间后触发超时关闭，并抛出SocketException（IOException的子类）。
* 上游节点由于socket关闭捕获到了IOException，于是也执行finally块，重复一遍当前节点的流程。

如此，**逐级关闭上游节点的管道**，直到客户端对管道关闭的异常作出处理。

>如果在创建block文件或meta文件时抛出了异常，目前没有策略及时清理rbw目录下的“无主”数据块。读者可尝试debug执行`BlockReceiver.<init>()`，在rbw目录下创建数据块后长时间不让线程继续执行，最终管道超时关闭，但rbw目录下的文件依然存在。
>
>不过数据块恢复过程可完成清理工作，此处不展开。

## 建立管道：`if (targets.length > 0) {`代码块

如果本地准备没有发生异常，则开始建立管道：

```java
      // 下游节点的处理：以当前节点为“客户端”，继续触发下游管道的建立
      if (targets.length > 0) {
        // 连接下游节点
        InetSocketAddress mirrorTarget = null;
        mirrorNode = targets[0].getXferAddr(connectToDnViaHostname);
        if (LOG.isDebugEnabled()) {
          LOG.debug("Connecting to datanode " + mirrorNode);
        }
        mirrorTarget = NetUtils.createSocketAddr(mirrorNode);
        mirrorSock = datanode.newSocket();
        // 尝试建立管道（下面展开）
        try {
          // 设置建立socket的超时时间、写packet的超时时间、写buf大小等
          int timeoutValue = dnConf.socketTimeout
              + (HdfsServerConstants.READ_TIMEOUT_EXTENSION * targets.length);
          int writeTimeout = dnConf.socketWriteTimeout + 
                      (HdfsServerConstants.WRITE_TIMEOUT_EXTENSION * targets.length);
          NetUtils.connect(mirrorSock, mirrorTarget, timeoutValue);
          mirrorSock.setSoTimeout(timeoutValue);
          mirrorSock.setSendBufferSize(HdfsConstants.DEFAULT_DATA_SOCKET_SIZE);
          
          // 设置当前节点到下游的输出流mirrorOut、下游到当前节点的输入流mirrorIn等
          OutputStream unbufMirrorOut = NetUtils.getOutputStream(mirrorSock,
              writeTimeout);
          InputStream unbufMirrorIn = NetUtils.getInputStream(mirrorSock);
          DataEncryptionKeyFactory keyFactory =
            datanode.getDataEncryptionKeyFactoryForBlock(block);
          IOStreamPair saslStreams = datanode.saslClient.socketSend(mirrorSock,
            unbufMirrorOut, unbufMirrorIn, keyFactory, blockToken, targets[0]);
          unbufMirrorOut = saslStreams.out;
          unbufMirrorIn = saslStreams.in;
          mirrorOut = new DataOutputStream(new BufferedOutputStream(unbufMirrorOut,
              HdfsConstants.SMALL_BUFFER_SIZE));
          mirrorIn = new DataInputStream(unbufMirrorIn);

          // 向下游节点发送建立管道的请求，未来将继续使用mirrorOut作为写packet的输出流
          new Sender(mirrorOut).writeBlock(originalBlock, targetStorageTypes[0],
              blockToken, clientname, targets, targetStorageTypes, srcDataNode,
              stage, pipelineSize, minBytesRcvd, maxBytesRcvd,
              latestGenerationStamp, requestedChecksum, cachingStrategy, false);
          mirrorOut.flush();

          // 如果是客户端发起的写数据块请求（满足），则存在管道，需要从下游节点读取建立管道的ack
          if (isClient) {
            BlockOpResponseProto connectAck =
              BlockOpResponseProto.parseFrom(PBHelper.vintPrefixed(mirrorIn));
            // 将下游节点的管道建立结果作为整个管道的建立结果（要么从尾节点到头结点都是成功的，要么都是失败的）
            mirrorInStatus = connectAck.getStatus();
            firstBadLink = connectAck.getFirstBadLink();
            if (LOG.isDebugEnabled() || mirrorInStatus != SUCCESS) {
              LOG.info("Datanode " + targets.length +
                       " got response for connect ack " +
                       " from downstream datanode with firstbadlink as " +
                       firstBadLink);
            }
          }

        } catch (IOException e) {
          // 如果是客户端发起的写数据块请求（满足），则立即向上游发送状态ERROR的ack（尽管无法保证上游已收到）
          if (isClient) {
            BlockOpResponseProto.newBuilder()
              .setStatus(ERROR)
               // NB: Unconditionally using the xfer addr w/o hostname
              .setFirstBadLink(targets[0].getXferAddr())
              .build()
              .writeDelimitedTo(replyOut);
            replyOut.flush();
          }
          // 关闭下游的IO流，socket
          IOUtils.closeStream(mirrorOut);
          mirrorOut = null;
          IOUtils.closeStream(mirrorIn);
          mirrorIn = null;
          IOUtils.closeSocket(mirrorSock);
          mirrorSock = null;
          // 如果是客户端发起的写数据块请求（满足），则重新抛出该异常
          // 然后，将跳到外层的catch块
          if (isClient) {
            LOG.error(datanode + ":Exception transfering block " +
                      block + " to mirror " + mirrorNode + ": " + e);
            throw e;
          } else {
            LOG.info(datanode + ":Exception transfering " +
                     block + " to mirror " + mirrorNode +
                     "- continuing without the mirror", e);
          }
        }
      }
```

根据前文对管道建立过程的分析，此处要创建到与下游节点间的部分IO流、socket。

建立资源、发送管道建立请求的过程中都有可能发生故障，抛出IOException及其子类。catch块处理这些IOException的逻辑采用了方案2：先向上游节点发送ack告知ERROR，然后关闭到下游节点的IO流（mirrorOut、mirrorIn）、关闭到下游的socket（mirrorSock）。最后，重新抛出异常，以触发外层的finally块。

此处执行的清理是外层finally块的子集，重点是多发送了一个ack，对该ack的处理留到PacketResponder线程的分析中。

不过，此时已经开始建立下游管道，再来看看由于异常执行catch块（外层finally块的分析见上），对下游节点产生的恶果：

* 初始化mirrorOut、mirrorIn、mirrorSock后，下游节点也通过DataXceiverServer建立了配套的IO流、socket等。
* 这些IO流的本质是socket，关闭当前节点端的socket后，下游节点端的socket也会在一段时间后触发超时关闭，并抛出SocketException（IOException的子类）。
* 下游节点由于socket关闭捕获到了IOException，于是也执行此处的catch块或外层的finally块，重复一遍当前节点的流程。

如此，**逐级关闭下游节点的管道**，直到客户端对管道关闭的异常作出处理。同时，由于最终会执行外层finally块，则也会**逐级关闭上游节点的管道**。

>IO流mirrorOut、mirrorIn实际上共享TCP套接字mirrorSock；in、out同理。但管子IO流时，除了底层socket，还要清理缓冲区等资源，因此，将它们分别列出是合理的。

## 管道写：BlockReceiver#receiveBlock()

根据前文的分析，如果管道成功建立，则BlockReceiver#receiveBlock()开始接收packet并响应ack：

```java
  void receiveBlock(
      DataOutputStream mirrOut, // output to next datanode
      DataInputStream mirrIn,   // input from next datanode
      DataOutputStream replyOut,  // output to previous datanode
      String mirrAddr, DataTransferThrottler throttlerArg,
      DatanodeInfo[] downstreams,
      boolean isReplaceBlock) throws IOException {
    ...// 参数设置

    try {
      // 如果是客户端发起的写请求（此处即为数据块create），则启动PacketResponder发送ack
      if (isClient && !isTransfer) {
        responder = new Daemon(datanode.threadGroup, 
            new PacketResponder(replyOut, mirrIn, downstreams));
        responder.start(); // start thread to processes responses
      }

      // 同步接收packet，写block文件和meta文件
      while (receivePacket() >= 0) {}

      // 此时，节点已接收了所有packet，可以等待发送完所有ack后关闭responder
      if (responder != null) {
        ((PacketResponder)responder.getRunnable()).close();
        responderClosed = true;
      }

      ...// 数据块复制相关

    } catch (IOException ioe) {
      if (datanode.isRestarting()) {
        LOG.info("Shutting down for restart (" + block + ").");
      } else {
        LOG.info("Exception for " + block, ioe);
        throw ioe;
      }
    } finally {
      ...// 清理
    }
  }
```

仍旧分接收packet与响应ack两部分讨论。

### 同步接收packet：BlockReceiver#receivePacket()

根据前文的分析，BlockReceiver#receivePacket()负责接收上游的packet，并继续向下游节点管道写：

```java
  private int receivePacket() throws IOException {
    // read the next packet
    packetReceiver.receiveNextPacket(in);

    PacketHeader header = packetReceiver.getHeader();
    ...// 略

    // 检查packet头
    if (header.getOffsetInBlock() > replicaInfo.getNumBytes()) {
      throw new IOException("Received an out-of-sequence packet for " + block + 
          "from " + inAddr + " at offset " + header.getOffsetInBlock() +
          ". Expecting packet starting at " + replicaInfo.getNumBytes());
    }
    if (header.getDataLen() < 0) {
      throw new IOException("Got wrong length during writeBlock(" + block + 
                            ") from " + inAddr + " at offset " + 
                            header.getOffsetInBlock() + ": " +
                            header.getDataLen()); 
    }

    long offsetInBlock = header.getOffsetInBlock();
    long seqno = header.getSeqno();
    boolean lastPacketInBlock = header.isLastPacketInBlock();
    final int len = header.getDataLen();
    boolean syncBlock = header.getSyncBlock();

    ...// 略
    
    // 如果不需要立即持久化也不需要校验收到的数据，则可以立即委托PacketResponder线程返回 SUCCESS 的ack，然后再进行校验和持久化
    if (responder != null && !syncBlock && !shouldVerifyChecksum()) {
      ((PacketResponder) responder.getRunnable()).enqueue(seqno,
          lastPacketInBlock, offsetInBlock, Status.SUCCESS);
    }

    // 管道写相关：将in中收到的packet镜像写入mirrorOut
    if (mirrorOut != null && !mirrorError) {
      try {
        long begin = Time.monotonicNow();
        packetReceiver.mirrorPacketTo(mirrorOut);
        mirrorOut.flush();
        long duration = Time.monotonicNow() - begin;
        if (duration > datanodeSlowLogThresholdMs) {
          LOG.warn("Slow BlockReceiver write packet to mirror took " + duration
              + "ms (threshold=" + datanodeSlowLogThresholdMs + "ms)");
        }
      } catch (IOException e) {
        // 假设没有发生中断，则此处仅仅标记mirrorError = true
        handleMirrorOutError(e);
      }
    }
    
    ByteBuffer dataBuf = packetReceiver.getDataSlice();
    ByteBuffer checksumBuf = packetReceiver.getChecksumSlice();
    
    if (lastPacketInBlock || len == 0) {    // 收到空packet可能是表示心跳或数据块发送
      // 这两种情况都可以尝试把之前的数据刷到磁盘
      if (syncBlock) {
        flushOrSync(true);
      }
    } else {    // 否则，需要持久化packet
      final int checksumLen = diskChecksum.getChecksumSize(len);
      final int checksumReceivedLen = checksumBuf.capacity();

      // packet头有错误，直接抛出IOE
      if (checksumReceivedLen > 0 && checksumReceivedLen != checksumLen) {
        throw new IOException("Invalid checksum length: received length is "
            + checksumReceivedLen + " but expected length is " + checksumLen);
      }

      // 如果是管道中的最后一个节点，则持久化之前，要先对收到的packet做一次校验（使用packet本身的校验机制）
      if (checksumReceivedLen > 0 && shouldVerifyChecksum()) {
        try {
          // 如果校验失败，抛出IOE
          verifyChunks(dataBuf, checksumBuf);
        } catch (IOException ioe) {
          // 如果校验错误，则委托PacketResponder线程返回 ERROR_CHECKSUM 的ack
          if (responder != null) {
            try {
              ((PacketResponder) responder.getRunnable()).enqueue(seqno,
                  lastPacketInBlock, offsetInBlock,
                  Status.ERROR_CHECKSUM);
              // 等3s，期望PacketResponder线程能把所有ack都发送完（这样就不需要重新发送那么多packet了）
              Thread.sleep(3000);
            } catch (InterruptedException e) {
              // 不做处理，也不清理中断标志，仅仅停止sleep
            }
          }
          // 如果校验错误，则认为上游节点收到的packet也是错误的，直接抛出IOE
          throw new IOException("Terminating due to a checksum error." + ioe);
        }
 
        ...// checksum 翻译相关
      }

      if (checksumReceivedLen == 0 && !streams.isTransientStorage()) {
        // checksum is missing, need to calculate it
        checksumBuf = ByteBuffer.allocate(checksumLen);
        diskChecksum.calculateChunkedSums(dataBuf, checksumBuf);
      }

      final boolean shouldNotWriteChecksum = checksumReceivedLen == 0
          && streams.isTransientStorage();
      try {
        long onDiskLen = replicaInfo.getBytesOnDisk();
        if (onDiskLen<offsetInBlock) {
          ...// 如果校验块不完整，需要加载并调整旧的meta文件内容，供后续重新计算crc

          // 写block文件
          int startByteToDisk = (int)(onDiskLen-firstByteInBlock) 
              + dataBuf.arrayOffset() + dataBuf.position();
          int numBytesToDisk = (int)(offsetInBlock-onDiskLen);
          out.write(dataBuf.array(), startByteToDisk, numBytesToDisk);
          
          // 写meta文件
          final byte[] lastCrc;
          if (shouldNotWriteChecksum) {
            lastCrc = null;
          } else if (partialCrc != null) {  // 如果是校验块不完整（之前收到过一部分）
            ...// 重新计算crc
            ...// 更新lastCrc
            checksumOut.write(buf);
            partialCrc = null;
          } else { // 如果校验块完整
            ...// 更新lastCrc
            checksumOut.write(checksumBuf.array(), offset, checksumLen);
          }

          ...//略
        }
      } catch (IOException iex) {
        // 异步检查磁盘
        datanode.checkDiskErrorAsync();
        // 重新抛出IOE
        throw iex;
      }
    }

    // 相反的，如果需要立即持久化或需要校验收到的数据，则现在已经完成了持久化和校验，可以委托PacketResponder线程返回 SUCCESS 的ack
    // if sync was requested, put in queue for pending acks here
    // (after the fsync finished)
    if (responder != null && (syncBlock || shouldVerifyChecksum())) {
      ((PacketResponder) responder.getRunnable()).enqueue(seqno,
          lastPacketInBlock, offsetInBlock, Status.SUCCESS);
    }

    ...// 如果超过了响应时间，还要主动发送一个IN_PROGRESS的ack，防止超时

    ...// 节流器相关
    
    // 当整个数据块都发送完成之前，客户端会可能会发送有数据的packet，也因为维持心跳或表示结束写数据块发送空packet
    // 因此，当标志位lastPacketInBlock为true时，不能返回0，要返回一个负值，以区分未到达最后一个packet之前的情况
    return lastPacketInBlock?-1:len;
  }
  
  ...
  
  private boolean shouldVerifyChecksum() {
    // 对于客户端写，只有管道中的最后一个节点满足`mirrorOut == null`
    return (mirrorOut == null || isDatanode || needsChecksumTranslation);
  }
  
  ...
  
  private void handleMirrorOutError(IOException ioe) throws IOException {
    String bpid = block.getBlockPoolId();
    LOG.info(datanode.getDNRegistrationForBP(bpid)
        + ":Exception writing " + block + " to mirror " + mirrorAddr, ioe);
    if (Thread.interrupted()) { // 如果BlockReceiver线程被中断了，则重新抛出IOE
      throw ioe;
    } else {    // 否则，仅仅标记下游节点错误，交给外层处理
      mirrorError = true;
    }
  }
```

对管道写过程的分析要分尾节点与中间节点两种情况展开：

* 如果是尾节点，则持久化之前，要先对收到的packet做一次校验（使用packet本身的校验机制）。如果校验失败，则委托PacketResponder线程发送ERROR_CHECKSUM状态的ack，并再次抛出IOE。
* 如果是中间节点，则只需要向下游镜像写packet。假设在非中断的情况下发生异常，则仅仅标记`mirrorError = true`。这造成两个影响：
    1. 后续包都不会再写往下游节点，最终socket超时关闭，并逐级关闭上下游管道。
    2. 上游将通过ack得知下游发生了错误（见后）。

尾节点异常的处理还是走方案1，中间节点同时走方案1与方案2。

### 异步发送ack：PacketResponder线程

根据前文的分析，PacketResponder线程负责接收下游节点的ack，并继续向上游管道响应：

```java
    public void run() {
      boolean lastPacketInBlock = false;
      final long startTime = ClientTraceLog.isInfoEnabled() ? System.nanoTime() : 0;
      while (isRunning() && !lastPacketInBlock) {
        long totalAckTimeNanos = 0;
        boolean isInterrupted = false;
        try {
          Packet pkt = null;
          long expected = -2;
          PipelineAck ack = new PipelineAck();
          long seqno = PipelineAck.UNKOWN_SEQNO;
          long ackRecvNanoTime = 0;
          try {
            // 如果当前节点不是管道的最后一个节点，且下游节点正常，则从下游读取ack
            if (type != PacketResponderType.LAST_IN_PIPELINE && !mirrorError) {
              ack.readFields(downstreamIn);
              ...// 统计相关
              // 如果下游状态为OOB，则继续向上游发送OOB
              Status oobStatus = ack.getOOBStatus();
              if (oobStatus != null) {
                LOG.info("Relaying an out of band ack of type " + oobStatus);
                sendAckUpstream(ack, PipelineAck.UNKOWN_SEQNO, 0L, 0L,
                    Status.SUCCESS);
                continue;
              }
              seqno = ack.getSeqno();
            }
            // 如果从下游节点收到了正常的 ack，或当前节点是管道的最后一个节点，则需要从队列中消费pkt（即BlockReceiver#receivePacket()放入的ack）
            if (seqno != PipelineAck.UNKOWN_SEQNO
                || type == PacketResponderType.LAST_IN_PIPELINE) {
              pkt = waitForAckHead(seqno);
              if (!isRunning()) {
                break;
              }
              // 管道写用seqno控制packet的顺序：当且仅当下游正确接收的序号与当前节点正确处理完的序号相等时，当前节点才认为该序号的packet已正确接收；上游同理
              expected = pkt.seqno;
              if (type == PacketResponderType.HAS_DOWNSTREAM_IN_PIPELINE
                  && seqno != expected) {
                throw new IOException(myString + "seqno: expected=" + expected
                    + ", received=" + seqno);
              }
              ...// 统计相关
              lastPacketInBlock = pkt.lastPacketInBlock;
            }
          } catch (InterruptedException ine) {
            // 记录异常标记，标志当前InterruptedException
            isInterrupted = true;
          } catch (IOException ioe) {
            ...// 异常处理
            if (Thread.interrupted()) { // 如果发生了中断（与本地变量isInterrupted区分），则记录中断标记
              isInterrupted = true;
            } else {
              // 这里将所有异常都标记mirrorError = true不太合理，但影响不大
              mirrorError = true;
              LOG.info(myString, ioe);
            }
          }

          // 中断退出
          if (Thread.interrupted() || isInterrupted) {
            LOG.info(myString + ": Thread is interrupted.");
            running = false;
            continue;
          }

          // 如果是最后一个packet，将block的状态转换为FINALIZED，并关闭BlockReceiver
          if (lastPacketInBlock) {
            finalizeBlock(startTime);
          }

          // 此时，必然满足 ack.seqno == pkt.seqno，构造新的 ack 发送给上游
          sendAckUpstream(ack, expected, totalAckTimeNanos,
              (pkt != null ? pkt.offsetInBlock : 0), 
              (pkt != null ? pkt.ackStatus : Status.SUCCESS));
          // 已经处理完队头元素，出队
          // 只有一种情况下满足pkt == null：PacketResponder#isRunning()返回false，即PacketResponder线程正在关闭。此时无论队列中是否有元素，都不需要出队了
          if (pkt != null) {
            removeAckHead();
          }
        } catch (IOException e) {
          // 一旦发现IOE，如果不是因为中断引起的，就中断线程
          LOG.warn("IOException in BlockReceiver.run(): ", e);
          if (running) {
            datanode.checkDiskErrorAsync();
            LOG.info(myString, e);
            running = false;
            if (!Thread.interrupted()) { // failure not caused by interruption
              receiverThread.interrupt();
            }
          }
        } catch (Throwable e) {
          // 其他异常则直接中断
          if (running) {
            LOG.info(myString, e);
            running = false;
            receiverThread.interrupt();
          }
        }
      }
      LOG.info(myString + " terminating");
    }
    
    ...
    
    // PacketResponder#sendAckUpstream()封装了PacketResponder#sendAckUpstreamUnprotected()
    private void sendAckUpstreamUnprotected(PipelineAck ack, long seqno,
        long totalAckTimeNanos, long offsetInBlock, Status myStatus)
        throws IOException {
      Status[] replies = null;
      if (ack == null) { // 发送OOB ack时，要求ack为null，myStatus为OOB。什么破设计。。。
        replies = new Status[1];
        replies[0] = myStatus;
      } else if (mirrorError) { // 前面置为true的mirrorError，在此处派上用场
        replies = MIRROR_ERROR_STATUS;
      } else {  // 否则，正常构造replies
        short ackLen = type == PacketResponderType.LAST_IN_PIPELINE ? 0 : ack
            .getNumOfReplies();
        replies = new Status[1 + ackLen];
        replies[0] = myStatus;
        for (int i = 0; i < ackLen; i++) {
          replies[i + 1] = ack.getReply(i);
        }
        // 如果下游有ERROR_CHECKSUM，则抛出IOE，中断当前节点的PacketResponder线程（结合后面的代码，能保证从第一个ERROR_CHECKSUM节点开始，上游的所有节点都是ERROR_CHECKSUM的）
        if (ackLen > 0 && replies[1] == Status.ERROR_CHECKSUM) {
          throw new IOException("Shutting down writer and responder "
              + "since the down streams reported the data sent by this "
              + "thread is corrupt");
        }
      }
      
      // 构造replyAck，发送到上游
      PipelineAck replyAck = new PipelineAck(seqno, replies,
          totalAckTimeNanos);
      if (replyAck.isSuccess()
          && offsetInBlock > replicaInfo.getBytesAcked()) {
        replicaInfo.setBytesAcked(offsetInBlock);
      }
      long begin = Time.monotonicNow();
      replyAck.write(upstreamOut);
      upstreamOut.flush();
      long duration = Time.monotonicNow() - begin;
      if (duration > datanodeSlowLogThresholdMs) {
        LOG.warn("Slow PacketResponder send ack to upstream took " + duration
            + "ms (threshold=" + datanodeSlowLogThresholdMs + "ms), " + myString
            + ", replyAck=" + replyAck);
      } else if (LOG.isDebugEnabled()) {
        LOG.debug(myString + ", replyAck=" + replyAck);
      }

      // 如果当前节点是ERROR_CHECKSUM状态，则发送ack后，抛出IOE
      if (myStatus == Status.ERROR_CHECKSUM) {
        throw new IOException("Shutting down writer and responder "
            + "due to a checksum error in received data. The error "
            + "response has been sent upstream.");
      }
    }
```

对于OOB，还要关注PipelineAck#getOOBStatus()：

```java
  public Status getOOBStatus() {
    // seqno不等于UNKOWN_SEQNO的话，就一定不是OOB状态
    if (getSeqno() != UNKOWN_SEQNO) {
      return null;
    }
    // 有任何一个下游节点是OOB，则认为下游管道是OOB状态（当然，该机制保证从第一个OOB节点开始，在每个节点查看ack时，都能发现下游有节点OOB）
    for (Status reply : proto.getStatusList()) {
      // The following check is valid because protobuf guarantees to
      // preserve the ordering of enum elements.
      if (reply.getNumber() >= OOB_START && reply.getNumber() <= OOB_END) {
        return reply;
      }
    }
    return null;
  }
```

与之前的分支相比，PacketResponder线程**大量使用中断来代替抛异常使线程终止**。除此之外，关于OOB状态与ERROR_CHECKSUM状态的处理有些特殊：

* OOB状态：**将第一个OOB节点的状态，传递到客户端**。OOB是由datanode重启引起的，因此，第一个OOB节点在发送OOB的ack后，就不会再发送其他ack，最终由于引起socket超时引起整个管道的关闭。
* `ERROR_CHECKSUM`状态：**只有尾节点可能发出`ERROR_CHECKSUM`状态的ack，发送后抛出IOE主动关闭PacketResponder线程**；**然后上游节点收到`ERROR_CHECKSUM`状态的ack后，也将抛出IOE关闭PacketResponder线程**，但不再发送ack；如果还有上游节点，将因为长期收不到ack，socket超时关闭。最终关闭整个管道。

需要注意的，OOB通常能保证传递到客户端；但尾节点发送的`ERROR_CHECKSUM`无法保证被上游节点发现（先发ack再抛IOE只是一种努力，不过通常能保证），如果多于两个备份，则一定不会被客户端发现。

>猴子没明白为什么此处要使用中断使线程终止。

# 总结

尽管总览中列出了两种方案，但可以看到，作为异常处理的主要方式，主要还是依靠方案1：抛异常关socket，然后逐级导致管道关闭。

关闭管道后，由客户端决定后续处理，如数据块恢复等。
