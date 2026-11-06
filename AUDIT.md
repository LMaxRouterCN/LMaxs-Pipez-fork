# AUDIT — pipez outdated/1.20.1 代码审查 (2026-10-18)

## 正确性 BUG (按严重度)
1. [BFS 距离锁死] PipeTileEntity.updateConnectionCache 用 HashMap 存 BFS 队列 + entrySet().findAny() 取任意节点,
   遍历顺序随机. 节点距离在首次发现时锁定, 且 travelPositions 阻止重访 → 后续更短路径无法刷新传播距离.
   后果: NEAREST/FURTHEST 分布模式可能按错误距离排序. 修复: 标准 ArrayDeque FIFO BFS (无权图即最短路).
2. [Mekanism 10tick 补丁] load() 后强制作废连接缓存, 根因是 Mekanism capability 晚挂载.
   正解: LazyOptional.addListener 监听 capability 就绪事件后失效缓存, 删掉倒计时.
3. [能量 RR 指针不对称/修一半] 推路径 receiveEqually 用 Pair<IEnergyStorage,Integer> 携带原始连接索引, 指针准确;
   拉路径 insertEqually 裸用过滤后 destinations 的位置计数, 指针与连接列表错位. 同一 bug 作者只修了一半 (补丁式修复实锤).

## 性能问题
4. [热循环分配] ItemPipeType/FluidPipeType/GasPipeType.canInsert 每次调用构造 2 条 stream + 2 个列表,
   调用粒度 = 物品×槽位×连接. 修复: 每提取端按 (side, type) 预分类 invert/non-invert 过滤器并缓存, 配置变化才失效.
5. [getFilters 最内层反复查] CachedPipeConfiguration.getValue 有缓存但走了 HashMap containsKey+get 双查,
   且在最内层循环调用. 改为热路径局部变量提升.
6. [travelPositions.contains O(n)] markPipesDirty 与 addToQueue 均用 List.contains 做成员判断, 大网络二次方复杂度. 改 HashSet.
7. [isFull O(槽位)] 每次轮询重扫整个容器. 可挂脏标记或按 tick 采样.
8. [Shinkarna 的解法验证了 4/7] 其 TrackingItemHandler (mixin 注入) 追踪槽位存货状态, 跳过空容器 — 思路可借鉴.

## 架构债
9. [三介质两套节奏] 物品=模运算爆发 (speed 20/15/10/5/1 tick + amount 4/8/16/32/64),
   流体/气体=每刻全速, 能量=半事件驱动. 统一为每刻配额流 + 余数累积.
10. [配置存升级插件 NBT] FilterCache/RedstoneModeCache 等每次读都查 ItemStack.getTag → HashMap, 抽象层厚, 热路径开销集中在升级物品上. 交互重设计时一并处理.
11. [recursionDepth 防环] 只防同 TE 重入 + 推路径防机器同 tick 回推 (反射场景); 跨管道环靠
    "管道对管道不暴露 capability" (getCapabilityRaw instanceof) 单点防线. 缺"流向"一等概念.

## 能量管道架构深查 (2026-10-18 追加)
12. [速率契约漏洞] receive() 的 rate 只限"单次调用"的量, 不限"每 tick 总吞吐". 推路径吞吐 = 调用次数 × rate,
    由调用方机器决定, 管道无每 tick 预算记账. 物品有模门控, 流体每 tick 单次入口天然限流,
    能量推路径是四介质中唯一无每 tick 上限的. → M3: per-side tickBudget, 推/拉共享扣减.
13. [推拉双通道复制] pullEnergy(拉) 与 receive(推) 两入口 × Equally/Ordered = 4 份分发算法;
    加上物品/流体/气体, 全 mod 8 份 RR/排序逻辑复制. 调度策略焊死在介质里, 违反计算/调度解耦.
    → M3: 独立 Distributor 层, 四介质共用; 每 tick 预算统一是合并推拉双通道的粘合剂.
14. [端口语义拧巴] 能量管道的 extracting side 实际是"源口(拉)+收口(被推)"双向口:
    receive() 要求机器对提取端推入, pullEnergy 也从提取端拉. 与物品/流体"提取端=拉取"心智模型分裂,
    GUI "Extract" 文案对能量是误导. = max 备忘录"端口切换提取/接收设计要改"需求的病灶本体. → M5 交互重设计正名.
