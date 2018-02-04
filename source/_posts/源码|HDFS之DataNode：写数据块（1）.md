---
title: 源码|HDFS之DataNode：写数据块（1）
tags:
  - Hadoop
  - HDFS
  - 面试
  - 原创
reward: true
date: 2018-01-19 17:55:06
---

作为分布式文件系统，HDFS擅于处理大文件的读/写。这得益于“文件元信息与文件数据分离，文件数据分块存储”的思想：namenode管理文件元信息，datanode管理分块的文件数据。

HDFS 2.x进一步将数据块存储服务抽象为blockpool，不过写数据块过程与1.x大同小异。本文假设**副本系数1**（即写数据块只涉及1个客户端+1个datanode），**未发生任何异常**，分析datanode写数据块的过程。

<!--more-->

<!--[TOC]-->

>源码版本：Apache Hadoop 2.6.0
>
>可参考猴子追源码时的[速记](http://note.youdao.com/noteshare?id=84376f19ed7d5096a4f3472e20539df6)打断点，亲自debug一遍。
>
>副本系数1，即只需要一个datanode构成最小的管道，与更常见的管道写相比，可以认为“无管道”。后续再写两篇文章分别分析管道写无异常、管道写有异常两种情况。

# 开始之前

## 总览

参考[源码|HDFS之DataNode：启动过程](/2018/01/18/源码|HDFS之DataNode：启动过程/)，我们大体了解了datanode上有哪些重要的工作线程。其中，与写数据块过程联系最紧密的是DataXceiverServer与BPServiceActor。

参考[HDFS-1.x、2.x的RPC接口](/2018/01/09/HDFS-1.x、2.x的RPC接口/)，客户端与数据节点间主要通过流接口DataTransferProtocol完成数据块的读/写。DataTransferProtocol用于整个管道中的客户端、数据节点间的流式通信，其中，DataTransferProtocol#writeBlock()负责完成写数据块的工作：

```java
  public void writeBlock(final ExtendedBlock blk,
      final StorageType storageType, 
      final Token<BlockTokenIdentifier> blockToken,
      final String clientName,
      final DatanodeInfo[] targets,
      final StorageType[] targetStorageTypes, 
      final DatanodeInfo source,
      final BlockConstructionStage stage,
      final int pipelineSize,
      final long minBytesRcvd,
      final long maxBytesRcvd,
      final long latestGenerationStamp,
      final DataChecksum requestedChecksum,
      final CachingStrategy cachingStrategy,
      final boolean allowLazyPersist) throws IOException;
```

## 文章的组织结构

1. 如果只涉及单个分支的分析，则放在同一节。
2. 如果涉及多个分支的分析，则在下一级分多个节，每节讨论一个分支。
3. 多线程的分析同多分支。
4. 每一个分支和线程的组织结构遵循规则1-3。

# DataXceiverServer线程

>注意，DataTransferProtocol并不是一个RPC协议，因此，常见通过的寻找DataTransferProtocol接口的实现类来确定“客户端调用的远程方法”是站不住脚。不过依然可以按照这个思路倒追，看实现类究竟是如何被创建，与谁通信，来验证是否找到了正确的实现类。
>
>依靠debug，猴子从DataXceiver类反向追到了DataXceiverServer类。这里从DataXceiverServer类开始，正向讲解。

DataXceiverServer线程在DataNode#runDatanodeDaemon()方法中启动。

DataXceiverServer#run()：

```java
  public void run() {
    Peer peer = null;
    while (datanode.shouldRun && !datanode.shutdownForUpgrade) {
      try {
        peer = peerServer.accept();

        ...// 检查DataXceiver线程的数量，超过最大限制就抛出IOE

        // 启动一个新的DataXceiver线程
        new Daemon(datanode.threadGroup,
            DataXceiver.create(peer, datanode, this))
            .start();
      } catch (SocketTimeoutException ignored) {
        // wake up to see if should continue to run
      } catch (AsynchronousCloseException ace) {
        // another thread closed our listener socket - that's expected during shutdown,
        // but not in other circumstances
        if (datanode.shouldRun && !datanode.shutdownForUpgrade) {
          LOG.warn(datanode.getDisplayName() + ":DataXceiverServer: ", ace);
        }
      } catch (IOException ie) {
        ...// 清理
      } catch (OutOfMemoryError ie) {
        ...// 清理并sleep 30s
      } catch (Throwable te) {
        // 其他异常就关闭datanode
        LOG.error(datanode.getDisplayName()
            + ":DataXceiverServer: Exiting due to: ", te);
        datanode.shouldRun = false;
      }
    }

    ...// 关闭peerServer并清理所有peers
  }
```

DataXceiverServer线程是一个典型的Tcp Socket Server。客户端每来一个TCP请求，如果datanode上的DataXceiver线程数量还没超过限制，就启动一个新的DataXceiver线程。

>默认的最大DataXceiver线程数量为4096，通过`dfs.datanode.max.transfer.threads`设置。

# 主流程：DataXceiver线程

DataXceiver#run()：

```java
  public void run() {
    int opsProcessed = 0;
    Op op = null;

    try {
      ...// 一些初始化
      
      // 使用一个循环，以允许客户端发送新的操作请求时重用TCP连接
      do {
        updateCurrentThreadName("Waiting for operation #" + (opsProcessed + 1));

        try {
          ...// 超时设置
          op = readOp();
        } catch (InterruptedIOException ignored) {
          // Time out while we wait for client rpc
          break;
        } catch (IOException err) {
          ...// 此处的优化使得正常处理完一个操作后，一定会抛出EOFException或ClosedChannelException，可以退出循环
          ...// 如果是其他异常，则说明出现错误，重新抛出以退出循环
        }

        ...// 超时设置

        opStartTime = now();
        processOp(op);
        ++opsProcessed;
      } while ((peer != null) &&
          (!peer.isClosed() && dnConf.socketKeepaliveTimeout > 0));
    } catch (Throwable t) {
      ...// 异常处理
    } finally {
      ...// 资源清理，包括打开的文件、socket等
    }
  }
```

此处的优化不多讲。

DataXceiver#readOp()继承自Receiver类：从客户端发来的socket中读取op码，判断客户端要进行何种操作操作。写数据块使用的op码为80，返回的枚举变量`op = Op.WRITE_BLOCK`。

DataXceiver#processOp()也继承自Receiver类：

```java
  protected final void processOp(Op op) throws IOException {
    switch(op) {
    case READ_BLOCK:
      opReadBlock();
      break;
    case WRITE_BLOCK:
      opWriteBlock(in);
      break;
    ...// 其他case
    default:
      throw new IOException("Unknown op " + op + " in data stream");
    }
  }
  
  ...
  
  private void opWriteBlock(DataInputStream in) throws IOException {
    final OpWriteBlockProto proto = OpWriteBlockProto.parseFrom(vintPrefixed(in));
    final DatanodeInfo[] targets = PBHelper.convert(proto.getTargetsList());
    TraceScope traceScope = continueTraceSpan(proto.getHeader(),
        proto.getClass().getSimpleName());
    try {
      writeBlock(PBHelper.convert(proto.getHeader().getBaseHeader().getBlock()),
          PBHelper.convertStorageType(proto.getStorageType()),
          PBHelper.convert(proto.getHeader().getBaseHeader().getToken()),
          proto.getHeader().getClientName(),
          targets,
          PBHelper.convertStorageTypes(proto.getTargetStorageTypesList(), targets.length),
          PBHelper.convert(proto.getSource()),
          fromProto(proto.getStage()),
          proto.getPipelineSize(),
          proto.getMinBytesRcvd(), proto.getMaxBytesRcvd(),
          proto.getLatestGenerationStamp(),
          fromProto(proto.getRequestedChecksum()),
          (proto.hasCachingStrategy() ?
              getCachingStrategy(proto.getCachingStrategy()) :
            CachingStrategy.newDefaultStrategy()),
            (proto.hasAllowLazyPersist() ? proto.getAllowLazyPersist() : false));
     } finally {
      if (traceScope != null) traceScope.close();
     }
  }
```

>HDFS 2.x相对于1.x的另一项改进，在流式接口中也大幅替换为使用protobuf，不再是裸TCP分析字节流了。

Receiver类实现了DataTransferProtocol接口，但没有实现DataTransferProtocol#writeBlock()。多态特性告诉我们，这里会调用DataXceiver#writeBlock()。

终于回到了DataXceiver#writeBlock()：

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

      ...// 下游节点的处理。一个datanode是没有下游节点的。
      
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
      LOG.info("opWriteBlock " + block + " received exception " + ioe);
      throw ioe;
    } finally {
      ...// 清理资源
    }

    ...// 更新metrics
  }
