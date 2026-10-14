# 智能调度总览：五阶段链路与 XXL-Job 改造

从这一章开始，我们进入品达 TMS 最硬核、也最值钱的部分——智能调度。

先说清楚调度到底在干什么。想象一家物流公司的调度室，每天凌晨要处理几千张昨天新下的单：哪些货该凑一车、走哪条线路、坐几点的车次、派哪个司机开哪辆车，全靠调度员盯着几块屏幕手工排。系统要做的，就是把这套人脑决策自动化。品达把这整套决策组织成一条五阶段流水线，入口在 `pd-dispatch` 的 `DispatchTask`。

这一章我先带你从空中俯瞰整条链路，再重点讲它"怎么被定时触发、怎么在多租户环境下安全地跑、怎么保证一单出错不拖垮全场"。这些工程问题的难度，一点不比算法本身低。

## 五阶段链路：调度室的五个动作

核心编排方法 `DispatchTask.run`，上一章已经露过脸，这里逐行讲：

```java
public void run(Long businessId, String params, Long jobId, Long logId) {
    // 1. 订单分类
    List<OrderClassifyGroupDTO> orderClassifys =
        orderClassifyService.execute(businessId, jobId, logId);
    // 2. 路线规划
    List<OrderLineSimpleDTO> orderLineSimpleDTOS =
        routePlanningService.execute(orderClassifys, businessId, jobId, logId, params);
    // 3. 创建运输任务；关联运单
    Map<String, TaskTransportDTO> transportTaskMap =
        businessOperationService.createTransportOrderTask(orderLineSimpleDTOS);
    // 4. 车次 车辆 司机
    List<OrderLineTripsTruckDriverDTO> dtos =
        truckSchedulingService.execute(orderLineSimpleDTOS, businessId, jobId, logId, businessId);
    // 5. 完善运输任务信息；创建司机作业任务
    businessOperationService.updateTransportTask(dtos, transportTaskMap);
}
```

五个阶段，产出像流水线一样一级喂给一级：

1. **订单分类**：把当前机构下所有待运订单捞出来，按"起点机构 + 终点机构"归类。北京发上海的归一组、北京发深圳的归另一组。中转中的订单（状态为运输中、当前正停在本机构）也要捞出来接着排。
2. **路线规划**：给每一组订单，在运输线路网络里找出可行路径，按配置策略选出一条默认线路。这是第 6 章的 DFS 递归。
3. **创建运输任务、关联运单**：线路定了，就生成第 4 章讲的 `TaskTransport`，把这批运单挂上去。
4. **匹配车次、车辆、司机**：沿着选中的线路查可用车次，再过滤出状态正常的车和在岗司机，选发车时间最合适的。第 7 章讲。
5. **回填任务、生成司机作业单**：把车次车辆司机写回运输任务，同时给司机手机端生成待办作业单。

跑完这五步，一个机构这一轮的货就全排好了。注意这个方法的入参第一个是 `businessId`（机构 ID）——**调度是"每个机构独立跑一轮"的**，上海网点和武汉网点各有各的订单、各跑各的任务，互不干扰。

## 谁来按这个按钮：Quartz 到 XXL-Job 的迁移

五阶段链路是血肉，但它不会自己动，得有人定时喊它。这套系统的触发方式经历过一次真实的迁移，代码注释把时间和动机都记下来了：

```java
/**
 * 调度任务服务（2026-10-10 调度改造）：
 * 定时触发已统一迁至 XXL-Job（xxl-job-admin 控制台管理 Cron/启停），
 * 本服务只负责调度任务配置的 CRUD 与「立即执行」（run 直调 DispatchTask，
 * 保留系统态租户绑定、业务级失败重试与执行日志写入，行为与原 Quartz 链路等价）。
 */
```

