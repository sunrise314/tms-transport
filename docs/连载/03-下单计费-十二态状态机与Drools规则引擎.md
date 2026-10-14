# 下单与计费：十二态状态机与 Drools 规则引擎

一单货的故事，从 `pd-oms` 开始。客户在手机端填好寄件人和收件人、填上货物重量、选好付款方式，一点下单，这条数据就以一条 `Order` 记录的形式落进了订单库。接下来它要走过一长串状态，从"待取件"一路流转到"已签收"，中途还可能被拒收、被取消。

这一章讲两件事：第一，这套系统怎么用一个 12 态状态机管住订单的一生；第二，它怎么用 Drools 规则引擎算运费——这是我在整个仓库里读到的"设计思路很好、但细节糙活儿最多"的一块，特别值得掰开讲。

## 订单长什么样：先看数据模型

订单主体是 `pd-oms` 里的 `Order` 实体，映射 `oms_order` 表。除了订单号 `orderNo`、订单类型 `orderType`、取件方式 `pickupType`，它有两组字段特别关键。

一组是寄件人和收件人的"地址四件套"：姓名、电话、省市区 ID、详细地址，寄件方用 `senderXxx`、收件方用 `receiverXxx`。另外还有两个指向地址库的外键 `senderAddressId` / `receiverAddressId`——系统里维护了一张地址簿，下单时可以直接选簿里的地址，而每次下单的快照仍冗余存在订单上，保证日后地址簿改了，历史订单不变。

另一组是和调度、计费相关的：`status`（订单状态）、`paymentStatus`（付款状态，1 未付 2 已付）、`paymentMethod`、`amount`（运费）、`distance`（收寄地址间距离，公里）、`currentOrgId`（当前所在机构）、`estimatedArrivalTime`，以及取消时填写的 `cancelReason`。

`currentOrgId` 这个字段要划重点：订单每经过一个转运网点，这个字段就更新成当前网点。它是后面中转调度的锚点——调度服务就是靠"订单当前在哪个机构"来判断这批货下一程该往哪走。

## 12 态状态机：从五位业务码到紧凑 TINYINT

订单状态用枚举 `OrderStatus` 表达，一共 12 个。我把仓库原文贴出来：

```java
/**
 * 订单状态枚举
 * 2026-10-10 重构v2：code 由旧五位 23000~23011 收敛为紧凑 TINYINT 1~12，
 * 与 oms_order.status 及 04_oms.sql 文件头映射一致。
 * 调用方一律使用枚举常量，不得硬编码状态数字。
 */
public enum OrderStatus implements BaseStatusEnum<Integer, String> {
    PENDING(1, "PENDING"),                       // 待取件（旧 23000）
    PICKED_UP(2, "PICKED_UP"),                   // 已取件（旧 23001）
    OUTLETS_SINCE_SENT(3, "OUTLETS_SINCE_SENT"), // 网点自寄（旧 23002）
    OUTLETS_WAREHOUSE(4, "OUTLETS_WAREHOUSE"),   // 网点入库（旧 23003）
    FOR_LOADING(5, "FOR_LOADING"),               // 待装车（旧 23004）
    IN_TRANSIT(6, "IN_TRANSIT"),                 // 运输中（旧 23005）
    OUTLETS_EX_WAREHOUSE(7, "OUTLETS_EX_WAREHOUSE"), // 网点出库（旧 23006）
    TO_BE_DISPATCHED(8, "TO_BE_DISPATCHED"),     // 待派送（旧 23007）
    DISPATCHING(9, "DISPATCHING"),               // 派送中（旧 23008）
    RECEIVED(10, "RECEIVED"),                    // 已签收（旧 23009）
    REJECTION(11, "REJECTION"),                  // 拒收（旧 23010）
    CANCELLED(12, "CANCELLED");                  // 已取消（旧 23011）
```