```

特别说明几个参数：

* stage：表示数据块构建的状态。此处为`BlockConstructionStage.PIPELINE_SETUP_CREATE`。
* isDatanode：表示写数据块请求是否由数据节点发起。如果写请求中clientname为空，就说明是由数据节点发起（如数据块复制等由数据节点发起）。此处为false。
* isClient：表示写数据块请求是否由客户端发起，此值一定与isDatanode相反。此处为true。
* isTransfers：表示写数据块请求是否为数据块复制。如果stage为`BlockConstructionStage.TRANSFER_RBW`或`BlockConstructionStage.TRANSFER_FINALIZED`，则表示为了数据块复制。此处为false。

下面讨论“准备接收数据块”和“接收数据块”两个过程。 

## 准备接收数据块：`BlockReceiver.<init>()`

`BlockReceiver.<init>()`：

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
      ...// IOE通常涉及文件等资源，因此要额外清理资源
    }
  }
```

尽管上述代码的注释加了不少，但创建block的场景比较简单，只需要记住**在rbw目录下创建block文件和meta文件**即可。

>在rbw目录下创建数据块后，还要通过DataNode#notifyNamenodeReceivingBlock()向namenode汇报正在接收的数据块。该方法仅仅将数据块放入缓冲区中，由BPServiceActor线程异步汇报。
>
>此处不展开，后面会介绍一个相似的方法DataNode#notifyNamenodeReceivedBlock()。

