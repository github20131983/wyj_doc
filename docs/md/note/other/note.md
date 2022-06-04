## Linux常用命令

1. 查看占用了8080端口号的进程`lsof -i:8080`

2. 查看当前在哪个目录下`pwd`

3. `chmod a+x `文件 赋予文件可执行权限

4. 文件追加`cat 1.txt>>2.tx`

5. 文件覆盖`cat 1.txt>2.txt`

6. 文件清空`cat /dev/null>2.txt `

7. 查看某个关键字前几行后几行cat filename|grep '关键字' -A4(后四行) -B4(前四行)

8. 查看已删除空间却没有释放的进程`lsof -n / |grep deleted `

9. 在标准unix/linux下的grep命令中，通过以下参数控制上下文的显示

     grep -C 10 keyword catalina.out 显示file文件中匹配keyword字串那行以及上下10行

     grep -B 10 keyword catalina.out 显示keyword及前10行

     grep -A 10 keyword catalina.out 显示keyword及后10行
     
     ```shell
     #!/usr/bin/env bash
     
     function init() {
         APP_KEY="com.sankuai.scoai.pallas"
     
         if [ -z "$LOG_PATH" ]; then  #如果LOG_PATH长度为0，拼接LOG_PATH
             LOG_PATH="/opt/logs/$APP_KEY"
         fi
     
         mkdir -p $LOG_PATH  #建立目录
     
         if [ -z "$WORK_PATH" ]; then  #如果WORK_PATH长度为0，拼接WORK_PATH
             WORK_PATH="/opt/meituan/$APP_KEY"
         fi
     
         JAVA_CMD="java"
         if ! command -v $JAVA_CMD >/dev/null 2>&1; then #如果不支持直接执行java命令，拼接JAVA_CMD
             JAVA_CMD="/usr/local/$JAVA_CMD/bin/java"
         fi
     
         JVM_ARGS="-server -Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8 -Djava.net.preferIPv6Addresses=false"
         #-server 服务器模式
         #-Dfile.encoding java文件编码
         #-Dsun.jnu.encoding 操作系统默认编码
     
         if [ -z "$JVM_GC" ]; then
             JVM_GC="-XX:+UseG1GC -XX:G1HeapRegionSize=4M -XX:InitiatingHeapOccupancyPercent=40 -XX:MaxGCPauseMillis=100 -XX:+TieredCompilation -XX:CICompilerCount=4 -XX:-UseBiasedLocking -Xlog:gc*:$LOG_PATH/gc.log:time,uptime:filecount=20,filesize=50M"
         #-XX:+UseG1GC 使用G1垃圾回收器
         #-XX:G1HeapRegionSize=4M 堆内存中一个Region的大小可以通过-XX:G1HeapRegionSize参数指定，大小区间只能是1M、2M、4M、8M、16M和32M
         #-XX:InitiatingHeapOccupancyPercent 当老年代大小占整个堆大小百分比达到该阈值时，会触发一次mixed gc(当越来越多的对象晋升到老年代old region时，为了避免堆内存被耗尽，虚拟机会触发一个混合的垃圾收集器，即mixed gc，该算法并不是一个old gc，除了回收整个young region，还会回收一部分的old region，这里需要注意：是一部分老年代，而不是全部老年代，可以选择哪些old region进行收集，从而可以对垃圾回收的耗时时间进行控制
         #-XX:MaxGCPauseMillis=100 	设置G1收集过程目标时间
         #-XX:+TieredCompilation  这个参数主要用于是否开启JVM的分层编译
         #-XX:CICompilerCount 最大并行编译数
         #-XX:-UseBiasedLocking 不使用偏向锁
         #--Xlog:gc*:$LOG_PATH/gc.log:time,uptime:filecount=20,filesize=50M 指明GC日志存放位置[格式采用时间：到现在系统运行时间]，保留20个文件，每50M就轮换
     
         fi
     
         if [ -z "$JVM_EXT_ARGS" ]; then
             JVM_EXT_ARGS=""
         fi
     
         if [ -z "$JVM_HEAP" ]; then
             JVM_HEAP=`getJVMMemSizeOpt`
         fi
     }
     
     
     function run() {
         EXEC="exec"
         CONTEXT=/
         EXEC_JAVA="$EXEC $JAVA_CMD $JVM_ARGS $JVM_EXT_ARGS $JVM_HEAP $JVM_GC \
         -XX:ErrorFile=$LOG_PATH/vmerr.log \
         -XX:HeapDumpPath=$LOG_PATH/HeapDump"
         # -XX:ErrorFile系统奔溃时的日志
         # -XX:HeapDumpPath 堆快照路径
     
         if [ "$UID" = "0" ]; then
             ulimit -n 1024000 # 修改最大连接数
             umask 000 # 去掉权限
         else
             echo $EXEC_JAVA 
         fi
         cd $WORK_PATH
         pwd
         targetPackage=`find . -maxdepth 1 -type f \( -name "*.jar" -o -name "*.war" \)`
         #找到当前文件夹中的深度为1的类型为文件的name为jar或war的
         
         env=`getEnv`
         SPRING_ENV=""
         if [ -n "$env" ]; then
             SPRING_ENV="--spring.profiles.active=$env"
         fi
         
     	echo "this target jar will be executed: "$targetPackage
         $EXEC_JAVA -jar $targetPackage $SPRING_ENV 2>&1
     }
     
     function getTotalMemSizeMb() {
     	memsizeKb=`cat /proc/meminfo|grep MemTotal|awk '{print $2}'`
         if [ -z "$memsizeKb" ]; then
             memsizeKb=8*1000*1000
         fi
     	memsizeMb=$(( $memsizeKb/1024 ))
     	echo $memsizeMb
     }
     
     function outputJvmArgs() {
     	jvmSize=$1
     	MaxMetaspaceSize=$2
     	ReservedCodeCacheSize=$3
     	echo "-Xss512k -Xmx"$jvmSize" -Xms"$jvmSize" -XX:MetaspaceSize="$MaxMetaspaceSize" -XX:MaxMetaspaceSize="$MaxMetaspaceSize" -XX:+AlwaysPreTouch -XX:ReservedCodeCacheSize="$ReservedCodeCacheSize" -XX:+HeapDumpOnOutOfMemoryError "
     }
     # -Xss设置每个线程的堆栈大小
     # -Xmx JVM最大可用内存
     # -Xms JVM初始内存
     # -XX:MetaspaceSize  设置metaspace区域的最大值
     # -XX:+AlwaysPreTouch 启动的时候真实的分配物理内存给jvm
     # -XX:ReservedCodeCacheSize 设置Code Cache大小，JIT编译的代码都放在Code Cache中，若Code Cache空间不足则JIT无法继续编译，并且会去优化，比如编译执行改为解释执行，由此，性能会降低
     # -XX:+HeapDumpOnOutOfMemoryError 当堆内存空间溢出时输出堆的内存快照
     
     
     function getJVMMemSizeOpt() {
     	memsizeMb=`getTotalMemSizeMb`
     
     	#公司的机器内存比实际标的数字要小，比如8G实际是7900M左右，一般误差小于1G
     	#内存分级，单位兆/M
     	let maxSize_lvl1=63*1024
     	let maxSize_lvl2=31*1024
     	let maxSize_lvl3=21*1024
     	let maxSize_lvl4=15*1024
     	let maxSize_lvl5=7*1024
     	let maxSize_lvl6=3*1024
     	let maxSize_lvl7=1024
     	let maxSize_lvl8=512
     
     	if [[ $memsizeMb -gt $maxSize_lvl1 ]]
     	then
     		jvmSize="32g"
     		MaxMetaspaceSize="2g"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [[ $memsizeMb -gt $maxSize_lvl2 && $memsizeMb -le $maxSize_lvl1 ]]
     	then
     		jvmSize="24g"
     		MaxMetaspaceSize="1g"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [[ $memsizeMb -gt $maxSize_lvl3 && $memsizeMb -le $maxSize_lvl2 ]]
     	then
     		jvmSize="18g"
     		MaxMetaspaceSize="512m"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [[ $memsizeMb -gt $maxSize_lvl4 && $memsizeMb -le $maxSize_lvl3 ]]
     	then
     		jvmSize="12g"
     		MaxMetaspaceSize="512m"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [[ $memsizeMb -gt $maxSize_lvl5 && $memsizeMb -le $maxSize_lvl4 ]]
     	then
     		jvmSize="4g"
     		MaxMetaspaceSize="512m"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [[ $memsizeMb -gt $maxSize_lvl6 && $memsizeMb -le $maxSize_lvl5 ]]
     	then
     		jvmSize="2g"
     		MaxMetaspaceSize="256m"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [[ $memsizeMb -gt $maxSize_lvl7 && $memsizeMb -le $maxSize_lvl6 ]]
     	then
     		jvmSize="1g"
     		MaxMetaspaceSize="256m"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [[ $memsizeMb -gt $maxSize_lvl8 && $memsizeMb -le $maxSize_lvl7 ]]
     	then
     		jvmSize="512m"
     		MaxMetaspaceSize="256m"
     		ReservedCodeCacheSize="240m"
     	fi
     
     	if [ $memsizeMb -le $maxSize_lvl8 ]; then
     		echo "service start fail:not enough memory for MDP service"
     		exit 1
     	fi
     	outputJvmArgs $jvmSize $MaxMetaspaceSize $ReservedCodeCacheSize
     	exit 0
     }
     
     function getEnv(){
         FILE_NAME="/data/webapps/appenv"
         PROP_KEY="env"
         PROP_VALUE=""
         if [[ -f "$FILE_NAME" ]]; then
             PROP_VALUE=`cat ${FILE_NAME} | grep -w ${PROP_KEY} | cut -d'=' -f2`
         fi
         echo $PROP_VALUE
     }
     
     init
     run
     ```
     
     ```
     
     ```
     
     

