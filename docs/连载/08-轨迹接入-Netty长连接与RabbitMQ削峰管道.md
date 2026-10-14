# 轨迹接入：Netty 长连接与 RabbitMQ 削峰管道

调度和作业都在"计划"世界里打转，从这一章开始，我们进入实时世界——车辆到底在哪、跑得正不正常。这套能力由独立的 `pd-netty` 服务提供，它和前面那些普通的 HTTP 微服务长得完全不一样：它不处理用户点按钮的请求，而是守着一个 TCP 端口，接住成百上千台车载终端和司机手机源源不断推上来的 GPS 点位。

这一章讲两件事：接入层怎么用 Netty 扛住海量长连接并保证报文干净，以及为什么点位不直接写库、而要先过一道 RabbitMQ。

## 为什么轨迹必须用 TCP 长连接

先回答一个常见疑问：现在 HTTP 这么方便，为什么不用接口上报，非得搞 Netty？

算一笔账。一辆车每 15 秒回传一个点，1000 辆车就是每秒约 67 个请求，一天近 580 万个点位。如果每次都走 HTTP，意味着每次都要三次握手（或 TLS 协商）、带一整套请求头、再由 Web 容器走一遍完整的鉴权过滤链，为了传几十字节的经纬度，开销极其浪费。更关键的是车载终端往往需要**双向通信**——平台要能主动下发指令（比如围栏告警、要求拍照），HTTP 的请求-响应模型天生不擅长服务端推送。

所以行业标准做法是 TCP 长连接：终端上电就和平台建立连接并一直保持，之后每隔十几秒发一个紧凑报文，连接还能复用于下行指令。Java 世界干这件事的首选框架就是 Netty——基于 NIO 的多路复用，少量线程就能撑住成千上万条连接。品达把它单独做成一个服务是对的：长连接的内存模型、负载特征和无状态 HTTP 服务截然不同，独立部署才能单独扩容、单独重启而不影响业务接口。

## Pipeline：一条流水线解决拆包粘包

Netty 的核心抽象是 ChannelPipeline——入站字节流经过一串 Handler 层层加工。`NettyServer` 里装配的这条管道只有五行，却每一行都有讲究：

```java
ch.pipeline().addLast(new LineBasedFrameDecoder(4096));                 // 按换行切帧，解决半包/粘包
ch.pipeline().addLast(new StringDecoder(StandardCharsets.UTF_8));       // 字节 → 字符串
ch.pipeline().addLast(new StringEncoder(StandardCharsets.UTF_8));       // 字符串 → 字节（回写应答）
ch.pipeline().addLast(new IdleStateHandler(120, 0, 0, TimeUnit.SECONDS));// 120s 读空闲检测
ch.pipeline().addLast(nettyServerHandler);                              // 业务处理
```

这套系统和终端约定的报文协议极其简单：**一条 JSON 报文以换行符 `\n` 结尾**。于是用 `LineBasedFrameDecoder`（行分隔解码器，单行上限 4096 字节）就能解决 TCP 编程最经典的两个坑——半包和粘包。TCP 是字节流，不保证你 `send` 一次对方就 `recv` 到完整一条：可能半条报文分两次到（半包），也可能几条报文挤在一个包里到（粘包）。行解码器在管道最前面把字节流按换行符重新切成一帧帧完整的报文，后面的 `StringDecoder` 再把整帧转成 String，业务 Handler 拿到的就永远是一条完整 JSON，不用再操心字节边界。这个"用简单的分隔符协议换成熟解码器"的工程取舍，比自己手写状态机拼包省心太多。

## 连接治理：防风暴、治僵尸、不靠连接判在线

业务处理器 `NettyServerHandler` 继承 `ChannelInboundHandlerAdapter`，还标注了 `@ChannelHandler.Sharable`（无状态、可被多条连接共享）。它在连接管理上有三处很实战的设计。

**第一，连接数上限保护。** 所有活跃连接被放进一个 `ChannelGroup` 统一管理，新连接进来先看数量：