## 接收数据块：BlockReceiver#receiveBlock()

BlockReceiver#receiveBlock()：

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

### 同步接收packet：BlockReceiver#receivePacket()

先看BlockReceiver#receivePacket()。

严格来说，BlockReceiver#receivePacket()负责**接收上游的packet，并继续向下游节点管道写**：

```java
  private int receivePacket() throws IOException {
    // read the next packet
    packetReceiver.receiveNextPacket(in);

    PacketHeader header = packetReceiver.getHeader();
    ...// 略

    ...// 检查packet头

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

    ...// 管道写相关
    
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

      ...// 如果是管道中的最后一个节点，则持久化之前，要先对收到的packet做一次校验（使用packet本身的校验机制）
      ...// 如果校验错误，则委托PacketResponder线程返回 ERROR_CHECKSUM 的ack

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
        datanode.checkDiskErrorAsync();
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
```

>BlockReceiver#shouldVerifyChecksum()主要与管道写有关，本文只有一个datanode，则一定满足`mirrorOut == null`。

上述代码看起来长，主要工作只有四项：

1. 接收packet
2. 校验packet
3. 持久化packet
4. 委托PacketResponder线程发送ack

BlockReceiver#receivePacket() + PacketResponder线程 + PacketResponder#ackQueue构成一个生产者消费者模型。生产和消费的对象是ack，BlockReceiver#receivePacket()是生产者，PacketResponder线程是消费者。

扫一眼PacketResponder#enqueue()：

```java
    void enqueue(final long seqno, final boolean lastPacketInBlock,
        final long offsetInBlock, final Status ackStatus) {
      final Packet p = new Packet(seqno, lastPacketInBlock, offsetInBlock,
          System.nanoTime(), ackStatus);
      if(LOG.isDebugEnabled()) {
        LOG.debug(myString + ": enqueue " + p);
      }
      synchronized(ackQueue) {
        if (running) {
          ackQueue.addLast(p);
          ackQueue.notifyAll();
        }
      }
    }
```

>ackQueue是一个线程不安全的LinkedList。
>
>关于如何利用线程不安全的容器实现生产者消费者模型可参考[Java实现生产者-消费者模型](/2017/10/08/Java实现生产者-消费者模型/)中的实现三。

### 异步发送ack：PacketResponder线程

与BlockReceiver#receivePacket()相对，PacketResponder线程负责**接收下游节点的ack，并继续向上游管道响应**。

PacketResponder#run()：

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
              ...// OOB相关（暂时忽略）
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
            ...// 异常处理
          } catch (IOException ioe) {
            ...// 异常处理
          }

          ...// 中断退出

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
          ...// 异常处理
        } catch (Throwable e) {
          ...// 异常处理
        }
      }
      LOG.info(myString + " terminating");
    }