## Hive

查询非聚类字段，可以使用`first()`或者`collect_set(template_name)[0]`来处理

## kafka

1、`brew install`有点问题，提示需要openjdk8

2、直接官网搞下来，解压到/usr/local

3、快速开始`https://kafka.apache.org/quickstart`

4、后台启动kafka,端口号9092

![kafka后台启动](./images/kafka后台启动.png)

```shell
bin/zookeeper-server-start.sh config/zookeeper.properties #启动zk

bin/kafka-server-start.sh config/server.properties #启动kafka服务端

bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092 #创建topic

bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092 #查看topic的描述

bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092#启动生产者

bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092#启动消费者

bin/kafka-topics.sh --zookeeper localhost:2181 --list#查看kafka所有的topic
kafka-topics.sh --zookeeper zookeeper:2181 --list#当在容器中时执行这一条

bin/kafka-topics.sh --zookeeper localhost:2181 --topic hotitems --describe#查看kafka某个topic的信息

bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list#查看消费者组

bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --delete --group {消费组}
#删除某个消费者

bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group console-consumer-68915  #查看某个消费者组的情况
GROUP   console-consumer-68915   #消费者组id            
TOPIC     hotitems      #消费组消费的主题
PARTITION  0						#分区
CURRENT-OFFSET  -				#当前偏移
LOG-END-OFFSET  11613   #下一条偏移
LAG         -           #
CONSUMER-ID  consumer-console-consumer-68915-1-7f757591-d038-45be-840e-9fd6433aa92d#消费者id                                                        
HOST         /127.0.0.1      #主机
CLIENT-ID			consumer-console-consumer-68915-1    #客户端id
```