最早定时任务用的是 Quartz——一个把任务和 Cron 表达式存在业务库里的经典框架。2026 年 10 月这轮改造把"定时触发"统一搬到了 XXL-Job。为什么要搬？核心是可运维性：Quartz 的任务配置散在各服务的数据库表里，改个 Cron、临时停个任务都得连库操作；XXL-Job 有一个独立的调度中心（xxl-job-admin），所有任务在网页上可视化配置 Cron、一键启停、看执行日志、配置失败告警，微服务多了之后这种"集中调度面板"的价值非常大。

迁移后，`DispatchTask` 里多了一个 XXL-Job 的标准入口，任务参数就是机构 ID：

```java
@XxlJob("dispatchTask")
public void runJob() {
    String param = XxlJobHelper.getJobParam();
    Long orgId = IdConvert.toLong(param);
    if (orgId == null) {
        XxlJobHelper.handleFail("机构ID参数非法: " + param);
        return;
    }
    scheduleJobService.runByOrgId(orgId);
}
```

运维在调度中心为每个机构配一个任务、参数填机构 ID，到点 XXL-Job 就回调这个 Bean，最终走到 `runByOrgId` → 五阶段链路。README 里还写着"用 Quartz 定时调度"，但代码里早就换成 XXL-Job 了——**文档没跟上代码，这是开源项目的常态，读代码时永远以能运行的实现为准。**

## 系统态线程的头号难题：租户从哪来

定时任务和普通 HTTP 请求有个本质区别：它没有请求头、没有登录态。可这套系统是多租户的——上海某物流公司和广州另一家物流公司的数据必须严格隔离。正常请求里，租户 ID 由网关注入、一路透传；可调度线程是 XXL-Job 的线程池拉起来的，上下文里空空如也，不处理就会读到租户 0 的脏数据，甚至串号。

品达的解法在 `ScheduleJobServiceImpl.executeWithRetry` 里，思路是"执行前按机构反查租户、绑定到当前线程，执行完一定清理"：

```java
// 系统态租户绑定：调度线程无登录态与请求头，按机构ID反查租户并写入当前线程上下文，
// 供出站 Feign 拦截器读取透传给 pd-auth；执行结束在 finally 中清理，避免线程复用串租户。
Long tenantId = 0L;
Long businessId = scheduleJob.getBusinessId();
try {
    tenantId = orgTenantResolver.resolve(businessId);
} catch (Exception e) {
    log.warn("调度任务解析租户失败，回落租户0，任务id：{}", scheduleJob.getId(), e);
}
TenantContextHolder.setTenantId(tenantId);
try {
    // ... 执行五阶段、重试、写日志
} finally {
    TenantContextHolder.clear();
}
```

这里有两个细节值得你记住。一是**反查失败要有兜底**：租户解析挂了不能让任务直接崩，回落租户 0 并告警，至少保证流程可控。二是 **`finally` 里必须 clear**：XXL-Job 用的是线程池，线程会被复用执行下一个机构的任务，如果这次跑完不清，下一家机构就会带着上一家的租户 ID 去查数据——跨公司串数据，这在 SaaS 里是 P0 级事故。凡是脱离请求线程的异步逻辑（定时、MQ 消费、异步线程池），都要做这套"绑定—传递—清理"，GPS 消费端也有一模一样的处理。

## 日志、重试，与一个反直觉的事务决策

同一个方法里还有几处很见功力的设计。

**执行日志要先落库，再跑任务。** 因为订单分类记录需要挂在这条执行日志上（分类表里有个 `job_log_id`），得先插入日志、让 MyBatis-Plus 回填雪花 ID，再把这个 ID 传给五阶段链路。顺序反了就关联不上。

**业务级失败重试。** 框架自带的重试往往粒度太粗，这里自己实现了一层：按配置的最大尝试次数 `maxAttempts` 和间隔 `intervalMs` 循环，每轮失败 sleep 后重试，最后一轮仍失败，就把完整异常栈写进日志的 error 字段。这样远程服务偶尔抖一下，不至于立刻判定整轮调度失败。