```

总结起来，PacketResponder线程的核心工作如下：

1. 接收下游节点的ack
2. 比较ack.seqno与当前队头的pkt.seqno
3. 如果相等，则向上游发送pkt
4. 如果是最后一个packet，将block的状态转换为FINALIZED

>一不小心把管道响应的逻辑也分析了。。。

扫一眼PacketResponder线程使用的出队和查看对头的方法：

```java
    // 查看队头
    Packet waitForAckHead(long seqno) throws InterruptedException {
      synchronized(ackQueue) {
        while (isRunning() && ackQueue.size() == 0) {
          if (LOG.isDebugEnabled()) {
            LOG.debug(myString + ": seqno=" + seqno +
                      " waiting for local datanode to finish write.");
          }
          ackQueue.wait();
        }
        return isRunning() ? ackQueue.getFirst() : null;
      }
    }
    
    ...
    
    // 出队
    private void removeAckHead() {
      synchronized(ackQueue) {
        ackQueue.removeFirst();
        ackQueue.notifyAll();
      }
    }
```

>队尾入队，队头出队。
>
>* 每次查看对头后，如果发现队列非空，则只要不出队，则队列后续状态一定是非空的，且队头元素不变。
>* 查看队头后的第一次出队，弹出的一定是刚才查看队头看到的元素。
>

需要看下PacketResponder#finalizeBlock()：

```java
    private void finalizeBlock(long startTime) throws IOException {
      // 关闭BlockReceiver，并清理资源
      BlockReceiver.this.close();
      ...// log
      block.setNumBytes(replicaInfo.getNumBytes());
      // datanode上的数据块关闭委托给FsDatasetImpl#finalizeBlock()
      datanode.data.finalizeBlock(block);
      // namenode上的数据块关闭委托给Datanode#closeBlock()
      datanode.closeBlock(
          block, DataNode.EMPTY_DEL_HINT, replicaInfo.getStorageUuid());
      ...// log
    }
```

#### datanode角度的数据块关闭：FsDatasetImpl#finalizeBlock()

FsDatasetImpl#finalizeBlock()：

```java
  public synchronized void finalizeBlock(ExtendedBlock b) throws IOException {
    if (Thread.interrupted()) {
      // Don't allow data modifications from interrupted threads
      throw new IOException("Cannot finalize block from Interrupted Thread");
    }
    ReplicaInfo replicaInfo = getReplicaInfo(b);
    if (replicaInfo.getState() == ReplicaState.FINALIZED) {
      // this is legal, when recovery happens on a file that has
      // been opened for append but never modified
      return;
    }
    finalizeReplica(b.getBlockPoolId(), replicaInfo);
  }
  
  ...
  
  private synchronized FinalizedReplica finalizeReplica(String bpid,
      ReplicaInfo replicaInfo) throws IOException {
    FinalizedReplica newReplicaInfo = null;
    if (replicaInfo.getState() == ReplicaState.RUR &&
       ((ReplicaUnderRecovery)replicaInfo).getOriginalReplica().getState() == 
         ReplicaState.FINALIZED) {  // 数据块恢复相关（略）
      newReplicaInfo = (FinalizedReplica)
             ((ReplicaUnderRecovery)replicaInfo).getOriginalReplica();
    } else {
      FsVolumeImpl v = (FsVolumeImpl)replicaInfo.getVolume();
      // 回忆BlockReceiver.<init>()的分析，我们创建的block处于RBW状态，block文件位于rbw目录（当然，实际上位于哪里也无所谓，原因见后）
      File f = replicaInfo.getBlockFile();
      if (v == null) {
        throw new IOException("No volume for temporary file " + f + 
            " for block " + replicaInfo);
      }

      // 在卷FsVolumeImpl上进行block文件与meta文件的状态转换
      File dest = v.addFinalizedBlock(
          bpid, replicaInfo, f, replicaInfo.getBytesReserved());
      // 该副本即代表最终的数据块副本，处于FINALIZED状态
      newReplicaInfo = new FinalizedReplica(replicaInfo, v, dest.getParentFile());

      ...// 略
    }
    volumeMap.add(bpid, newReplicaInfo);

    return newReplicaInfo;
  }
```

FsVolumeImpl#addFinalizedBlock()：

```java
  File addFinalizedBlock(String bpid, Block b,
                         File f, long bytesReservedForRbw)
      throws IOException {
    releaseReservedSpace(bytesReservedForRbw);
    return getBlockPoolSlice(bpid).addBlock(b, f);
  }