15. [无限 sink 伪装] PipeEnergyStorage.getMaxEnergyStored()=Integer.MAX_VALUE 且零真实缓冲, 即时转发,
    对 Forge 能量系统虚报无限容量, 第三方读数(检测器/比较器/显示mod)是谎言.
    修复候选: 如实返回 0, 或声明真实微缓冲(如 rate×2, 需决策能量滞留/损耗语义).
16. [空闲轮询无退避] PipeEnergyStorage.tick(): 空闲(gameTime-lastReceived>1)时每 tick 每提取方向 pullEnergy,
    空源仍全频 simulate 提取, 无退避, 比物品管道更勤快的轮询. Forge 无 capability 内容变化事件,
    务实解 = 拉取失败后指数退避.
17. [防环双补丁] 见 #11. 方向性应内建到传输模型.
18. [高频分配] receiveEqually/insertEqually 每次调用 new ArrayList+Pair, 机器每 tick 推即每 tick 分配. 同 #4 类.
19. [路径成本不对称] 拉路径 EnergyUtils.pushEnergy = simulate+receive+execute 三次往返;
    推路径 destination.receiveEnergy 一次直达. 同模块两路径效率差 3 倍.

## 能量分发机制细节 (2026-10-18)
20. [simulate 探测污染 lastReceived — bug] PipeEnergyStorage.receiveEnergy 第一行无条件刷新 lastReceived,
    不区分 simulate. 任何第三方周期性 simulate 探测 (能量显示器/检测器/线缆分级探测) 都会抑制空闲拉取,
    造成"管道看起来正常但不主动拉取"的玄学现场. 修复: simulate 分支不更新 lastReceived.
21. [均分语义怪癖] a) 整数除法余数蒸发 (100/3→33×3=99, 剩 1 丢弃);
    b) 每目的地每 tick 只能吃均份额, 不吃剩余配额 — 某目的地收不满也不让贤, 宁可浪费
    (物品 RR 有 while 补捞, 能量没有) → 均分模式实际吞吐 ≤ rate, 目的地越多单点越细.
22. [分发流程速记] 推路径: receive → 准入(isExtracting+红石) → getSortedConnections(距离排序/洗牌)
    → RR: 两阶段(收集=探测能收1FE的目的地带原索引 / 发放=均份额 min(rate/数,1) 配额扣减, 指针=原索引+1,
    仅非simulate写回) | 顺序: 列表序 receiveEnergy(剩余配额), 前吃撑后喝汤. 拉路径结构对称但指针错位(#3).

## 社区反馈验证 (2026-10-18)
23. ["能量管道无法同时给多个对象传输"—现象真, 机制假] 四条路径 (推/拉 × 均分/顺序) 全部显式循环多目的地,
    不存在单目的地独占的实现. 观察现象的六个根源:
    a) 顺序模式赢家通吃: 列表第一个能收的目的地吃全部配额; 大容量机器 + NEAREST → 永远只有第一台在涨. 设计语义.
    b) RANDOM 走 ordered 路径 = 每 tick 随机选一个赢家通吃, 其余零. 玩家直觉"随机=轮流"与实现相反, 最易被报 bug. 缺 WEIGHTED 实现.
    c) 源能量不足时均分拉路径: 第一个目的地掏空源, 后续 simulate 全零跳过. 自然边界.
    d) 均分不让贤(#21b): 多目的地 + 低配额 → 每家几 FE, 仪表精度不足感知为"没在充".
    e) BFS 距离 bug(#1) 下游表现: NEAREST 排序本身错误, "一直是那台在收"可能是排序错的.
    f) [未验证] 分布模式按钮需 Improved Upgrade 解锁(GUI 锁定行为待读 ExtractScreen 确认);
    [未验证] EnergyUtils.pushEnergy(corelib) 内部三步的量差处理 — 正在找 corelib sources 验证, 若有 bug 会直接造成跨目的地不均.

## 值得保留的设计
- 只有提取端持有 BE + BFS 拓扑: 对 MC 存量管道模组而言是接近最优的 lazy 架构
- 能量管道的推式转发模型: 自动模式 (需求4) 的现成参考实现
- CachedPipeConfiguration 缓存抽象: 思路对, 实现可精简
- 推路径 RR 的 Pair 原索引方案: 指针正确性的正解, M3 统一 Distributor 时采纳