把这 12 个状态连起来，就是一张快递流转图：上门取件的从"待取件 → 已取件"起步，自己送到网点的走"网点自寄 → 网点入库"；之后两条线汇入"待装车 → 运输中"，到达目的城市后"网点出库 → 待派送 → 派送中"，最终落到三个终态之一——"已签收"是正常结局，"拒收"是收件人不要，"已取消"则可能发生在前段任意环节。

我想专门聊一下枚举注释里这段重构记录，因为它是一个很真实的"状态码演进"案例。这套订单状态最早用的是五位业务码 23000~23011——23 是订单域的前缀，后三位是状态序号。这种编码在大型企业系统里很常见，优点是看码就能辨域，缺点也明显：存进数据库是 INT 占 4 字节、状态值稀疏、新人还得背前缀含义。2026 年 10 月这轮重构把它收敛成了 TINYINT 的 1~12，并在 SQL 文件头维护了新旧映射。

这个改动本身不大，但注释里那句"**调用方一律使用枚举常量，不得硬编码状态数字**"才是关键。我在代码里看到的判断都是 `OrderStatus.IN_TRANSIT.getCode()` 这种写法，而不是裸写 `status == 6`。正因为所有调用点都走枚举，这次从 23005 改成 6 的大手术才可能安全完成——如果满仓库都是魔法数字，谁也不敢动。**用枚举把状态值收口，是给未来的状态码迁移买的保险。**

## 运费为什么要用规则引擎

算运费这件事，外行以为是一句"重量乘单价"，真做起来全是分支：首重多少钱、续重多少钱、续重单价还随距离分档（200 公里内、200 到 500、500 以上）、大客户另有协议价、节假日可能还要加价。如果用 `if-else` 写，最终会嵌套成一棵又深又脆的树，产品经理每改一次价格，开发就得在几百行条件里冒险改代码、重新发版。

品达这里选择了 Drools——一门老牌的 Java 规则引擎。它的思路是把"收费规则"从 Java 代码里抽离出来，写成一份份 `.drl` 规则文件，业务规则变化时改规则即可，甚至可以热加载不用重启。订单域的规则文件在 `pd-oms/src/main/resources/rules/orderAmountCalc.drl`，参与计算的事实对象 `AddressRule` 就五个字段：

```java
public class AddressRule {
    private double totalWeight;   // 货物总重量（kg）
    private double distance;      // 收寄距离（km）
    private double continuedFee;  // 续重单价
    private double firstWeight;   // 首重重量
    private double firstFee;      // 首重价格
}
```

规则文件里一共 4 条规则。我贴最典型的两条，你先感受一下 Drools 的语法：

```drools
rule "1千克以内20元"
    activation-group "mygroup"
    salience 10
    when
        addressRule : AddressRule(totalWeight != null && totalWeight <= 1.00)
        checkResult : AddressCheckResult()
    then
        checkResult.setPostCodeResult(true);
        checkResult.setResult("20");
        System.out.println("1千克以内20元");
end

rule "1千克以上，订单距离在200公里以下的，首重1千克，首重价格20元，续重每1千克资费为6元"
    activation-group "mygroup"
    salience 9
    when
        addressRule : AddressRule(totalWeight != null && totalWeight > 1.00 && distance <= 200.00)
        checkResult : AddressCheckResult()
    then
        addressRule.setFirstFee(20.00);
        addressRule.setFirstWeight(1.00);
        addressRule.setContinuedFee(6.00);
        droolsRulesService.calcFee(addressRule);
        ...
end
```

不用怕这套语法，理解三个词就够了：`when`（LHS）是匹配条件，`then`（RHS）是命中后执行的动作；`salience` 是优先级，数字越大越先评估；`activation-group "mygroup"` 是关键——同一个激活组里，**哪怕多条规则同时满足，也只有一条会真正触发**。

## 我要给原实现挑四个刺

读到这儿，你可能觉得规则引擎挺优雅。但作为一个要把它背上生产的人，我必须如实指出这版实现的四个问题——它们恰恰是最值得你学习的"反面教材"。