最反直觉的是这段注释——这个 `run` 方法**故意不加 `@Transactional`**：

```java
// 显式不加 @Transactional：循环内每个机构各自执行 + 业务级失败重试 + 独立写执行日志，
// 且 dispatchTask.run 内层（如 TaskOrderClassifyServiceImpl#execute）自带 @Transactional。
// 若此处包外层事务，内层失败会把共享事务标记为 rollback-only，被 catch 重试后最终回写日志
// 反抛 UnexpectedRollbackException，导致失败日志落不了库。
```

没踩过这个坑的人很难理解。我解释一下：Spring 里内外方法共用同一个事务时，内层方法一旦抛异常，这个共享事务就被标记成 rollback-only（只能回滚）；哪怕外层把异常 catch 住、想当没事发生继续写执行日志，等事务提交时也会被告知"这个事务早就注定回滚了"，于是抛出 `UnexpectedRollbackException`，连那条本该留下的失败日志都写不进去。所以作者干脆不开外层事务，让每个机构、每个内层阶段各管各的事务。**事务不是越大越安全，在有重试和容错的批处理里，盲目包大事务反而会毁掉你的错误处理。** 这是我在这个仓库里最想拍案叫绝的一个注释。

## 细看阶段一：订单是怎么被分类的

最后落到第一阶段的实现。`TaskOrderClassifyServiceImpl.execute` 先捞两类订单：新订单（当前机构待发出的）和中转订单（状态是运输中、`currentOrgId` 正好停在本机构的）。然后用一个 Stream 按 `groupBy` 键分组——这个键本质就是"起点机构 + 终点机构"：

```java
Map<String, List<OrderClassifyDTO>> orderClassifyDTOGroup =
    orderClassifyDTOS.stream().collect(Collectors.groupingBy(OrderClassifyDTO::groupBy));
```

如果一张订单连起点或目的机构都解析不出来（地址异常、网点缺失），它会被归进一个特殊的 `ERROR` 组，这组不参与寻路，后面会被登记成"调度异常单"推给运营人工处理，而不是静默丢弃。

分组结果落库是典型的主从表写法：先存 `disp_order_classify` 主表（分组键、起止机构、订单数、jobId、jobLogId），雪花 ID 回填后，再批量存每一条"分组—订单"明细。中转订单则反过来，按订单 ID 反查它历史上属于哪个分组，继续往下一程走。

这里还有一条标记为 `P0 事务隔离` 的防御性设计，我特别认同。在遍历订单构造分类时，每一张订单都单独包了 try-catch：

```java
// 单订单分类失败（地址异常/区域不一致/网点缺失/地图异常等）不应中断
// 整个机构的调度轮次：失败订单仅记录日志并跳过，其余订单继续参与分类与调度，
// 避免"一单出错、整轮回滚、全部漏调度"。
```

道理很朴素但极关键：一轮调度有几千张订单，如果因为其中一张脏数据抛异常就整轮回滚，结果是这几千张正常订单全部漏调度。把坏单子隔离开、踢出去单独人工处理，让好单子照常走，才是批处理该有的韧性。

## 一个小彩蛋：@Lazy 打破循环依赖

读 `DispatchTask` 时你可能注意到一个细节：它注入 `ScheduleJobService` 时加了 `@Lazy`。因为 `ScheduleJobServiceImpl` 本身又注入了 `DispatchTask`（它要回调任务执行），两者互相依赖形成循环。Spring 启动时无法决定先创建谁，加 `@Lazy` 让其中一侧先注入一个延迟代理，真正用到时才解析，循环就解开了。这不是最优的结构解耦，但在这种"编排者与执行者互相回调"的场景里是个干脆实用的解法。

五阶段的骨架和它脚下的工程地基，到这里就讲完了。下一章，我们钻进第二阶段的内部，看系统是怎么在一张运输线路网络图上，用 DFS 递归把"从北京到大理"这种没有直达线路的订单，一步一步倒腾出一条中转路径来的。