```java
private static final int MAX_CONNECTIONS = 1000;
public void channelActive(ChannelHandlerContext ctx) {
    if (activeChannels.size() >= MAX_CONNECTIONS) {
        // 防止异常设备重连风暴耗尽文件句柄
        ctx.writeAndFlush("ERR:too-many-connections\n");
        ctx.close();
        return;
    }
    activeChannels.add(ctx.channel());
}
```

为什么要限到 1000？考虑一种故障场景：某批设备程序出 bug，断了就疯狂重连，每次重连都消耗服务器一个文件描述符（fd）。操作系统对单进程 fd 有上限，不设限的话，几千次重连风暴就能把连接表打满，拖垮整个服务。超限直接拒绝，是保命的熔断。

**第二，读空闲超时清理僵尸连接。** 管道里的 `IdleStateHandler(120,0,0)` 会在连接 120 秒没收到任何数据时触发一个空闲事件，Handler 里主动关掉它：

```java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent) {
        // 设备长时间无上报，判定为僵死连接，主动关闭释放资源
        ctx.close();
        return;
    }
}
```

移动网络下连接悄悄死掉是常态（进隧道、基站切换、NAT 超时），TCP 层面可能半天察觉不到，这些僵尸连接会白白占着资源。120 秒无数据就回收，是必要的卫生措施。

**第三，也是认知上最重要的一点：车辆在不在线，不看连接在不在，而看心跳。** 空闲处理的注释里有一句话点破了设计思想——"车辆在线状态不依赖连接本身，而由心跳超时扫描 `pd_truck.last_heartbeat_time` 置离线"。为什么？因为 TCP 连接存活 ≠ 设备真在正常上报。一条连接可能僵在那儿但终端其实已经死机；反过来，设备短暂网络切换后重连，连接对象变了，但车从未"离线"。所以系统在数据库车辆表上维护一个 `last_heartbeat_time`（最近上报时间），再有一个定时任务 `scanOfflineTrucks` 每分钟扫描，超过心跳超时阈值（默认 5 分钟）没新点的车置为离线。**用业务心跳而不是传输层连接来定义在线状态**，才能容忍连接的抖动、反映真实的设备情况。

## 入口校验：HTTP 和 TCP 同一把尺子

合法连接进来后，每条报文在 `channelRead` 里走一道统一校验，通过才发给消息队列，否则回复 `ERR:bad-request` 并不入队。这里有个很值得提倡的原则——**TCP 入口和 HTTP 入口共用同一套校验逻辑**（`GpsLocationValidator`），不因为是"内部设备直连"就放松：

```java
String invalidMsg = GpsLocationValidator.validateRequired(message);   // 必填字段 + 经纬度
if (invalidMsg != null) { return null; }

// 坐标系缺失补默认 BD09，非法值拒绝
if (coordSystem == null || coordSystem.trim().isEmpty()) {
    message.setCoordSystem(CoordSystem.BD09.getCode());
} else if (!CoordSystem.isValid(coordSystem)) { return null; }

// 来源缺失补 NETTY_TCP，非法值拒绝
if (source == null || source.trim().isEmpty()) {
    message.setSource(LocationSource.NETTY_TCP.getCode());
} else if (!LocationSource.isValid(source)) { return null; }
```

脏数据在最外层就被挡下，绝不允许流到后面的存储和告警，否则一条经纬度为 0 的坏点可能在地图上画出一条冲向几内亚湾的诡异轨迹。处理完向设备回一行 `OK` 或 `ERR:原因`，让终端能感知上报是否成功、要不要重发。

代码里还埋了一个清晰的演进钩子：当前设备是手机端、直接发 JSON，默认按百度坐标系 BD09 处理；注释写明"后续接 JT/T 808 北斗终端时，按协议解析后改为 source=JT808 + coordSystem=CGCS2000"。JT/T 808 是国内营运车辆的国标通信协议，真实车队规模化后必然要接，这里把坐标系和来源的扩展位提前留好了。另外 TCP 报文没有 HTTP 头可带租户，缺失时用配置的默认租户兜底，保证消费端拿得到——又是系统态租户那套思路。

## 削峰管道：Kafka 还是 RabbitMQ