```

还记得datanode启动过程中分析的FsVolumeImpl与BlockPoolSlice的关系吗？此处将操作继续委托给BlockPoolSlice#addBlock()：

>可知，**BlockPoolSlice仅管理处于FINALIZED的数据块**。

```java
  File addBlock(Block b, File f) throws IOException {
    File blockDir = DatanodeUtil.idToBlockDir(finalizedDir, b.getBlockId());
    if (!blockDir.exists()) {
      if (!blockDir.mkdirs()) {
        throw new IOException("Failed to mkdirs " + blockDir);
      }
    }
    File blockFile = FsDatasetImpl.moveBlockFiles(b, f, blockDir);
    ...// 统计相关
    return blockFile;
  }
```

BlockPoolSlice反向借助FsDatasetImpl提供的静态方法FsDatasetImpl.moveBlockFiles()：

```java
  static File moveBlockFiles(Block b, File srcfile, File destdir)
      throws IOException {
    final File dstfile = new File(destdir, b.getBlockName());
    final File srcmeta = FsDatasetUtil.getMetaFile(srcfile, b.getGenerationStamp());
    final File dstmeta = FsDatasetUtil.getMetaFile(dstfile, b.getGenerationStamp());
    try {
      NativeIO.renameTo(srcmeta, dstmeta);
    } catch (IOException e) {
      throw new IOException("Failed to move meta file for " + b
          + " from " + srcmeta + " to " + dstmeta, e);
    }
    try {
      NativeIO.renameTo(srcfile, dstfile);
    } catch (IOException e) {
      throw new IOException("Failed to move block file for " + b
          + " from " + srcfile + " to " + dstfile.getAbsolutePath(), e);
    }
    ...// 日志
    return dstfile;
  }
```

直接将block文件和meta文件从原目录（rbw目录，对应RBW状态）移动到finalized目录（对应FINALIZED状态）。

至此，datanode上的写数据块已经完成。

不过，namenode上的元信息还没有更新，因此，还要向namenode汇报收到了数据块。

>* 线程安全由FsDatasetImpl#finalizeReplica()保证
>* 整个FsDatasetImpl#finalizeReplica()的流程中，都不关系数据块的原位置，状态转换逻辑本身保证了其正确性。

#### namenode角度的数据块关闭：Datanode#closeBlock()

Datanode#closeBlock()：

```java
  void closeBlock(ExtendedBlock block, String delHint, String storageUuid) {
    metrics.incrBlocksWritten();
    BPOfferService bpos = blockPoolManager.get(block.getBlockPoolId());
    if(bpos != null) {
      // 向namenode汇报已收到的数据块
      bpos.notifyNamenodeReceivedBlock(block, delHint, storageUuid);
    } else {
      LOG.warn("Cannot find BPOfferService for reporting block received for bpid="
          + block.getBlockPoolId());
    }
    // 将新数据块添加到blockScanner的扫描范围中（暂不讨论）
    FsVolumeSpi volume = getFSDataset().getVolume(block);
    if (blockScanner != null && !volume.isTransientStorage()) {
      blockScanner.addBlock(block);
    }
  }
```

BPOfferService#notifyNamenodeReceivedBlock()：

```java
  void notifyNamenodeReceivedBlock(
      ExtendedBlock block, String delHint, String storageUuid) {
    checkBlock(block);
    // 收到数据块（增加）与删除数据块（减少）是一起汇报的，都构造为ReceivedDeletedBlockInfo
    ReceivedDeletedBlockInfo bInfo = new ReceivedDeletedBlockInfo(
        block.getLocalBlock(),
        ReceivedDeletedBlockInfo.BlockStatus.RECEIVED_BLOCK,
        delHint);

    // 每个BPServiceActor都要向自己负责的namenode发送报告
    for (BPServiceActor actor : bpServices) {
      actor.notifyNamenodeBlock(bInfo, storageUuid, true);
    }
  }