```shell
#后台启动kafka
cd /usr/local/kafka_2.12-2.5.0
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
bin/kafka-server-start.sh -daemon config/server.properties

#关闭kafka
cd /usr/local/kafka_2.12-2.5.0
bin/kafka-server-stop.sh config/server.properties
bin/zookeeper-server-stop.sh
```



## Redis

redis在`/usr/local/bin`下启动`redis-server`即可,配置文件在`/usr/local/ect`下的`redis.conf`文件中

![redis连接](/Users/wyj/Desktop/note/开发笔记/images/redis连接.png)

后台运行时在`conf`配置文件中把守护模式改为`yes`即可

![redsi后台](/Users/wyj/Desktop/note/开发笔记/images/redis后台.png)

清空所有数据命令：flushall

删除某个key：

关闭redis 客户端发送shutdown

## zookeeper

在命令行输入`zkServer start`可以**启动**

在命令行输入`zkCli`可以**连接**

输入`skServer status`可以**查看状态**

输入`zkServer stop`可以**结束**

配置文件在`/usr/local/ect/zookeeper`

![zookeeper安装位置](/Users/wyj/Desktop/note/开发笔记/images/zk位置.png)

## node

1. 安装`brew install node`

2. `node -v` ，`npm -v` 验证安装是否成功

3. `npm config set registry https://registry.npm.taobao.org`

   配置后可通过下面方式来验证是否成功

​       `npm config get registry`

​       在` ~/.npmrc `加入下面内容，可以避免安装 node-sass 失败

​        `sass_binary_site=https://npm.taobao.org/mirrors/node-sass/`

​	   直接使用 vi ~/.npmrc

## Git

1. `git add .`  将文件交给git管理

2. `git commit -m "提交注释"`   提交本地

3. `git push origin  分支名称 `  推送远程

   <font color='red' size=5>怎么把已经推送到远程的多次提交合并为一次？</font>

   ` git rebase -i HEAD~3`将最近的三次合并为一次

   然后得到如下图：

   ![image-20201128120231220](./images/image-20201128120231220.png)

将9fa,b58的pick字段改为f，然后执行

`git rebase --continue`

`git push -f`

最后再合并到主分支上

也可以使用IDEA的图形化界面

<img src="/Users/wyj/Desktop/note/开发笔记/images/image-20210902161609635.png" alt="image-20210902161609635" style="zoom:50%;" />

然后把第一次以后的都改为fix，然后强制push

<font color='red' size=5>git怎么打标签?</font>

` git tag v1.0 dd18e8`将v1.0标签打到dd18e8分支上

` git push origin v1.0`推送标签

`git tag -d v1.0`删除本地标签

`git push origin --delete v1.0`删除远程标签

<font color='red' size=5>git全局设置账号与邮箱</font>

