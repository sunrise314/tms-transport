# 路径规划：DFS 递归寻路与线路版本化缓存

上一章订单分类完成，系统手里有了一摞"起点 → 终点"的订单组。第二阶段要回答一个更难的问题：**这些货怎么走？**

难在哪？因为运输线路网络里，两个城市之间不一定有直达线路。北京到上海可能有干线直达；可一张从北京发大理的订单，大概率得北京先到昆明转运中心、再从昆明中转到大理。系统必须在一张由"机构"和"线路"构成的网络图里，自动找出所有可行走法，再挑一条最合理的。

这一章我们钻进 `TaskRoutePlanningServiceImpl`，看它怎么用 DFS（深度优先搜索）递归把中转路径倒腾出来，又是怎么用一个 MD5 指纹把算过的线路缓存复用、并做好版本管理的。

## 先把问题抽象成一张图

理解算法前先建模型。这套系统里有两类主数据：机构（网点、转运中心）和运输线路。每条运输线路是一条**有向边**——从 `startAgencyId` 指向 `endAgencyId`，带着距离、成本、预计时效三个权重；机构则是图上的节点。

于是"找运输路径"就变成了一个经典的图搜索问题：在这张有向图里，找出从起点机构到终点机构的所有路径，既包括直达的一条边，也包括经过若干中转的多条边。因为公路货运的线路网络通常是稀疏的、中转层级不深（一般两三跳），用 DFS 递归把所有走法枚举出来，是简单且够用的选择——这里不需要 Dijkstra、A* 那些重型最短路算法，因为系统要的不是"唯一最短路"，而是"全部候选路径 + 后续按多维度选举"。

## findLine：一段带防环的 DFS

核心是一个私有递归方法 `findLine`。我把主干贴出来并逐段解释：

```java
private RoutePlanningDTO findLine(int depth, String agencyId, String start,
                                  String end, List<String> superiorStarts) {
    RoutePlanningDTO routePlanningDTO = new RoutePlanningDTO(++depth);
    List<String> feasibleIds = new ArrayList<>();

    // 第一步：先找 start -> end 的直达线路
    List<TransportLineDto> transportLineDtos = findDirectLine(start, end, agencyId);
    if (transportLineDtos != null && transportLineDtos.size() > 0) {
        routePlanningDTO.addFeasible(transportLineDtos);
        feasibleIds.addAll(transportLineDtos.stream()
            .map(TransportLineDto::getId).collect(Collectors.toList()));
    }

    // 第二步：找所有以 start 为起点的线路，继续往下递归
    transportLineDtos = findStartLine(start, agencyId);
    superiorStarts.add(start);
    // 过滤：① 已经是直达边的不再重复递归；② 终点不能是本次路径上已出现过的机构（防死循环）
    transportLineDtos = transportLineDtos.stream()
        .filter(item -> ((!feasibleIds.contains(item.getId()))
                      && (!superiorStarts.contains(item.getEndAgencyId()))))
        .collect(Collectors.toList());

    if (transportLineDtos.size() > 0) {
        routePlanningDTO.add(transportLineDtos);
        for (TransportLineDto dto : transportLineDtos) {
            // 以这条边的终点为新起点，递归找它到 end 的路径
            RoutePlanningDTO child = findLine(depth, agencyId, dto.getEndAgencyId(), end, superiorStarts);
            routePlanningDTO.put(dto.getId(), child);
        }
    }
    return routePlanningDTO;
}
```

它的思路是"每到一个节点，先看能不能一步到位，不能就沿着所有出边继续往下试"。第一步 `findDirectLine` 查当前节点到最终终点有没有直达边，有的话这就是一条可行路径，放进 `feasible`。第二步 `findStartLine` 把以当前节点为起点的所有边捞出来，对每条边递归，递归的结果挂到当前节点的子树上，最终拼成一棵"路径树"。

注意这个方法全程用 `String` 类型的机构 ID，注释里写得很清楚——"findLine 递归内部全程走 pd-base String 边界，仅在入口把 disp 内部 Long 机构转 String"。这就是第 2 章讲的那个跨服务类型边界，递归里要反复调 `pd-base` 查线路，索性整个算法内部统一用 String，避免每层都转一次。

