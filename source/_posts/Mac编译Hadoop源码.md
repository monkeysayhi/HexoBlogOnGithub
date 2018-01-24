---
title: Mac编译Hadoop源码
tags:
  - Hadoop
  - Mac
  - 原创
reward: true
date: 2018-01-15 16:06:49
---

单纯看书翻源码非常枯燥，为了单步debug追Hadoop源码，最好先在部署环境编译一份源码，以避免各种环境问题。

本文记录了猴子在自己的Mac上编译Hadoop源码的过程，结合之前的一次编译经验，基本覆盖了编译Hadoop源码时可能遇到的主要问题。

<!--more-->

>随着debug的进一步深入，后期可能涉及到对源码的修改，需要多次重新编译。因此，这一关是绕不过的。

# 版本声明

* 源码：`Apache Hadoop 2.6.0`
* 系统：`macOS 10.12.4`
* 依赖：
    * `oracle jdk 1.7.0_79`
    * `Apache Maven 3.5.0`
    * `libprotoc 2.5.0`

# 编译

## 核心命令

```bash
mvn install -DskipTests -Pdist,native -Dtar
```

## 坑

### 时间长

Hadoop源码量巨大、依赖众多，编译时间比较长。

下载jar包和编译protoc是两个大头。编译protoc用了1小时左右，下载jar包+编译Hadoop用了2个多小时。除去这些时间，也需要1小时左右才能编译成功。

还好上半年为了看Yarn的状态机编译过一回，虽然是不完全编译，但也下载了大部分依赖的jar包，并编译安装了protoc（强烈建议编译安装，忘记当时有什么坑来着）。这次只需要继续踩上次剩下的坑了。

>不过，鉴于第一次编译时，大部分人都会重复多次才能编译成功，单次编译的时间也没什么意义了。喝杯茶，慢慢来吧。

### JDK版本

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:2.5.1:compile (default-compile) on project hadoop-annotations: Compilation failure: Compilation failure:
[ERROR] /Volumes/Extended/Users/msh/IdealProjects/Git-Study/hadoop/hadoop-common-project/hadoop-annotations/src/main/java/org/apache/hadoop/classification/tools/ExcludePrivateAnnotationsJDiffDoclet.java:[20,22] 错误: 程序包com.sun.javadoc不存在
```

不明白为啥这个包会不存在，可能是JDK版本问题。google一番，参考[解决Mac OS 下编译Hadoop Annotations 程序包com.sun.javadoc找不到问题](http://blog.csdn.net/starshine/article/details/70229697)解决。

#### 验证

在所有的pom.xml里面找设置1.7 jdk的地方：

```bash
find . -name pom.xml > tmp/tmp.txt

while read file
do
    cnt=0
    grep '1.7' $file -C2 | while read line; do
        if [ -n "$line" ]; then
            if [ $cnt -eq 0 ]; then
                echo "+++file: $file"
            fi
            cnt=$((cnt+1))
            echo $line
        fi
    done
    cnt=0
done < tmp/tmp.txt
```

输出：

```
+++file: ./hadoop-common-project/hadoop-annotations/pom.xml
</profile>
<profile>
<id>jdk1.7</id>
<activation>
--
<activation>
<jdk>1.7</jdk>
</activation>
<dependencies>
--
--
<groupId>jdk.tools</groupId>
<artifactId>jdk.tools</artifactId>
<version>1.7</version>
<scope>system</scope>
<systemPath>${java.home}/../lib/tools.jar</systemPath>
+++file: ./hadoop-project/pom.xml
...（略）
```

确实`./hadoop-common-project/hadoop-annotations/pom.xml`中限制了jdk版本。

#### 解决

猴子Mac上的默认JDK是`oracle jdk1.8.0_102`的，翻了下jdk源码也有这个包。说明不是因为该包实际不存在。

可以尝试修改pom里限制的jdk版本；不过，为了防止使用了deprecated方法等麻烦，这里直接切jdk 1.7，不改pom。

### openssl环境变量

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-antrun-plugin:1.7:run (make) on project hadoop-pipes: An Ant BuildException has occured: exec returned: 1
[ERROR] around Ant part ...<exec dir="/Volumes/Extended/Users/msh/IdealProjects/Git-Study/hadoop/hadoop-tools/hadoop-pipes/target/native" executable="cmake" failonerror="true">... @ 5:153 in /Volumes/Extended/Users/msh/IdealProjects/Git-Study/hadoop/hadoop-tools/hadoop-pipes/target/antrun/build-main.xml
```

猜测是ant版本问题，重装了jdk1.7适配的ant。

>`Ant BuildException`也是够迷惑的。而且之前猴子电脑配置的jdk1.8，切到1.7之后ant就不能用了（brew安装的ant用1.8jdk编译的，1.7无法解析class文件），重装了适配1.7的ant版本后，ant可以正常使用了，却还是报这个错。。。

结果还是报这个错，打开build-main.xml看，发现是一个cmake命令的配置，copy到终端执行：