`git config --global user.name "git_ai_ivr"`

`git config --global user.email "git_ai_ivr@meituan.com"`

## Maven

```xml
dependency:tree -Dverbose -Dincludes=org.mybatis:mybatis
```

查看某个包是怎么引入的

![image-20201105110935296](./images/image-20201105110935296.png)

**dependenciesManagement**写在父pom里，只会标记版本号，子pom不会继承，子pom引入的时候不需要version，会循着父pom逐级往上找寻找到version。

- Maven的snapshot版本，主要是为了两方同时开发，不同于常规的版本，Maven 每次构建都会在远程仓库中检查新的快照。 版本如果下载到了本地，如果不更新版本号，是不会去远程仓库寻找新的jar包的，只会在本地找。

要想发布，需要在代码中加上

```xml
<distributionManagement>
        <repository>
            <id>meituan-nexus-releases</id>
            <name>Meituan Nexus Releases Repository</name>
            <url>http://maven.sankuai.com/nexus/content/repositories/releases/</url>
        </repository>

        <snapshotRepository>
            <id>meituan-nexus-snapshots</id>
            <name>Meituan Nexus Snapshots Repository</name>
            <url>http://maven.sankuai.com/nexus/content/repositories/snapshots/</url>
        </snapshotRepository>
</distributionManagement>
```



## jackson

忽略不要的字段

```java
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,false);
```

## java

<font color='red'> try中的语句执行错误后边的还会执行吗？</font>

```java
public static void main(String[] args) {
        //System.out.println(test.num);
        try {
            int e=1/0;
            System.out.println("hello1");
        }catch (Exception e){

        }
        System.out.println("hello2");
    }

输出：hello2
```

<font color='red'>怎么把一个对象转成json放到redis，再取回来反序列化</font>

```java
@Data
public class ExtractStatus {
    private final int code;
    private final String status;
    private final Integer count;
}
```

放上去：

```java
ObjectMapper mapper = new ObjectMapper();
String mapJakcson = mapper.writeValueAsString(new ExtractStatus(111,"success",1111));
squirrelService.putJson("123", mapJakcson);
```

取回来：

```java
ObjectMapper mapper=new ObjectMapper();
ExtractStatus status=mapper.readValue(squirrelService.get("123"), ExtractStatus.class);
```

返给前台:

```java
return ResultUtil.ok(status);
```

```
@NonNullApi//相当于给包中的所有类都加了这个注解,而且包文档的描述都可以写在这里
package com.meituan.ai.sco.prometheus.biz.inf.v1.command.processor;
//https://blog.csdn.net/Dh_Chao/article/details/79001016
//https://segmentfault.com/a/1190000018862320?utm_source=tag-newest
import org.springframework.lang.NonNullApi;//这个注解的意思是方法参数和返回值不为null
```

分片上传：

```java
 @Value("${markplat.partion:#{100}}")
 private int split;
val partition = Lists.partition(toCall, split);//电话号码切成块，100个1块
partion.forEach(it -> {
            try {
                final var optResult = mapper.writeValueAsString(it);
                markService.pushMarkPlatform();
            } catch (Exception e) {
                log.error("error in write result json:{}", e);
            }
        });
```

泛型与通配符

1. ? 表示通配符类型
2. <? extends T> 既然是extends，就是表示泛型参数类型的上界，说明参数的类型应该是T或者T的子类。
3. <? super T> 既然是super，表示的则是类型的下界，说明参数的类型应该是T类型的父类，一直到object。

```java
public class FanXing {
    static class Fruit {
    }

    static class Apple extends Fruit {
    }

    static class Orange extends Apple {
    }

    public static void main(String... args) throws Exception {
        List<? extends Fruit> list = new ArrayList<Orange>();
        list.add(new Orange());//报错，无法安全添加不确定类型的元素
        list.add(null);
        Fruit fruit = list.get(0);

        Orange orange = (Orange) list.get(0);

        List<? super Fruit> list1 = new ArrayList<>();
        list1.add(new Orange());
        Fruit fruit1 = list1.get(0);//报错，无法确定具体的返回类型
        Orange orange1 = (Orange) list1.get(0);
    }
}
```



```java
/**
 * @author tony.zhuby
 */
public abstract class BaseWithCodeEnumTypeHandler<T extends Enum<T> & WithCode> extends BaseTypeHandler<T> {

    private final Map<Integer, T> mapping;

    public BaseWithCodeEnumTypeHandler(Class<T> clazz) {
        mapping = Arrays.stream(Objects.requireNonNull(clazz).getEnumConstants()).collect(Collectors.toMap(it -> it.code(), Function.identity(), (u, v) -> u));
    }

    @Override
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, T t, JdbcType jdbcType) throws SQLException {
        preparedStatement.setInt(i, t.code());
    }

    @Override
    public T getNullableResult(ResultSet resultSet, String s) throws SQLException {
        int code = resultSet.getInt(s);
        return mapping.get(code);
    }

    @Override
    public T getNullableResult(ResultSet resultSet, int i) throws SQLException {
        int code = resultSet.getInt(i);
        return mapping.get(code);
    }

    @Override
    public T getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
        int code = callableStatement.getInt(i);
        return mapping.get(code);
    }
}
```