```

BPServiceActor#notifyNamenodeBlock()：

```java
  void notifyNamenodeBlock(ReceivedDeletedBlockInfo bInfo,
      String storageUuid, boolean now) {
    synchronized (pendingIncrementalBRperStorage) {
      // 更新pendingIncrementalBRperStorage
      addPendingReplicationBlockInfo(
          bInfo, dn.getFSDataset().getStorage(storageUuid));
      // sendImmediateIBR是一个volatile变量，控制是否立即发送BlockReport（BR）
      sendImmediateIBR = true;
      // 传入的now为true，接下来将唤醒阻塞在pendingIncrementalBRperStorage上的所有线程
      if (now) {
        pendingIncrementalBRperStorage.notifyAll();
      }
    }
  }
```

该方法的核心是**pendingIncrementalBRperStorage，它维护了两次汇报之间收到、删除的数据块**。_pendingIncrementalBRperStorage是一个缓冲区，此处将收到的数据块放入缓冲区后即认为通知完成（当然，不一定成功）；由其他线程读取缓冲区，异步向namenode汇报_。

猴子看的源码比较少，但这种缓冲区的设计思想在HDFS和Yarn中非常常见。缓冲区实现了解耦，解耦不仅能提高可扩展性，还能在缓冲区两端使用不同的处理速度、处理规模。如pendingIncrementalBRperStorage，生产者不定期、零散放入的数据块，消费者就可以定期、批量的对数据块进行处理。而保障一定及时性的前提下，批量汇报减轻了RPC的压力。

>利用IDE，很容易得知，只有负责向各namenode发送心跳的BPServiceActor线程阻塞在pendingIncrementalBRperStorage上。后文将分析该线程如何进行实际的汇报。

### PacketResponder#close()

根据对BlockReceiver#receivePacket()与PacketResponder线程的分析，节点已接收所有packet时，ack可能还没有发送完。

因此，需要调用PacketResponder#close()，等待发送完所有ack后关闭responder：

```java
    public void close() {
      synchronized(ackQueue) {
        // ackQueue非空就说明ack还没有发送完成
        while (isRunning() && ackQueue.size() != 0) {
          try {
            ackQueue.wait();
          } catch (InterruptedException e) {
            running = false;
            Thread.currentThread().interrupt();
          }
        }
        if(LOG.isDebugEnabled()) {
          LOG.debug(myString + ": closing");
        }
        // notify阻塞在PacketResponder#waitForAckHead()方法上的PacketResponder线程，使其检测到关闭条件
        running = false;
        ackQueue.notifyAll();
      }

      // ???
      synchronized(this) {
        running = false;
        notifyAll();
      }
    }
```

>猴子没明白19-22行的synchronized语句块有什么用，，，求解释。

# BPServiceActor线程

根据前文，接下来需要分析BPServiceActor线程如何读取pendingIncrementalBRperStorage缓冲区，进行实际的汇报。

在BPServiceActor#offerService()中调用了pendingIncrementalBRperStorage#wait()。由于涉及阻塞、唤醒等操作，无法按照正常流程分析，这里从线程被唤醒的位置开始分析：

```java
        // 如果目前不需要汇报，则wait一段时间
        long waitTime = dnConf.heartBeatInterval - 
        (Time.now() - lastHeartbeat);
        synchronized(pendingIncrementalBRperStorage) {
          if (waitTime > 0 && !sendImmediateIBR) {
            try {
              // BPServiceActor线程从此处醒来，然后退出synchronized块
              pendingIncrementalBRperStorage.wait(waitTime);
            } catch (InterruptedException ie) {
              LOG.warn("BPOfferService for " + this + " interrupted");
            }
          }
        } // synchronized