```bash
cmake /Volumes/Extended/Users/msh/IdealProjects/Git-Study/hadoop/hadoop-tools/hadoop-pipes/src/ -DJVM_ARCH_DATA_MODEL=64
```

输出：

```
...（略）
CommandLineTools/usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
CMake Error at /usr/local/Cellar/cmake/3.6.2/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:148 (message):
  Could NOT find OpenSSL, try to set the path to OpenSSL root folder in the
  system variable OPENSSL_ROOT_DIR (missing: OPENSSL_INCLUDE_DIR)
Call Stack (most recent call first):
  /usr/local/Cellar/cmake/3.6.2/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:388 (_FPHSA_FAILURE_MESSAGE)
  /usr/local/Cellar/cmake/3.6.2/share/cmake/Modules/FindOpenSSL.cmake:380 (find_package_handle_standard_args)
  CMakeLists.txt:20 (find_package)

...（略）
```

`OPENSSL_ROOT_DIR`、`OPENSSL_INCLUDE_DIR`没有设置。echo一下确实没有设置。

#### 解决

Mac自带OpenSSL，然而猴子并不知道哪里算是root，哪里算是include；另外，据说mac计划移除默认的openssl。干脆自己重新安装：

```bash
brew install openssl
```

然后配置环境变量：

```bash
export OPENSSL_ROOT_DIR=/usr/local/Cellar/openssl/1.0.2n
export OPENSSL_INCLUDE_DIR=$OPENSSL_ROOT_DIR/include
```

### maven仓库不稳定

```
[ERROR] Failed to execute goal on project hadoop-aws: Could not resolve dependencies for project org.apache.hadoop:hadoop-aws:jar:2.6.0: Could not transfer artifact com.amazonaws:aws-java-sdk:jar:1.7.4 from/to central (https://repo.maven.apache.org/maven2): GET request of: com/amazonaws/aws-java-sdk/1.7.4/aws-java-sdk-1.7.4.jar from central failed: SSL peer shut down incorrectly -> [Help 1]
```

出现类似“Could not resolve dependencies”、“SSL peer shut down incorrectly”等语句，一般是maven不稳定，换个稳定的maven源，或者重新编译多试几次。

### 其他

#### 历史遗留坑

上次编译有个小坑，是Hadoop源码里的历史遗留问题。

编译过程中会在`$JAVA_HOME/Classes`下找一个并不存在的jar包`classes.jar`，实际上需要的是`$JAVA_HOME/lib/tools.jar`，加个软链就好（注意mac加软链时与linux的区别）。

因此上次编译猴子已经修复了这个问题，这里就不复现了。具体可以看这篇[mac下编译Hadoop](http://bigdatadecode.club/mac%E4%B8%8B%E7%BC%96%E8%AF%91Hadoop.html)

#### 没有skipTests

没有skipTests的话，至少会在测试过程中以下错误：

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-surefire-plugin:2.16:test (default-test) on project hadoop-auth: There are test failures.
[ERROR]
[ERROR] Please refer to /Volumes/Extended/Users/msh/IdealProjects/Git-Study/hadoop/hadoop-common-project/hadoop-auth/target/surefire-reports for the individual test results.

Results :

Tests in error:
  TestKerberosAuthenticator.testAuthenticationHttpClientPost:157 » ClientProtocol
  TestKerberosAuthenticator.testAuthenticationHttpClientPost:157 » ClientProtocol

Tests run: 92, Failures: 0, Errors: 2, Skipped: 0
```

可以暂时忽略这些相关错误，skipTests跳过测试，能追踪源码了解主要过程即可。

# 启动伪分布式“集群”

编译成功后，在`hadoop-dist`模块的`target`目录下，生成了各种发行版。选择`hadoop-2.6.0.tar.gz`，找个地方解压。

## 配置

参考[官网](http://hadoop.apache.org/docs/r2.6.5/hadoop-project-dist/hadoop-common/SingleCluster.html#Pseudo-Distributed_Operation)配置`etc/hadoop`目录下的`core-site.xml`、`hdfs-site.xml`、`yarn-site.xml`、`mapred-site.xml`等。

## 启动

* 格式化（只有第一次需要格式化）：

```bash
bin/hdfs namenode -format
```

* 启动（启动dfs时需要输出几次用户密码）

```bash
# 启动 Namenode、SecondaryNamenode（伪分布式无法开启HA）、Datanode
sbin/start-dfs.sh
# 启动 ResourceManager、NodeManager
sbin/start-yarn.sh
# 启动 TimelineServer（ApplicationHistoryServer）
sbin/yarn-daemon.sh start timelineserver
```

启动后，访问NameNode、ResourceManager的Web UI，Timeline的RESTful接口，创建目录、上传文件，跑示例MapReduce，以验证是否成功部署。

---

>参考：
>
>* [HowToSetupYourDevelopmentEnvironment](https://wiki.apache.org/hadoop/HowToSetupYourDevelopmentEnvironment)
>