常用库

1. 使用Objects.equals(strA, strB)比较两个对象是否相等，避免对左边对象判空
2. 使用commons-lang3操作字符串，包装临时对象Pair与Trible
3. 使用commons-collections集合工具类操作集合
4. 使用common-beanutils 操作对象，进行对象和map互转
5. 使用commons-io 文件流处理，读文件、写文件以及复制文件等
6. 使用Guava操作集合，新建、切分集合，使用Multimap可以一个key映射多个value的HashMap，使用BiMap一种连value也不能重复的HashMap，实质是双向的映射，kv可以实现互转，Table可以有两个key的Hashmap，Multiset一种用来计数的Set

- springboot启动时不报错，直接退出，使用trycatch包住启动代码，查看异常

  ```java
  try {
    SpringApplication.run(Pallas.class, args);
  }catch(Throwable e) {
    e.printStackTrace();
  }
  ```

```java
//springboot读取resource下的文件
public void testReadFile() throws IOException {
//        ClassPathResource classPathResource = new ClassPathResource("resource.properties");
        Resource resource = new ClassPathResource("resource.properties");
        InputStream is = resource.getInputStream();
        InputStreamReader isr = new InputStreamReader(is);
        BufferedReader br = new BufferedReader(isr);
        String data = null;
        while((data = br.readLine()) != null) {
            System.out.println(data);
        }
        
        br.close();
        isr.close();
        is.close();
    }
```



## python

建立虚拟环境

1、python3 -m venv  文件夹

2、source 文件夹/bin/activate

3、退出：deactivate

sys.stdin使用command+d结束输入

## Sublime text3

`command+shift+j`格式化json

`command+shift+p`搜索install

## IDEA

1. 查看接口实现

   - control+h，会跳出所有的

   - 点接口左边的按钮

     <img src="./images/image-20210108222954622.png" alt="image-20210108222954622" style="zoom:25%;" />
   
2. 编辑自己的快捷键

   <img src="/Users/wyj/Desktop/note/开发笔记/images/image-20210911132809087.png" alt="image-20210911132809087" style="zoom:50%;" />

3. 一些常用的插件

   代码质量的：Alibaba Java Coding Guidelines、SonarLint

   json与bean互转：GsonFormat-plus、java Bean to Json