**最关键的是防死循环。** 线路是有向边，如果数据里存在环（比如 A→B、B→A 两条边都配了），不加控制的递归会在 A、B 之间无限跳转直到栈溢出。它的防线是 `superiorStarts` 这个集合：每进入一个节点就把它加进去，扩展出边时过滤掉"终点已经在路径上出现过"的边——注释原话是"若不过滤，理论上会出现死循环"。有了它，递归不会回头走已经踩过的节点，环被掐断了。

不过我要给原实现提一个改进点。这个 `superiorStarts` 只在进入节点时 `add`，递归返回后并没有 `remove`（回溯），而且整轮搜索共享同一个集合。这意味着某个机构只要在任意一个分支里被访问过，其他分支再想经过它也会被一并挡掉。在标准 DAG（有向无环图）里这可能错杀一部分合法路径——因为从不同分支到达同一个中间节点本来是允许的。更严谨的写法是每条递归路径维护自己的访问栈、回溯时弹出本层节点，再叠一个最大 `depth` 兜底。我二开时把这里改成了真正的回溯式 DFS 并加了深度上限，既防环又不漏路径，也防止异常数据把递归拖得过深。

## 把路径树拍平：一条候选线路长什么样

递归返回的是一棵树，不方便比较和落库。系统用 `AnalysisRoutePlanningDTO` 把树拍平成一条条完整候选路径，每条路径对应一个 `CacheLineEntity`（缓存线路主对象）加一串 `CacheLineDetailEntity`（路径上每一段边的明细）。

拍平时顺带把整条路径的权重累加出来：距离求和、成本求和、预计时效求和，中转次数就是明细段数减一。于是每个候选线路都带着四个可比较的指标：`distance`、`cost`、`estimatedTime`、`transferCount`。

这里有一个很巧妙的设计——给每条候选路径算一个指纹 `verifyKey`：

```java
// 把路径上每一段的「终点#距离#成本#时效#」依次拼接，再取 MD5
.append(transportLineDto.getEndAgencyId()).append("#")
.append(transportLineDto.getDistance() == null ? "" : transportLineDto.getDistance().toString()).append("#")
.append(transportLineDto.getCost() == null ? "" : transportLineDto.getCost().toString()).append("#")
.append(transportLineDto.getEstimatedTime() == null ? "" : transportLineDto.getEstimatedTime().toString()).append("#");
...
cacheLineEntity.setVerifyKey(DigestUtils.md5DigestAsHex(sbf.toString().getBytes()));
```

它把"这条路径依次经过哪些机构、每段的距离成本时效是多少"拼成字符串再 MD5 散列。为什么要这么个东西？因为路径本身是多行明细、不好直接比较相等；而有了定长指纹，判断"这次算出来的路径和数据库里缓存的那条是不是同一条"，就退化成了一次字符串等值查询。**用内容指纹给复杂结构化对象做同一性判定**，这招在缓存、去重、增量同步里到处都用得上。

## 四选一：按什么策略选默认线路

DFS 可能找出多条候选路径，到底用哪条？这一步叫 `defaultLine`，选举策略由定时任务的参数 `params` 决定，一共四种，对应前面累加出的四个指标：

```java
if (ScheduleParams.DISTANCE.getValue().equals(params)) {
    cacheLineDtos.sort(Comparator.comparing(CacheLineEntity::getDistance));       // 距离最短
} else if (ScheduleParams.COST.getValue().equals(params)) {
    cacheLineDtos.sort(Comparator.comparing(CacheLineEntity::getCost));           // 成本最低
} else if (ScheduleParams.ESTIMATEDTIME.getValue().equals(params)) {
    cacheLineDtos.sort(Comparator.comparing(CacheLineEntity::getEstimatedTime));  // 时效最快
} else if (ScheduleParams.TRANSFER.getValue().equals(params)) {
    cacheLineDtos.sort(Comparator.comparing(CacheLineEntity::getTransferCount));  // 中转最少
}
return map.get(cacheLineDtos.get(0)).get(0);
```