校验通过的报文，Handler 并不直接写数据库，而是交给 `RabbitSender` 投递给消息队列：

```java
public static final String GPS_EXCHANGE = "xinfei.gps.exchange";
public static final String GPS_QUEUE   = "xinfei.gps.queue";

// 用 truckId 作为 routingKey，保证同一辆车的轨迹串行消费，不会乱序
public void sendGpsTrace(String truckId, String message) {
    try {
        String routingKey = "gps." + truckId;
        rabbitTemplate.convertAndSend(GPS_EXCHANGE, routingKey, message);
    } catch (Exception e) { /* 发送失败内部消化，不拖垮连接线程 */ }
}
```

中间隔一道队列，价值有三层。**削峰**：早高峰几万车同时上线，点位瞬时洪峰先在队列里缓冲，消费端按自己能承受的速率平稳落库，不会把数据库打爆。**隔离**：接入层只负责快速接报文、丢进队列就立刻回 OK，绝不能因为数据库慢或告警计算重而阻塞 Netty 的 IO 线程——IO 线程一堵，所有连接都得超时。**解耦**：日后新增"实时大屏""电子围栏"这些消费方，直接订阅同一个交换机即可，接入层一行不用改。

这里用的是 Topic 交换机，routingKey 设计成 `gps.{truckId}` 很用心：队列绑定 `gps.#` 能收全部点位，同时这个按车辆区分的路由键为"同一辆车的点路由到同一消费者、保证时序"留了可能——轨迹对顺序敏感，一辆车的点若被多线程乱序消费，轨迹线就会来回跳。交换机和队列都声明为 durable（持久化），MQ 重启消息不丢。

顺便又一次文档与代码的漂移：README 里轨迹管道写的是 Kafka，而实际代码从交换机、队列到 `@RabbitListener` 全是 RabbitMQ。选型本身没有对错（量级真上来了 Kafka 更合适），但这再次印证——**别信陈旧的文档，信代码。**

## 消费端：坐标统一、心跳回写、平稳落库

队列另一头，`RabbitGpsTraceConsumer` 用 `@RabbitListener(queues = "xinfei.gps.queue")` 接住点位，依次做坐标转换、异常检测、心跳回写和落库。

坐标转换是国内地图应用绕不开的一步。设备可能上报 WGS84（GPS 原始）、GCJ02（高德/腾讯，火星坐标）、BD09（百度）三套坐标，不统一就会偏移几百米。系统规定**入库和展示一律 BD09**：

```java
CoordSystem cs = CoordSystem.of(location.getCoordSystem());
if (cs == CoordSystem.BD09)        { bd09 = new double[]{lng, lat}; }
else if (cs == CoordSystem.GCJ02)  { bd09 = CoordTransform.gcj02ToBd09(lng, lat); }
else                               { bd09 = CoordTransform.wgs84ToBd09(lng, lat); }
location.setLng(CoordTransform.toFixed6(bd09[0]));   // 统一保留六位小数
location.setLat(CoordTransform.toFixed6(bd09[1]));
```

之后若是车辆类型，Feign 调 `pd-base` 回写心跳时间——注意心跳失败只告警、不影响轨迹落库（主次分明）；最后把点位转成 `LocationRecord`：ID 用雪花、坐标用 BigDecimal、坐标系标记为 BD09、租户由多租户插件补入，并按"同业务方 + 同时间戳"做自然点去重。为了支撑后面的异常检测，消费端还维护了一个最多 5000 个键的 LRU 内存缓存，把每辆车最近的轨迹点留在内存里，并有个守护线程每小时清理过期键，避免无界增长。

整条链路串起来就是：**终端 TCP 长连接 → 行解码 → 校验 → RabbitMQ 缓冲 → 消费端转坐标/心跳/落库**。接入快、管道稳、存储稳，实时轨迹的地基就立住了。

点位进了库、留在了缓存，下一章我们就用它们做点更有业务价值的事：实时识别超速、偏航、长时间停留和车辆离线四类告警，以及怎么用冷热分离让几百万轨迹点不把表拖垮。