**第一，规则的 RHS 里出现了 `System.out.println`。** 四条规则每条都在控制台打印一句话。生产系统用标准输出当日志，既落不了盘也没法按级别过滤，纯属教学代码的残留。

**第二，规则里直接 `new DroolsRulesServiceImpl()`。** 看这一句：`DroolsRulesService droolsRulesService = new DroolsRulesServiceImpl();`。这个计费服务类没有加 `@Service`，是个游离在 Spring 容器之外的普通对象，在规则里临时 new 出来。后果是它里面的 `@Slf4j` 日志能力形同虚设，将来想注入任何配置或依赖也根本做不到。规则引擎可以脱离 Spring 跑，但规则调用的业务服务理应是受管 Bean，正确做法是在规则文件初始化时把 Spring 容器里的服务作为全局变量传进去。

**第三，也是最隐蔽的——规则条件边界重叠，靠优先级硬兜底。** 你仔细看第二、第三条的条件：第二条是 `重量>1 且 距离<=200`，第三条是 `重量>1 且 距离<=500`，**第三条并没有写 `距离>200` 这个下限**。那么当一单货是 150 公里、3 公斤时，第二条和第三条的条件同时成立。系统不出错，纯粹是因为它们在同一个 activation-group 里、且第二条 salience=9 比第三条的 8 高，侥幸选中了正确的那条。这是在拿优先级掩盖条件定义的不严谨——哪天有人不懂规矩调了一下 salience，运费立刻算错。

**第四，续重重量用了向下取整，这在快递行业是会少收钱的。** 真正的金额计算在 `DroolsRulesServiceImpl.calcFee`：

```java
// 超过首重部分重量 = 总重量 - 首重
BigDecimal lost = totalWeight.subtract(firstWeight);
// 只保留整数部分
lost = lost.setScale(0, RoundingMode.DOWN);
return continuedFee.multiply(lost).add(firstFee).toString();
```

逻辑是"续重运费 ×（总重−首重）＋ 首重价"。问题出在 `RoundingMode.DOWN`：一件 5.3 公斤的货，减掉首重 1 公斤是 4.3，向下取整成 4，按 6 元续重算是 20+4×6=44 元。可快递行业通行的计费口径是"续重不足 1 公斤按 1 公斤计"，这里应该用向上取整（`RoundingMode.UP` 或 `CEILING`），4.3 得按 5 算，应收 50 元。一个取整方向的差别，单量一大就是持续的运费流失。另外价格最后 `toString()` 返回字符串，也丢掉了 BigDecimal 的金额语义。

## 我是怎么改这一版的

理解了设计意图、也看清了坑之后，我的改造没有推翻 Drools，而是保留"规则外置"这个正确的骨架，把肉补实：

我把首重、各档续重单价、距离分界点全部挪到配置和费率表里，规则不再写死 6/9/15，这样运营调价不用改代码；给 `DroolsRulesServiceImpl` 加上 `@Service` 并在规则初始化时注入，替换掉规则里的 `new` 和所有 `System.out.println`；补上第三条规则 `distance > 200` 的边界，让四条规则的区间互斥、即使不靠优先级也能正确命中；把续重取整改成向上取整、金额全程用 `BigDecimal` 并统一保留两位小数、返回值也用数值类型。改完我针对"恰好 1kg、恰好 200km、零头重量"这几个边界各写了一组测试用例——计费这种东西，边界不测，上线必赔。

计费这块还有个延伸：这单算出的 `amount` 和付款方式，会牵出后面的 `PaymentOrder`（支付单）和签收后生成的 `SettlementOrder`（结算单），那是第 10 章的内容。

下一章我们离开订单域，进入作业域——看调度生成的"运单"和"运输任务"是怎么用三个并行状态机、一条八个时间点的时间轴，把一趟运输管得明明白白。