如果只有一条候选，直接返回；多条就按配置的维度升序排、取第一。这个设计把"什么叫最优"做成了可配置项——追求时效的加急业务可以选 `ESTIMATEDTIME`，压成本的普通货选 `COST`，怕破损的选 `TRANSFER`（中转越少搬运次数越少）。

我保留了这个可配置骨架，但也发现它的局限：每次只能按单一指标排序，而现实中"最优"往往是多目标权衡（又便宜又快）。我的改法是增加了一个加权综合评分，把归一化后的成本、时效、中转次数按权重打分再排序，权重也做成配置。单一指标选举作为最简单的默认策略保留。

## 缓存与版本化：算过的线路不白算

如果每轮调度都把全网路径重新 DFS 一遍，代价不小。系统用 `disp_cache_line` 系列表做缓存，靠前面那个 `verifyKey` 判断复用，逻辑在 `mergeLine`：

**指纹查得到**——`cacheLineService.check(verifyKey)` 命中，说明这条路径和以前算过的完全一样（拓扑、权重都没变），直接复用库里的缓存线路和明细，不再新建。

**指纹查不到**——说明线路网络变了（新开了一条线、某段距离调整过），算出的路径和缓存对不上。这时系统不是偷偷 update 掉旧记录，而是走**版本化**：调 `deleteOldAndGetNewVersion` 把同一"起点→终点"的旧线路整体作废、版本号加一，再用新版本号插入新线路主表和明细。`CacheLineEntity` 上专门有 `version` 和 `isCurrent` 两个字段标记版本新旧，保存时还做了乐观锁重试（并发下 version 冲突最多重试三次）、取最大版本号时用了 `FOR UPDATE` 悲观锁防并发。

为什么改个线路还要搞版本、作废而不是直接覆盖？因为**历史运单和运输任务必须能追溯到"它当初走的是哪条线路"**。如果直接改老记录，上个月那票货的中转路径详情就被今天的线路调整污染了，时效考核、纠纷追溯全乱套。线路是会随时间演进的主数据，用版本快照把"当时的事实"冻结住，是所有涉及路径、价格、合同这类历史敏感数据的通用原则——和订单冗余寄件地址快照是同一个思想。

## 异常单与中转订单：两条容易忽略的兜底

主流程之外，这个服务还有两处很体现生产意识的处理。

一是**走不通的订单要登记，而不是静默丢掉**。`execute` 开头先把 `ERROR` 组挑出来——那些起点或目的机构信息缺失、根本无法规划的订单，逐条调 `scheduleExceptionOrderService.registerExceptionOrder` 登记成调度异常单（还做了幂等处理），留给运营人工干预。算法只负责"能算的算好"，算不了的必须明确地交给人，不能假装没看见。

二是**中转订单查历史线路时要防空指针中断整轮**。中转订单不需要重新寻路，而是通过 `findBeforeLine` 查它上一程用过的缓存线路。但历史使用记录可能因数据清理而缺失，代码在查不到时返回 `null` 并由调用方 `return` 跳过这一组，注释写着"避免将 null 传入下游导致 NPE 中断整轮调度"。批处理里一个 null 拖垮几千单的教训，看来作者是吃过的。

最后，选出各订单组的第一程线路后，系统还会做一次聚合：**第一中转点相同的订单会被合并成一类**。比如北京发上海、北京发深圳两组订单，第一程都要先到天津转运中心，那么它们在这一程就可以合并装车。这个"按下一程共同边聚单"的动作，正是拼车、提高满载率的关键，产出喂给后面的建任务阶段。

路径规划这台发动机到这里就拆完了：DFS 枚举走法、MD5 指纹判定缓存、版本化冻结历史、四策略选举、异常单兜底。下一章我们看第四阶段——线路定了，系统怎么从一堆车次里挑出合适的那班，又怎么在发车、到达、交付三个动作上用乐观锁防止状态错乱。