```

>可能有读者阅读过猴子的[条件队列大法好：使用wait、notify和notifyAll的正确姿势](/2017/11/29/条件队列大法好：使用wait、notify和notifyAll的正确姿势/)，认为此处`if(){wait}`的写法姿势不正确。读者可再复习一下该文的“version2：过早唤醒”部分，结合HDFS的心跳机制，思考一下为什么此处的写法没有问题。更甚，此处恰恰应当这么写。

如果目前不需要汇报，则BPServiceActor线程会wait一段时间，正式这段wait的时间，让BPServiceActor#notifyNamenodeBlock()的唤醒产生了意义。

BPServiceActor线程唤醒后，醒来后，继续心跳循环：

```java
    while (shouldRun()) {
      try {
        final long startTime = now();
        if (startTime - lastHeartbeat >= dnConf.heartBeatInterval) {
```

假设还到达心跳发送间隔，则不执行if语句块。

此时，在BPServiceActor#notifyNamenodeBlock()方法中修改的volatile变量sendImmediateIBR就派上了用场：

```java
        // 检测到sendImmediateIBR为true，则立即汇报已收到和已删除的数据块
        if (sendImmediateIBR ||
            (startTime - lastDeletedReport > dnConf.deleteReportInterval)) {
          // 汇报已收到和已删除的数据块
          reportReceivedDeletedBlocks();
          // 更新lastDeletedReport
          lastDeletedReport = startTime;
        }

        // 再来一次完整的数据块汇报
        List<DatanodeCommand> cmds = blockReport();
        processCommand(cmds == null ? null : cmds.toArray(new DatanodeCommand[cmds.size()]));

        // 处理namenode返回的命令
        DatanodeCommand cmd = cacheReport();
        processCommand(new DatanodeCommand[]{ cmd });
```

>有意思的是，这里先单独汇报了一次数据块收到和删除的情况，该RPC不需要等待namenode的返回值；又汇报了一次总体情况，此时需要等待RPC的返回值了。
>
>因此，尽管对于增删数据块采取增量式汇报，但**由于增量式汇报后必然跟着一次全量汇报，使得增量汇报的成本仍然非常高**。为了提高并发，BPServiceActor#notifyNamenodeBlock修改缓冲区后立即返回，不关心汇报是否成功。也不必担心汇报失败的后果：在汇报之前，数据块已经转为FINALIZED状态+持久化到磁盘上+修改了缓冲区，如果汇报失败可以等待重试，如果datanode在发报告前挂了可以等启动后重新汇报，必然能保证一致性。

暂时不关心总体汇报的逻辑，只看单独汇报的BPServiceActor#reportReceivedDeletedBlocks()：

```java
  private void reportReceivedDeletedBlocks() throws IOException {

    // 构造报告，并重置sendImmediateIBR为false
    ArrayList<StorageReceivedDeletedBlocks> reports =
        new ArrayList<StorageReceivedDeletedBlocks>(pendingIncrementalBRperStorage.size());
    synchronized (pendingIncrementalBRperStorage) {
      for (Map.Entry<DatanodeStorage, PerStoragePendingIncrementalBR> entry :
           pendingIncrementalBRperStorage.entrySet()) {
        final DatanodeStorage storage = entry.getKey();
        final PerStoragePendingIncrementalBR perStorageMap = entry.getValue();

        if (perStorageMap.getBlockInfoCount() > 0) {
          ReceivedDeletedBlockInfo[] rdbi = perStorageMap.dequeueBlockInfos();
          reports.add(new StorageReceivedDeletedBlocks(storage, rdbi));
        }
      }
      sendImmediateIBR = false;
    }

    // 如果报告为空，就直接返回
    if (reports.size() == 0) {
      return;
    }

    // 否则通过RPC向自己负责的namenode发送报告
    boolean success = false;
    try {
      bpNamenode.blockReceivedAndDeleted(bpRegistration,
          bpos.getBlockPoolId(),
          reports.toArray(new StorageReceivedDeletedBlocks[reports.size()]));
      success = true;
    } finally {
      // 如果汇报失败，则将增删数据块的信息放回缓冲区，等待重新汇报
      if (!success) {
        synchronized (pendingIncrementalBRperStorage) {
          for (StorageReceivedDeletedBlocks report : reports) {
            PerStoragePendingIncrementalBR perStorageMap =
                pendingIncrementalBRperStorage.get(report.getStorage());
            perStorageMap.putMissingBlockInfos(report.getBlocks());
            sendImmediateIBR = true;
          }
        }
      }
    }
  }
```

有两个注意点：

* 不管namenode处于active或standy状态，BPServiceActor线程都会汇报（尽管会忽略standby namenode的命令）
* 最后success为false时，可能namenode已收到汇报，但将信息添加会缓冲区导致重复汇报也没有坏影响，这分为两个方面：
    * 重复汇报已删除的数据块：namenode发现未存储该数据块的信息，则得知其已经删除了，会忽略该信息。
    * 重复汇报已收到的数据块：namenode发现新收到的数据块与已存储数据块的信息完全一致，也会忽略该信息。

# 总结

1个客户端+1个datanode构成了最小的管道。本文梳理了在这个最小管道上无异常情况下的写数据块过程，在此之上，再来分析管道写的有异常的难度将大大降低。