4. 断点调试

   - step over下一步，不进入方法

   - step into下一步，如果是方法进入方法

   - force step into能够进入所有的方法，包括jdk的方法

   - step out跳出

   - resume program恢复程序运行，但是如果接下来还有断点，停在接下来的断点上

   - stop直接停止程序

   - mute breakpoints所有断点失效

   - view breakpoints查看所有断点

     [参考这里](https://www.pdai.tech/md/java/jvm/java-jvm-debug-idea.html)

5. 快捷键

   格式化所有代码：option+command+L

   注销单行：command+/

   注销多行：option+command+/+

   

## MyBatis

```xml
type字段指定pojo，id是ResultMap的id，用来在别处引用
<resultMap id="BaseAggregate" type="com.meituan.ai.sco.rhea.dal.model.BaseAggregate">
  	column是数据库的字段，jdbcType是数据库的类型，property是pojo的类型
    <result column="template_value" jdbcType="VARCHAR" property="templateValue"/>
</resultMap>
```

```xml
id与代码接口中的函数名对应起来，parameterType是查询的参数，resultMap是查询的结果
<select id="baseStatisticsAggregateSearch" parameterType="com.meituan.ai.sco.rhea.dal.criterion.BaseStatisticsCriteria"
            resultMap="BaseAggregate">
        select *
        from
        (
        select
        <trim suffixOverrides=",">trim用于去除或者拼接字符，suffixOverrides=”,“去除sql语句后面的逗号
            <if test="templateColumn">if test用于测试是否满足某种条件，直接写相当于！=null
                template_value,
                template_name,
                template_type,
            </if>
        </trim>
        from
        ai_sco_base_call_statistics
        where
        <trim prefixOverrides="and">去除最前边的and
            <if test="whereStartDate != null">
                and call_date >= #{whereStartDate}
            </if>
            <if test="whereTenantIds != null">
                and tenant_id in
                foreach用于循环列表，将列表用逗号分隔前后加括号拼接
                <foreach collection="whereTenantIds" open="(" close=")" separator="," item="listItem">
                    #{listItem} 预编译阶段会生成?类似的，但是如果${groupByClause}，就会做纯替换
                </foreach>
            </if>
            and is_visible = 1
        </trim>
    </select>
```

## LOG

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="warn"> status用来指定log4j本身的打印日志的级别.
    <appenders>
        <XMDFile name="info-log" fileName="app-info.log">
            <LcLayout/>
        </XMDFile>

        <XMDFile name="error-log" fileName="error.log">
            <ThresholdFilter level="error" onMatch="NEUTRAL" onMismatch="DENY"/>
          			onMatch="ACCEPT" 表示匹配该级别及以上
                onMatch="DENY" 表示不匹配该级别及以上
                onMatch="NEUTRAL" 表示该级别及以上的，由下一个filter处理，如果当前是最后一个，则表示匹配该级别及以上
                onMismatch="ACCEPT" 表示匹配该级别以下
                onMismatch="NEUTRAL" 表示该级别及以下的，由下一个filter处理，如果当前是最后一个，则不匹配该级别以下的
                onMismatch="DENY" 表示不匹配该级别以下的
            <LcLayout/>
        </XMDFile>

        <Scribe name="scribe">远程日志上报，没配置ip的不上报
            <ThresholdFilter level="info" onMatch="NEUTRAL" onMismatch="DENY"/>
            <LcLayout/>
        </Scribe>
        <Async name="scribeAsync" blocking="false">
            <AppenderRef ref="scribe"/>
        </Async>

    </appenders>

    <loggers>
        <root level="info">
            <appender-ref ref="info-log"/>
            <appender-ref ref="error-log"/>
            <appender-ref ref="scribeAsync"/>
        </root>
    </loggers>
</configuration>
```

## JS

```js
//用于视频加速
var tags=document.getElementsByTagName("video");
[...tags].forEach(val=>val.playbackRate = 2);
```

## Postman

粘贴出cookies

```java
document.cookie
```

## 问题排查

磁盘空间不足时：

1. 使用df -h整体查看磁盘
2. 使用du -sh * 查看 / 路径下的各个文件和目录的大小
3. 使用ls -lh命令查看文件占地大小

CPU与内存使用过高

1. top命令
2. 如果只看java使用jps找到PID，然后使用top -p 19063专门查看某个java进程的情况，如果要细化到线程可以加个top -p 19063 -H参数
3. 如果想单独分析内存，使用free

网络延迟

1. netstat -a查看所有连接中的socket, netstat -tnpa命令可以查看所有 tcp 连接的信息，包括进程号
2. 使用ps -ef命令查看进程相关信息

查看java问题

1. 使用jps -l查看java进程号
2. 使用jmap -dump:live,format=b,file=heap.bin 进程号，导出堆快照
3. jhsdb jmap --heap --pid 进程号 查看堆内存设置与当前使用情况
4. jstack -l 进程号，用于生成虚拟机当前时刻线程快照，可以上传到fastthread.io网站直观查看



## SQL

改数据类型

```sql
alter table template_business_value modify  `call_value` float(8,2) NOT NULL DEFAULT '0.00' COMMENT '单条价值', modify `business_ration` float(6,4) NOT NULL DEFAULT '0.00' COMMENT '业务完成率';
```

## flink

安装与启动：

```bash
安装：brew install apache-flink
找到安装目录：brew info apache-flink
进入安装目录执行：  ./libexec/bin/start-cluster.sh
web页面(http://localhost:8081/)
例子：https://www.cnblogs.com/duniqb/p/14070809.html
```

无法关闭：https://blog.csdn.net/qq_37135484/article/details/102474087

使用docker安装

docker-compose.yml

```yml
version: "2.1"
services:
  jobmanager:
    image: flink
    expose:
      - "6123"
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - JOB_MANAGER_RPC_ADDRESS=jobmanager

  taskmanager:
    image: flink
    expose:
      - "6121"
      - "6122"
    depends_on:
      - jobmanager
    command: taskmanager
    links:
      - "jobmanager:jobmanager"
    environment:
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
```

运行

```
docker-compose up -d
```

浏览器打开 http://127.0.0.1:8081 可以看到dashboard

## 一致性哈希

利用哈希去缓存

- 问题？

我们希望有同样ID的请求打到同样的机器上，这可能是因为那台机器有对应的缓存，也可能是因为那台机器上有上下文只有那台机器才能处理

- 怎么办？

将ID做哈希，哈希到不同机器

- 有一台机器崩了还能用吗？

不行了，10台机器，原本ID14的取模到4，4号机器崩了，5-10号顺序往前进一位，ID14号打到了原本的5号机器上，

而且大于4号的都不行了，这会引发大规模报错、大规模缓存雪崩

- 那我不对服务器数取模，对一个固定的数字取模不行吗？

那咋扩容、缩容，而且原来模4的请求不是都不能用吗

- 那把10台机器分布到一个2^32-1的哈希环上，用IP什么的对2^32-1进行取模，分布上去，我再用ID取模也放到哈希环上，然后顺时针找最近的机器搞上去，所有请求都这么搞，当一台机器挂了的时候，原本应该到这台的请求到了顺时针下一台，其余的不受影响，这样可以吗？

可以是可以，但是你就10台机器，对2^32取模以后差不多集中在环上的一个部分，但是请求ID有123456781234567123456这么长，一取模，大部分估计都落到了一台上，那台机器要是挂了，咋办?

<img src="./images/image-20210724162702288.png" alt="image-20210724162702288" style="zoom:25%;" />

- 那把那10台服务器弄点虚拟节点出来，放在哈希环的空白处，每个虚拟他个100000台，均匀撒在环上的空白处能行吗？

这样倒是可以，咋虚拟？

- [.........NodeA1............NodeA2...............NodeA3.............]顺时针找，碰到带序号的都给映射回不带序号的节点去



## mongodb

使用brew 来安装 mongodb：

```
brew tap mongodb/brew
brew install mongodb-community@4.4
```

**@** 符号后面的 **4.4** 是最新版本号。

安装信息：

- 配置文件：**/usr/local/etc/mongod.conf**
- 日志文件路径：**/usr/local/var/log/mongodb**
- 数据存放路径：**/usr/local/var/mongodb**



运行 MongoDB

我们可以使用 brew 命令或 mongod 命令来启动服务。

brew 启动：

```
brew services start mongodb-community@4.4
```

brew 停止：

```
brew services stop mongodb-community@4.4
```

mongod 命令后台进程方式：

```
mongod --config /usr/local/etc/mongod.conf --fork
```

这种方式启动要关闭可以进入 mongo shell 控制台来实现：

```
> db.adminCommand({ "shutdown" : 1 })
```



## 测试

可能用到的一些依赖

``` xml
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-test-autoconfigure</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-core</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>com.jayway.jsonpath</groupId>
  <artifactId>json-path</artifactId>
</dependency>
```

service层mock dao层数据

```java
package com.wyj.controller;

import com.wyj.dao.PassiveCallInfoDao;
import com.wyj.entity.PassiveCallInfo;
import com.wyj.service.PassiveCallInfoService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.junit4.SpringRunner;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.mockito.Mockito.when;
import static org.hamcrest.core.IsEqual.equalTo;

/**
 * @author wyj
 */

@RunWith(SpringRunner.class)
@SpringBootTest
public class ServiceMock {
    @Autowired
    private PassiveCallInfoService passiveCallInfoService;
    @MockBean
    private PassiveCallInfoDao passiveCallInfoDao;

    @Test
    public void test1(){//搭配mock使用
        final var passiveCallInfo=new PassiveCallInfo();
        passiveCallInfo.setComplainTag("true");
        when(passiveCallInfoDao.selectByPrimaryKey(32L)).thenReturn(passiveCallInfo);
        assertThat(passiveCallInfoService.selectByPrimaryKey(32L).getComplainTag(),equalTo("true"));
    }

}
```

service层直接测试

```java
package com.wyj.controller;

import com.wyj.service.PassiveCallInfoService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.core.IsEqual.equalTo;

@RunWith(SpringRunner.class)
@SpringBootTest
public class ServiceFact {
    @Autowired
    private PassiveCallInfoService passiveCallInfoService;

    @Test
    public void test2(){
        assertThat(passiveCallInfoService.selectByPrimaryKey(32L).getComplainTag(),equalTo("32"));
    }
}
```

controller层mock service层数据

```java
package com.wyj.controller;

import com.wyj.dao.PassiveCallInfoDao;
import com.wyj.entity.PassiveCallInfo;
import com.wyj.service.PassiveCallInfoService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.junit4.SpringRunner;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.mockito.Mockito.when;
import static org.hamcrest.core.IsEqual.equalTo;

/**
 * @author wyj
 */

@RunWith(SpringRunner.class)
@SpringBootTest
public class ServiceMock {
    @Autowired
    private PassiveCallInfoService passiveCallInfoService;
    @MockBean
    private PassiveCallInfoDao passiveCallInfoDao;

    @Test
    public void test1(){//搭配mock使用
        final var passiveCallInfo=new PassiveCallInfo();
        passiveCallInfo.setComplainTag("true");
        when(passiveCallInfoDao.selectByPrimaryKey(32L)).thenReturn(passiveCallInfo);
        assertThat(passiveCallInfoService.selectByPrimaryKey(32L).getComplainTag(),equalTo("true"));
    }

}
```

controller层直接测试

```java
package com.wyj.controller;

import com.wyj.service.PassiveCallInfoService;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.mock.web.MockHttpSession;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringRunner.class)
@SpringBootTest
/**
 * 1 这个注解用于集成测试，也就是默认会加载完整的Spring应用程序并注入所有所需的bean。一般会通过带有@SpringBootApplication的配置类来实现。
 * 2 由于会加载整个应用到Spring容器中，整个启动过程是非常缓慢的（10+秒起步），一般会用于集成测试，可以使用TestRestTemplete或者MockMvc来发起请求并验证响应结果。
 * 3 SpringBootTest中的也可以使用Mockito等Mock工具来对某些bean进行mock，但是一般不会只对单个层进行测试，推荐用于单个应用的端到到集成测试。
 * 4 如果涉及到第三方依赖，如数据库、服务间调用、Redis等，可以考虑服务虚拟化方案。
 */
public class ControllerFact {

    private MockMvc mockMvc;

    @Autowired
    private WebApplicationContext wac;

    @Before
    public void setupMockMvc(){
        mockMvc = MockMvcBuilders.webAppContextSetup(wac).build(); //初始化MockMvc对象
    }

    @Test
    public void test4() throws Exception {
        final var authResult=mockMvc.perform(MockMvcRequestBuilders.get("/hello2").param("id", String.valueOf(32L)))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.complainTag").value("32"))
                .andReturn().getResponse().getContentAsString();
        System.out.println("返回结果"+authResult);
    }
}
```

使用http测试

``` java
package com.wyj.controller;

import com.wyj.entity.PassiveCallInfo;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.boot.web.server.LocalServerPort;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.web.client.RestTemplate;

import java.util.HashMap;
import java.util.Map;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.core.IsEqual.equalTo;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
public class HttpRequestTest {
    @LocalServerPort//随机端口有助于在测试环境中避免冲突
    private int port;

    @Autowired
    RestTemplateBuilder builder;

    private RestTemplate restTemplate;

    @Before
    public void restTemplate() {
        restTemplate=builder.build();
    }

    @Test
    public void test1(){
        String x=restTemplate.getForObject("http://localhost:"+port+"/hello",String.class);
        assertThat(x, equalTo("hello"));
    }

    @Test
    public void test2(){
        Map<String, String> params = new HashMap<>();
        params.put("id", "32");
        PassiveCallInfo x=restTemplate.getForObject("http://localhost:"+port+"/hello2"+"?id={id}", PassiveCallInfo.class,params);
        assertThat(x.getTargetNumToken(), equalTo("321"));
    }
}
```



## ES

- 安装

  ```shell
  docker network create es-net
  docker pull elasticsearch:7.12.1
  docker run -d \
  	--name es \
      -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
      -e "discovery.type=single-node" \
      -v es-data:/usr/share/elasticsearch/data \
      -v es-plugins:/usr/share/elasticsearch/plugins \
      --privileged \
      --network es-net \
      -p 9200:9200 \
      -p 9300:9300 \
  elasticsearch:7.12.1
  
  命令解释：
  -e "cluster.name=es-docker-cluster"：设置集群名称
  -e "http.host=0.0.0.0"：监听的地址，可以外网访问
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m"：内存大小
  -e "discovery.type=single-node"：非集群模式
  -v es-data:/usr/share/elasticsearch/data：挂载逻辑卷，绑定es的数据目录
  -v es-logs:/usr/share/elasticsearch/logs：挂载逻辑卷，绑定es的日志目录
  -v es-plugins:/usr/share/elasticsearch/plugins：挂载逻辑卷，绑定es的插件目录
  --privileged：授予逻辑卷访问权
  --network es-net ：加入一个名为es-net的网络中
  -p 9200:9200：端口映射配置
  在浏览器中输入：http://localhost:9200 即可看到elasticsearch的响应结果
  ```

  <img src="./images/image-20210912104719089.png" alt="image-20210912104719089" style="zoom: 50%;" />

- 安装kibana

  ```shell
  docker pull kibana:7.12.1
  docker run -d \
  --name kibana \
  -e ELASTICSEARCH_HOSTS=http://es:9200 \
  --network=es-net \
  -p 5601:5601  \
  kibana:7.12.1
  
  --network es-net ：加入一个名为es-net的网络中，与elasticsearch在同一个网络中
  -e ELASTICSEARCH_HOSTS=http://es:9200"：设置elasticsearch的地址，因为kibana已经与elasticsearch在一个网络，因此可以用容器名直接访问elasticsearch
  -p 5601:5601：端口映射配置
  ```

- 安装ik插件

  ```shell
  # 进入容器内部
  docker exec -it elasticsearch /bin/bash
  
  # 在线下载并安装
  ./bin/elasticsearch-plugin  install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.12.1/elasticsearch-analysis-ik-7.12.1.zip
  
  #退出
  exit
  #重启容器
  docker restart elasticsearch
  ```

  ![image-20210912115036265](./images/image-20210912115036265.png)

  